# Amazon SNS

Amazon SNS is a managed publish-subscribe messaging service. A publisher sends a message to a topic and SNS fans it out in parallel to every subscription on that topic: SQS queues, Lambda functions, HTTPS endpoints, email, SMS, mobile push, Kinesis Data Firehose, and event buses. It is the delivery backbone underneath most AWS alerting, so GuardDuty findings routed through EventBridge, CloudWatch alarms, Config compliance changes, and Budgets thresholds all typically land on an SNS topic before they reach a human or a remediation function. Two things make SNS a genuine security surface rather than a plumbing detail. First, the topic access policy is a resource policy, which means a misconfigured topic can be published to or subscribed to by anyone, and a subscription is a standing data egress path to whatever endpoint the subscriber named. Second, topics frequently carry the full text of security findings, including resource identifiers, IP addresses, and account numbers, so an unencrypted topic with a broad policy is a live feed of your security posture. The thing to hold onto is that the topic policy governs both who can publish and who can subscribe, and an over-broad subscribe permission is the more dangerous of the two.

## How it works

- **Topics.** Standard topics offer best-effort ordering, at-least-once delivery, and very high throughput. FIFO topics guarantee ordering and exactly-once publishing within a message group but deliver only to SQS FIFO queues, which constrains them to application integration rather than alerting.

- **Subscriptions.** Each subscription names a protocol and an endpoint. Email, SMS, and HTTPS subscriptions require confirmation by the endpoint owner before delivery begins, which is the mechanism preventing an attacker from silently adding your topic to their own inbox. Lambda and SQS subscriptions do not require confirmation; authorization is handled by the target's resource policy.

- **Access policy.** A resource-based policy on the topic with two distinct concerns. `sns:Publish` controls who can send. `sns:Subscribe`, `sns:Receive`, and `sns:SetTopicAttributes` control who can attach a new destination or change the topic's behavior. AWS service publishers such as CloudWatch, GuardDuty via EventBridge, and Config need explicit statements with `aws:SourceArn` and `aws:SourceAccount` conditions to prevent a confused deputy.

- **Encryption.** In transit over HTTPS for the API and for HTTPS subscriptions. At rest with server-side encryption using an AWS managed key or a customer managed KMS key. When SSE is enabled with a customer managed key, every publisher and every AWS service publishing to the topic needs `kms:GenerateDataKey` and `kms:Decrypt` in the key policy, which is the single most common cause of alerts silently disappearing after someone enables encryption.

- **Message filtering.** Subscription filter policies match on message attributes or, with the message body scope, on JSON fields in the payload. Filtering is evaluated by SNS before delivery, so a subscriber never receives messages that do not match. This is how one topic serves an on-call rotation that wants only high-severity findings and a ticketing system that wants everything.

- **Message data protection.** SNS supports data protection policies that inspect message payloads for sensitive data patterns such as credentials, government identifiers, or payment card numbers, and then audit, redact, or block the message. This is the control for preventing a workflow from broadcasting PII to every subscriber on a topic.

- **Delivery retry and dead-letter queues.** Retry policies differ by protocol, with HTTPS endpoints having configurable retry policies and AWS-managed targets having their own. A dead-letter queue is configured per subscription, not per topic, and captures messages that exhausted delivery attempts. Without one, an undeliverable security alert is simply lost.

- **Delivery status logging.** Optional per-protocol logging to CloudWatch Logs, recording delivery success and failure with the message identifier. This is off by default and is the only way to prove a message reached a subscriber.

- **VPC endpoints.** SNS supports interface endpoints, so publishing from a private subnet does not require internet egress, and an endpoint policy plus an `aws:SourceVpce` condition on the topic policy restricts where publishing may originate.

- **Cross-account and cross-Region.** A topic policy can permit publishing or subscribing from another account, which is the standard pattern for centralizing security alerts in an audit account. Cross-Region delivery to SQS and Lambda is supported.

- **Logging.** CloudTrail records `CreateTopic`, `Subscribe`, `Unsubscribe`, `SetTopicAttributes`, and `AddPermission`. `Publish` is a data plane event and is not logged by default, so message-level publishing history requires enabling SNS data events on the trail.

## SNS versus adjacent messaging and routing services

