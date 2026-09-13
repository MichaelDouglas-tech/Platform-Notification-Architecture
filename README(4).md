# Tech Challenge 4: AWS Notification Platform

## Overview

This repository documents the design and implementation plan for a centralized Notification Platform on AWS. The platform allows web applications, mobile applications, internal microservices, and business systems to submit notifications through one secure API and deliver them through Email, SMS, and Push Notifications.

The solution uses an asynchronous, event-driven architecture. Amazon API Gateway receives requests, AWS Lambda validates them, Amazon EventBridge routes notification events, and separate Amazon SQS queues isolate each delivery channel. Channel-specific Lambda workers send Email through Amazon SES and send SMS and mobile Push Notifications through Amazon SNS.

> **Project scope:** This guide explains how to build a functional proof of concept. Production use also requires organizational security review, verified domains, SMS registrations, mobile-provider credentials, load testing, service-quota review, cost controls, and operational runbooks.

## Architecture Diagram

Place the exported architecture image in `docs/aws-notification-platform-architecture.png` and use the following Markdown to display it:

```markdown
![AWS Notification Platform Architecture](docs/aws-notification-platform-architecture.png)
```

The primary request flow is:

```text
Applications
    -> Route 53
    -> AWS WAF
    -> API Gateway
    -> Notification Ingestion Lambda
    -> EventBridge
    -> Channel SQS Queue
    -> Channel Lambda Worker
    -> SES or SNS
    -> End User
```

## AWS Services Used

| Service | Purpose |
|---|---|
| Amazon Route 53 | Provides DNS routing for the notification API. |
| AWS WAF | Protects the API from common web attacks and abusive traffic. |
| Amazon API Gateway | Provides the HTTPS notification endpoint, validation, authorization, and throttling. |
| AWS Lambda | Validates requests and processes Email, SMS, and Push messages. |
| Amazon EventBridge | Routes normalized notification events by channel. |
| Amazon SQS | Buffers messages and isolates channel processing. |
| Amazon SES | Sends transactional Email notifications. |
| Amazon SNS | Sends SMS and mobile Push Notifications. |
| Amazon DynamoDB | Stores templates, preferences, endpoints, idempotency records, and delivery status. |
| AWS IAM | Provides authentication and least-privilege authorization. |
| AWS KMS | Encrypts supported data at rest. |
| AWS Secrets Manager | Protects provider credentials and sensitive configuration. |
| Amazon CloudWatch | Provides logs, metrics, dashboards, and alarms. |
| AWS X-Ray | Traces supported request-processing stages. |
| AWS CloudTrail | Records AWS API and administrative activity. |

---

# Step 1: Define the Objectives

## Goal

Create one platform that accepts notification requests from multiple services and sends them through Email, SMS, and Push channels.

## Functional Objectives

- Provide a versioned notification API.
- Support Email, SMS, and Push Notifications.
- Allow one request to select one or multiple channels.
- Apply user preferences before sending optional notifications.
- Use approved, versioned templates.
- Track the result of every channel attempt.
- Retry temporary failures.
- Store repeatedly failed messages in dead-letter queues.
- Prevent intentional duplicates with idempotency keys.
- Allow future notification channels to be added without redesigning existing channels.

## Nonfunctional Objectives

- Scale automatically during traffic increases.
- Remain available across infrastructure failures within an AWS Region.
- Protect the API with authentication, authorization, validation, and throttling.
- Encrypt sensitive data in transit and at rest.
- Provide centralized logs, metrics, tracing, dashboards, and alarms.
- Minimize personally identifiable information in logs and events.

## Initial Assumptions

- The first release handles transactional notifications rather than large marketing campaigns.
- The API returns `202 Accepted` after safely accepting a request; it does not promise immediate delivery.
- Delivery processing is at least once, so consumers must be duplicate-aware.
- The proof of concept is deployed in one AWS Region.
- Multi-Region disaster recovery is documented as a future production enhancement.

## Step 1 Verification

Confirm that the design has measurable answers for the following questions:

- Who can submit a notification?
- Which channels are supported?
- How is duplicate delivery reduced?
- What happens when a provider is unavailable?
- Where can an operator find the delivery status?
- How can a new channel be added?

---

# Step 2: Plan the System Architecture

## Request Contract

Applications send an HTTPS request to:

```http
POST /v1/notifications
Content-Type: application/json
Authorization: Bearer <token>
Idempotency-Key: <unique-producer-key>
```

Example request:

```json
{
  "recipientId": "user-4821",
  "templateId": "order-shipped-v2",
  "channels": ["EMAIL", "SMS", "PUSH"],
  "parameters": {
    "orderNumber": "A10452"
  },
  "priority": "NORMAL",
  "correlationId": "order-A10452"
}
```

Example accepted response:

```json
{
  "notificationId": "notification-generated-by-platform",
  "status": "ACCEPTED",
  "acceptedAt": "ISO-8601-timestamp"
}
```

## Processing Sequence

1. A producer sends a notification request to API Gateway.
2. API Gateway authenticates the producer, validates basic request properties, and applies throttling.
3. The Ingestion Lambda validates the complete schema and authorization scope.
4. The function checks the idempotency key and returns the existing notification ID when the same request was already accepted.
5. The function retrieves the template and recipient preferences from DynamoDB.
6. The function writes the initial notification record to DynamoDB.
7. The function publishes a normalized `NotificationRequested` event to EventBridge.
8. EventBridge routes the event to the selected channel queues.
9. Each channel Lambda consumes messages from its queue.
10. The worker sends through SES or SNS and records the provider response.
11. Delivery-feedback events update the final channel status.
12. CloudWatch captures operational metrics and sanitized logs.

## Recommended Repository Structure

```text
notification-platform/
├── README.md
├── docs/
│   └── aws-notification-platform-architecture.png
├── infrastructure/
│   ├── templates/
│   └── policies/
├── src/
│   ├── ingestion/
│   ├── email-worker/
│   ├── sms-worker/
│   ├── push-worker/
│   └── delivery-status/
└── tests/
    ├── unit/
    └── integration/
```

## Step 2 Verification

- Confirm that every component in the diagram has a defined responsibility.
- Confirm that the primary flow has directional arrows.
- Confirm that Email, SMS, and Push each have an independent queue and worker.
- Confirm that every primary queue has a dead-letter queue.
- Confirm that feedback returns to the delivery-status data store.

---

# Step 3: Set Up the Core Components

## Prerequisites

Before beginning, obtain:

- An AWS account with permission to create the documented services.
- AWS CLI v2.
- Git.
- Python 3.12 or another supported Lambda runtime.
- An AWS Region selected for the proof of concept.
- A verified Amazon SES Email address or domain.
- SMS registration or an approved origination identity where required.
- Firebase Cloud Messaging or Apple Push Notification credentials for a real Push test.

> **Command placeholders:** Values enclosed in angle brackets, such as `<token>`, `<api-id>`, and `<region>`, must be replaced with values from your own AWS environment before running a command.

Verify the AWS CLI identity:

```bash
aws sts get-caller-identity
```

Configure a default Region if one is not already configured:

```bash
aws configure set region us-east-2
```

> Replace `us-east-2` if your environment uses a different Region. Keep every resource in the same Region during the initial proof of concept.

## 3.1 Create DynamoDB Tables

For a small proof of concept, use the following logical tables:

- `NotificationRecords`
- `NotificationPreferences`
- `NotificationTemplates`
- `NotificationEndpoints`
- `NotificationIdempotency`

Example table creation:

```bash
aws dynamodb create-table \
  --table-name NotificationRecords \
  --attribute-definitions AttributeName=notificationId,AttributeType=S \
  --key-schema AttributeName=notificationId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

Repeat this pattern for the other tables using an appropriate string partition key:

| Table | Suggested partition key |
|---|---|
| NotificationPreferences | `recipientId` |
| NotificationTemplates | `templateId` |
| NotificationEndpoints | `recipientId` |
| NotificationIdempotency | `idempotencyKey` |

Verify the tables:

```bash
aws dynamodb list-tables
```

## 3.2 Create Dead-Letter Queues

Create the dead-letter queues before the primary queues:

```bash
aws sqs create-queue --queue-name notification-email-dlq
aws sqs create-queue --queue-name notification-sms-dlq
aws sqs create-queue --queue-name notification-push-dlq
```

## 3.3 Create Primary Queues

```bash
aws sqs create-queue --queue-name notification-email-queue
aws sqs create-queue --queue-name notification-sms-queue
aws sqs create-queue --queue-name notification-push-queue
```

Configure each primary queue with:

- Server-side encryption.
- A visibility timeout longer than the worker Lambda timeout.
- A redrive policy pointing to its corresponding DLQ.
- A maximum receive count appropriate for the proof of concept, such as five attempts.

The redrive policy can be configured from the AWS Console:

1. Open **Amazon SQS**.
2. Select the primary queue.
3. Choose **Edit**.
4. Enable the dead-letter queue setting.
5. Select the matching DLQ.
6. Set the maximum receives.
7. Save the configuration.

## 3.4 Create the EventBridge Bus

```bash
aws events create-event-bus --name notification-platform-bus
```

Create three EventBridge rules that match the normalized event and requested channel:

- `route-email-notifications`
- `route-sms-notifications`
- `route-push-notifications`

Representative Email event pattern:

```json
{
  "source": ["notification.platform"],
  "detail-type": ["NotificationRequested"],
  "detail": {
    "channels": ["EMAIL"]
  }
}
```

Create equivalent rules for `SMS` and `PUSH`. Add the corresponding SQS queue as each rule's target and grant EventBridge permission to send messages to that queue.

## 3.5 Create Lambda Functions

Create these functions:

- `notification-ingestion`
- `notification-email-worker`
- `notification-sms-worker`
- `notification-push-worker`
- `notification-delivery-status`

The Ingestion Lambda must:

- Validate required request fields.
- Authorize the template and channels.
- Check the idempotency table.
- Read the recipient preferences and template.
- Write an initial status record.
- Publish the normalized EventBridge event.
- Return `202 Accepted` with the notification ID.

Each channel worker must:

- Read the SQS event.
- Validate the normalized event version.
- Retrieve the correct template.
- Render approved parameters safely.
- Call the correct delivery provider.
- Store the provider request ID and result.
- Return failure for retryable errors.
- Store a terminal failure for nonretryable errors.

Connect each SQS queue to its worker through a Lambda event-source mapping.

## 3.6 Create API Gateway

1. Open **Amazon API Gateway**.
2. Create an HTTP API or REST API named `notification-platform-api`.
3. Create the route `POST /v1/notifications`.
4. Connect the route to `notification-ingestion`.
5. Configure authentication.
6. Configure request throttling and access logging.
7. Deploy a `dev` stage.
8. Record the invoke URL.

Test with a nonproduction recipient:

```bash
curl -X POST "https://<api-id>.execute-api.<region>.amazonaws.com/v1/notifications" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Idempotency-Key: order-A10452" \
  -d '{
    "recipientId":"test-user",
    "templateId":"order-shipped-v2",
    "channels":["EMAIL"],
    "parameters":{"orderNumber":"A10452"},
    "priority":"NORMAL",
    "correlationId":"order-A10452"
  }'
```

## Step 3 Verification

- API Gateway returns a notification ID and accepted status.
- DynamoDB contains the notification record.
- EventBridge reports a matched event.
- The selected SQS queue receives the message.
- The worker Lambda is invoked.
- Failed messages retry and eventually move to the correct DLQ.

---

# Step 4: Add Notification Channels

## 4.1 Configure Email with Amazon SES

1. Open **Amazon SES**.
2. Open **Verified identities**.
3. Verify a test Email address or domain.
4. Complete the verification message or DNS records.
5. Create an SES configuration set for delivery events.
6. Give the Email worker permission to call the required SES send operation.
7. Configure the Email worker with the verified sender identity.
8. Test delivery to an approved recipient.

> New SES accounts may begin in a sandbox. In sandbox mode, recipients may also need verification. Request production access only after the application, consent process, bounce handling, and complaint handling are ready.

## 4.2 Configure SMS with Amazon SNS

1. Open **Amazon SNS** and review the Text messaging settings.
2. Confirm the account spending limit and supported destination country.
3. Complete required sender registration or origination-identity approval.
4. Give the SMS worker only the permissions required to publish SMS messages.
5. Configure delivery-status logging where supported.
6. Test only with a phone number that you control and that has consented.

> SMS rules, registration, delivery feedback, and sender requirements vary by destination country. Review the current AWS requirements before production use.

## 4.3 Configure Mobile Push with Amazon SNS

1. Create a Firebase Cloud Messaging or Apple Push Notification application.
2. Obtain the provider credentials through the provider's secure workflow.
3. Store sensitive provider configuration in Secrets Manager when applicable.
4. Create an SNS platform application.
5. Register a test device token as a platform endpoint.
6. Store the platform endpoint reference in `NotificationEndpoints`.
7. Give the Push worker permission to publish only to approved platform endpoints.
8. Test with a controlled development device.

## Channel Verification

For each channel, verify:

- The correct queue receives the message.
- The correct Lambda worker processes it.
- The correct provider is called.
- DynamoDB records the provider request ID.
- The final status is updated.
- Retryable failures return to the queue.
- Permanent endpoint failures are recorded without endless retries.

---

# Step 5: Implement Data Storage

## Notification Status Model

Recommended channel states:

```text
ACCEPTED -> QUEUED -> PROCESSING -> PROVIDER_ACCEPTED -> DELIVERED
                                      |                  |
                                      v                  v
                                    FAILED             BOUNCED