| Service | Model | Persistence and replay | Filtering | Ordering guarantee | Encryption at rest | Typical security role |
|---|---|---|---|---|---|---|
| Amazon SNS | Pub/sub fan-out | None, delivery attempt only | Attribute and body filter policies per subscription | FIFO topics only | KMS, optional | Broadcast alerts to many destinations |
| Amazon SQS | Queue, single consumer group | Up to 14 days, replayable until deleted | None, consumer filters | FIFO queues | KMS, optional | Durable buffer in front of a processor |
| Amazon EventBridge | Event bus with rule matching | Archive and replay available | Full event pattern matching on payload | None | KMS on custom buses | Routing findings to targets by content |
| Kinesis Data Streams | Sharded stream, multiple consumers | Up to 365 days, replayable | Consumer side | Per shard | KMS | High-volume telemetry with replay |
| AWS User Notifications | Human-directed delivery | None | EventBridge patterns | None | Managed | Alerting people rather than systems |
| AWS Chatbot | Chat delivery from SNS | None | Upstream | None | Not applicable | Slack and Teams output |

## What gets tested

- **KMS key policy is why alerts stop after enabling encryption.** An AWS service publisher such as CloudWatch or EventBridge needs `kms:GenerateDataKey` and `kms:Decrypt` granted to its service principal in the key policy. This is a top-tier recurring exam item and a top-tier real-world failure.

- **Confused deputy on the topic policy.** Grants to service principals need `aws:SourceArn` and `aws:SourceAccount` conditions so another account's alarm cannot publish to your topic.

- **`sns:Subscribe` is an exfiltration permission.** A principal who can subscribe an arbitrary HTTPS endpoint or a cross-account SQS queue gets a live copy of everything on the topic. Restricting subscribe by protocol and by account, using `sns:Protocol` and `sns:Endpoint` conditions, is the hardening answer.

- **SNS does not persist messages.** If a requirement mentions replay, durability across a consumer outage, or guaranteed processing, the answer is SNS fanning out to SQS queues, the standard fan-out pattern, or EventBridge with an archive.

- **Dead-letter queues are per subscription.** A question about lost alerts from a failing Lambda subscriber is answered by a subscription DLQ, not a topic-level setting.

- **Delivery status logging is off by default.** Proving that a notification was delivered requires enabling it per protocol, and `Publish` calls require SNS data events on CloudTrail to be logged at all.

- **Message data protection policies** are the answer for preventing sensitive data from being broadcast, with audit, de-identify, or deny actions. Do not confuse this with Macie, which scans stored objects.

- **Filter policies over multiple topics.** When a scenario wants different teams receiving different severities from one source, the answer is one topic with subscription filter policies rather than several topics and duplicated publishing logic.

- **Interface VPC endpoint plus `aws:SourceVpce`** is the answer for publishing from private subnets without internet access and for restricting where publishes may originate.

- **Email and SMS subscriptions require confirmation**, which is the reason an attacker cannot quietly add their own address, and also the reason a subscription can sit in pending state indefinitely while everyone assumes alerting works.

## Limitations

- No persistence and no replay. A message that fails delivery after retries is gone unless a dead-letter queue was configured. SNS alone is not a durable pipeline.

- No ordering on standard topics and at-least-once delivery, so subscribers must be idempotent. FIFO topics fix both but deliver only to SQS FIFO queues.

- Message size is capped at 256 KB, with the extended client library pushing larger payloads to S3 and sending a pointer, which moves the real access control to the bucket.

- Filter policy complexity is bounded, and body-scope filtering requires the payload to be JSON. Complex routing decisions belong in EventBridge, which has a far more expressive pattern language.

- `Publish` is not logged by default, so without SNS data events enabled there is no record of who sent what to a topic, only who configured it.

- SMS delivery is subject to carrier filtering, per-account spend limits, and origination number registration requirements, which makes it an unreliable primary path for critical paging.

- HTTPS subscribers must validate the SNS signature to confirm the message actually came from SNS. Skipping validation means the endpoint will accept a forged POST from anyone who knows the URL.

- Subscription confirmation tokens are sent to the endpoint, so a compromised or mistyped endpoint that confirms becomes a permanent subscriber until someone notices it in the subscription list.

- Cross-account topic policies grant standing access. Revocation requires editing the policy, and there is no expiry mechanism on a subscription.