```

Do not allow an older event to overwrite a final state. Use DynamoDB conditional updates when changing status.

## Example Notification Record

```json
{
  "notificationId": "notification-id",
  "producerId": "order-service",
  "recipientId": "user-4821",
  "templateId": "order-shipped-v2",
  "correlationId": "order-A10452",
  "requestedChannels": ["EMAIL", "SMS", "PUSH"],
  "channelStatus": {
    "EMAIL": "DELIVERED",
    "SMS": "PROVIDER_ACCEPTED",
    "PUSH": "FAILED"
  },
  "createdAt": "ISO-8601-timestamp",
  "expiresAt": 0
}
```

## Data-Protection Rules

- Store recipient references instead of copying personal information into every event.
- Do not store passwords, authentication tokens, or secret values.
- Do not include sensitive information in notification bodies unless explicitly approved.
- Apply retention periods and DynamoDB TTL where appropriate.
- Encrypt tables with KMS.
- Restrict read and write permissions by component.

## Step 5 Verification

- A new request creates one notification record.
- Reusing an idempotency key does not intentionally create a new request.
- Every channel updates its own status.
- Expired temporary records are eligible for TTL removal.
- Unauthorized roles cannot read or update the tables.

---

# Step 6: Ensure High Availability and Scalability

## Scaling Controls

- Use API Gateway throttling per stage or client.
- Use SQS to absorb bursts and apply back pressure.
- Use Lambda reserved concurrency to protect providers.
- Configure a batch size appropriate for each channel.
- Use partial batch failure handling so successful SQS messages are not retried unnecessarily.
- Use DynamoDB on-demand capacity for variable proof-of-concept traffic.
- Monitor service quotas before load testing.
- Use separate high-priority queues if urgent traffic must bypass standard traffic.

## High Availability

The primary services in this design are AWS managed services designed for regional availability across multiple Availability Zones. No application server or single EC2 instance sits in the critical notification path.

## Disaster Recovery Option

For production recovery:

- Define infrastructure with Terraform, AWS CDK, or CloudFormation.
- Maintain a secondary-region deployment plan.
- Replicate approved configuration and selected data.
- Store deployment artifacts in a recoverable location.
- Use Route 53 health checks and a controlled failover policy.
- Test recovery procedures regularly.

## Load-Test Scenarios

- Normal request volume.
- Sudden traffic spike.
- Email provider throttling.
- SMS provider delay.
- Push endpoint failure.
- Lambda throttling.
- DynamoDB throttling.
- Messages moving to a DLQ.
- Recovery and controlled replay.

Measure accepted requests per second, API p95 latency, oldest-message age, worker errors, provider throttles, delivery latency, and cost.

---

# Step 7: Incorporate Security Measures

## Authentication

- Use IAM and Signature Version 4 for trusted AWS workloads.
- Use a JWT authorizer, such as Amazon Cognito, for approved external clients.
- Avoid shared, long-lived credentials.
- Use short-lived credentials and role assumption where possible.

## Authorization

Restrict each producer by:

- Allowed API actions.
- Permitted templates.
- Permitted channels.
- Tenant or application scope.
- Rate limits.
- Status records it may retrieve.

## Least-Privilege IAM

Examples:

- Ingestion Lambda: read approved configuration, write request status, and publish to one event bus.
- Email worker: read the Email queue, read approved templates, update status, and send through SES.
- SMS worker: read the SMS queue, update status, and publish approved SMS messages.
- Push worker: read the Push queue, update status, and publish to approved Push endpoints.

## Encryption

- Require HTTPS and TLS for requests in transit.
- Enable KMS encryption for DynamoDB, SQS, DLQs, SNS topics, and CloudWatch log groups where required.
- Limit access to each KMS key through key policies and IAM.

## Secrets

- Store external credentials in Secrets Manager.
- Never commit credentials to GitHub.
- Never place credentials in README examples.
- Enable rotation where the provider supports it.

## Vulnerability Management

- Scan application dependencies and deployment packages.
- Patch supported Lambda runtimes and libraries.
- Track remediation owners and due dates.
- Block deployment when unresolved critical vulnerabilities violate release policy.
- Retest after remediation.

---

# Step 8: Set Up Monitoring and Logging

## Required Metrics

| Layer | Metrics |
|---|---|
| API Gateway | Request count, p95 latency, 4xx, 5xx, throttles |
| Ingestion Lambda | Errors, duration, throttles, concurrency |
| SQS | Visible messages, oldest-message age, DLQ count |
| Channel workers | Success rate, retry rate, duration, provider throttling |
| SES | Deliveries, bounces, complaints, rejections |
| SNS | Accepted requests, failures, endpoint failures, available delivery feedback |
| DynamoDB | Latency, throttles, system errors |
| Business | Requests and outcomes by producer and channel |

## Structured Log Example

```json
{
  "timestamp": "ISO-8601-timestamp",
  "level": "INFO",
  "notificationId": "notification-id",
  "correlationId": "order-A10452",
  "producerId": "order-service",
  "channel": "EMAIL",
  "stage": "PROVIDER_ACCEPTED",
  "attempt": 1,
  "durationMs": 142,
  "errorCode": null
}
```

Do not log:

- Message bodies.
- Access tokens.
- Passwords.
- Provider credentials.
- Complete Email addresses.
- Complete phone numbers.

## Recommended Alarms

- API 5xx errors above the approved threshold.
- API latency above the service objective.
- Lambda errors or throttling above the approved threshold.
- Oldest SQS message exceeds the channel delivery objective.
- Any message appears in a DLQ.
- Email bounce or complaint rate exceeds policy.
- SMS or Push failure rate increases unexpectedly.
- DynamoDB reports sustained throttling.
- Notification volume or cost increases unexpectedly.

Build a CloudWatch dashboard containing API health, queue backlog, worker health, channel outcomes, DLQ activity, and estimated cost.

---

# Step 9: Create an Extensibility Plan

To add a channel such as Webhooks, Slack, or Microsoft Teams:

1. Define the channel destination schema.
2. Define consent, security, size, and rate requirements.
3. Define the provider request and response contract.
4. Create a new EventBridge routing rule.
5. Create a dedicated SQS queue and DLQ.
6. Create a channel-specific Lambda worker.
7. Implement a provider adapter behind the common sending interface.
8. Create a least-privilege IAM role.
9. Enable KMS encryption.
10. Add approved templates and preferences.
11. Add structured logs, metrics, dashboards, and alarms.
12. Test success, retryable failure, permanent failure, throttling, duplicate processing, DLQ routing, and replay.
13. Release gradually with controlled traffic and concurrency.

Existing producers continue using the same `/v1/notifications` API. Existing channel workers do not need to be modified.

---

# Step 10: Complete and Verify the Deliverables

## Deliverable Checklist

- [x] Architecture diagram
- [x] Design overview
- [x] Producer interface and request flow
- [x] Email, SMS, and Push design
- [x] Data-storage design
- [x] Scalability and high-availability explanation
- [x] Authentication, authorization, and encryption controls
- [x] Monitoring, logging, and alarm strategy
- [x] Retry and dead-letter-queue design
- [x] Extensibility plan
- [x] Design trade-offs
- [x] Verification and troubleshooting guidance
- [ ] Optional working proof of concept

## Design Trade-Offs

| Decision | Benefit | Trade-off |
|---|---|---|
| Asynchronous API | Absorbs traffic spikes and improves resilience | Final delivery is not known when the API responds |
| Separate channel queues | Isolates failures and supports independent scaling | Creates more resources and alarms to operate |
| Managed serverless services | Reduces infrastructure maintenance | Requires service-quota and cost governance |
| At-least-once processing | Provides durable and practical message processing | Requires idempotency and duplicate-aware workers |
| DynamoDB | Provides scalable key-based storage | Complex analytics may require another analytics store |
| Multi-Region recovery | Improves regional resilience | Adds cost and operational complexity |

## Troubleshooting

### The API returns 401 or 403

- Confirm the token or IAM signature is valid.
- Confirm the authorizer is attached to the route.
- Confirm the producer has permission to use the template and channels.
- Review API Gateway access logs without exposing the token.

### EventBridge does not route the message

- Compare the event with the rule's event pattern.
- Confirm the event bus name.
- Check EventBridge matched-event metrics.
- Confirm the queue policy allows EventBridge to send messages.

### Lambda does not process SQS messages

- Confirm the event-source mapping is enabled.
- Confirm the Lambda execution role can receive and delete SQS messages.
- Confirm the queue and Lambda are in the same Region.
- Review Lambda errors and the SQS oldest-message age.

### Email is not delivered

- Confirm the SES sender identity is verified.
- Check whether the account is in the SES sandbox.
- Confirm the test recipient is permitted.
- Review SES bounce, complaint, and rejection events.

### SMS is not delivered

- Confirm the destination country is supported.
- Confirm required registration or origination identity.
- Review the account SMS spending limit.
- Confirm that the recipient gave consent.
- Review available SMS delivery-status logs.

### Push Notification fails

- Confirm the device token is current.
- Confirm the SNS platform application credentials.
- Confirm the platform endpoint is enabled.
- Record invalid endpoints as permanent failures instead of retrying indefinitely.

### Messages appear in a DLQ

- Inspect the sanitized error code and correlation ID.
- Correct the application, configuration, permission, or provider issue.
- Test the correction in nonproduction.
- Redrive only messages that remain valid and safe to resend.
- Monitor the queue and worker during replay.

## Cleanup and Cost Control

Delete proof-of-concept resources when they are no longer required. Remove resources in a controlled order:

1. Disable API traffic and scheduled tests.
2. Confirm no required messages remain in queues or DLQs.
3. Remove API Gateway stages and APIs.
4. Remove Lambda event-source mappings and functions.
5. Remove EventBridge rules, targets, and the custom event bus.
6. Remove SQS queues and DLQs.
7. Remove test SNS platform endpoints and applications.
8. Remove test DynamoDB tables after exporting any required evidence.
9. Remove unused secrets and KMS keys according to approved deletion procedures.
10. Remove CloudWatch dashboards, alarms, and unneeded log groups.
11. Remove proof-of-concept IAM roles and policies after confirming they are not shared.

Never delete a shared or production resource merely because its name resembles a proof-of-concept resource. Verify the exact resource ARN, account, Region, tags, and dependencies first.

## Conclusion

I designed the AWS Notification Platform to give multiple applications one secure and consistent method for sending Email, SMS, and Push Notifications. API Gateway and Lambda provide controlled request ingestion. EventBridge and SQS provide routing, buffering, retry processing, and channel isolation. Amazon SES and Amazon SNS provide managed notification delivery, while DynamoDB stores templates, preferences, idempotency records, endpoint references, and delivery status.

CloudWatch, X-Ray, and CloudTrail provide operational visibility. IAM, KMS, WAF, and Secrets Manager protect access, credentials, and data. The architecture can scale with changing demand, tolerate temporary provider failures, and support future notification channels without major changes to the existing system.

## Official AWS Documentation

- [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/)
- [AWS Lambda](https://docs.aws.amazon.com/lambda/)
- [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/)
- [Amazon SQS](https://docs.aws.amazon.com/sqs/)
- [Amazon SES](https://docs.aws.amazon.com/ses/)
- [Amazon SNS](https://docs.aws.amazon.com/sns/)
- [Amazon DynamoDB](https://docs.aws.amazon.com/dynamodb/)
- [AWS IAM](https://docs.aws.amazon.com/iam/)
- [AWS KMS](https://docs.aws.amazon.com/kms/)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)
- [Amazon CloudWatch](https://docs.aws.amazon.com/cloudwatch/)
- [AWS X-Ray](https://docs.aws.amazon.com/xray/)
- [AWS CloudTrail](https://docs.aws.amazon.com/cloudtrail/)

---

**Author:** Michael Theo Douglas  
**Focus:** AWS Cloud Engineering, DevOps, Serverless Architecture, Infrastructure Automation, and Vulnerability Management
