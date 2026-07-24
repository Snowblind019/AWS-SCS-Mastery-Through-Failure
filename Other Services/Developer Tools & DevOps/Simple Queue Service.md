# Amazon SQS

Amazon SQS is a managed message queuing service that decouples producers from consumers: a producer sends a message, SQS stores it redundantly across Availability Zones, and a consumer polls for it, processes it, and deletes it. Its security value is durability and containment. A queue absorbs a traffic spike or a consumer outage without losing events, which matters when the events are security findings or remediation jobs, and it isolates failure so a broken consumer backs up its own queue rather than cascading. It is also a genuine trust boundary in a way that is easy to overlook: a consumer that acts on queue contents will act on whatever anyone with `sqs:SendMessage` puts there, so message injection into a queue feeding an automated remediation worker is a direct path to executing attacker-chosen actions with the consumer's permissions. The other overlooked property is audit coverage. SQS message operations are not recorded in CloudTrail at all, so who sent and who consumed a message is not directly knowable. The thing to hold onto is that SQS gives you durability and blast-radius containment, and takes away message-level attribution, so the consumer must validate what it receives rather than trusting that the queue is authentic.

## How it works

- **Queue types.** Standard queues provide nearly unlimited throughput, best-effort ordering, and at-least-once delivery, meaning duplicates are expected. FIFO queues provide strict ordering within a message group and exactly-once processing through a five-minute deduplication window, at substantially lower throughput.

- **The receive lifecycle.** `SendMessage` stores the message. `ReceiveMessage` returns it and starts the visibility timeout, during which no other consumer can see it. `DeleteMessage` removes it permanently. If the consumer does not delete before the timeout expires, the message becomes visible again and is redelivered, which is the mechanism behind at-least-once semantics and the reason consumers must be idempotent.

- **Visibility timeout.** Default 30 seconds, configurable up to 12 hours per queue or extended per message with `ChangeMessageVisibility`. Set shorter than the consumer's processing time it causes duplicate work; set very long it delays recovery after a consumer crash.

- **Retention and delay.** Messages are retained 60 seconds to 14 days, defaulting to 4 days. Delivery delay of up to 15 minutes can be set per queue or per message, which is a throttling and backoff mechanism rather than a scheduling one.

- **Dead-letter queues.** A redrive policy names a DLQ and a `maxReceiveCount`. A message received that many times without deletion moves to the DLQ, isolating poison messages so they stop consuming capacity. A DLQ must match the source queue's type, standard with standard and FIFO with FIFO. The redrive capability moves messages back once the consumer is fixed.

- **Encryption.** Server-side encryption with an AWS managed key or a customer managed KMS key, applied to the message body. Message attributes and metadata are not encrypted by SSE. A data key reuse period between 1 and 24 hours controls how often SQS calls KMS, trading cost against key usage granularity. SQS-managed keys are an alternative requiring no KMS configuration but offering no key policy control.

- **Access control.** IAM identity policies plus a resource-based queue policy. `sqs:SendMessage` controls injection, `sqs:ReceiveMessage` and `sqs:DeleteMessage` control consumption, and `sqs:PurgeQueue` deletes every message in the queue in one call, which is a destructive action worth restricting and alerting on separately.

- **TLS enforcement.** A queue policy statement denying all actions when `aws:SecureTransport` is false is the standard control. Additionally, a condition on `kms:EncryptionAlgorithm` or a deny on `sqs:SendMessage` without encryption is how you keep unencrypted messages out of an encrypted queue's ecosystem.

- **Network.** Interface VPC endpoints let consumers and producers reach SQS without internet egress, and an `aws:SourceVpce` condition on the queue policy restricts access to those endpoints. This is the strongest practical containment for a queue carrying sensitive payloads.

- **Cross-account and service integration.** Queue policies permit cross-account send or receive, and permit AWS service principals such as SNS, EventBridge, and S3 event notifications to publish, with `aws:SourceArn` and `aws:SourceAccount` conditions guarding against confused deputy.

- **Lambda event source mapping.** Lambda polls the queue on your behalf using the function's execution role, with batch size, batching window, and maximum concurrency controls. Function-level errors return the batch to the queue, so the DLQ and `maxReceiveCount` are what stop a failing function from reprocessing the same batch indefinitely.

- **Logging.** CloudTrail records control plane operations: `CreateQueue`, `SetQueueAttributes`, `AddPermission`, `PurgeQueue`, `DeleteQueue`. `SendMessage`, `ReceiveMessage`, and `DeleteMessage` are data plane operations and are not recorded. With a customer managed KMS key, the resulting `GenerateDataKey` and `Decrypt` calls do appear in CloudTrail, which is the only indirect signal of message activity and the identity behind it.

- **Monitoring.** `ApproximateNumberOfMessagesVisible` shows backlog. `ApproximateAgeOfOldestMessage` is the better health signal, since a rising age means the consumer has stopped working even if the queue is not growing. `ApproximateNumberOfMessagesNotVisible` shows in-flight work, and DLQ depth is the alarm that matters most for a security pipeline.

## SQS versus adjacent buffering and messaging options

| Option | Consumers per message | Retention | Ordering | Replay after processing | Message-level audit | Typical use |
|---|---|---|---|---|---|---|
| Amazon SQS Standard | One | Up to 14 days | Best effort | No, deleted messages are gone | None in CloudTrail | Durable work buffer, decoupling |
| Amazon SQS FIFO | One | Up to 14 days | Strict per message group | No | None in CloudTrail | Ordered, non-duplicated processing |
| Amazon SNS | All subscriptions | None | FIFO topics only | No | Data events available | Broadcast to many destinations |
| Amazon MQ | Depends on protocol | Broker configured | Protocol dependent | Depends | Broker logs | Lift and shift of existing JMS or AMQP |
| Kinesis Data Streams | Many, independently | Up to 365 days | Per shard | Yes, by iterator position | Data events available | High-volume telemetry with replay |
| EventBridge | Per rule target | None, archive optional | None | Yes, with archive and replay | CloudTrail on management | Content-based routing of events |

## What gets tested

- **Message injection is the primary threat to a queue-driven automation.** If a consumer performs privileged actions based on queue contents, the answers are a tightly scoped queue policy on `sqs:SendMessage`, source conditions on service principals, and validation of message structure and origin inside the consumer.

- **`aws:SecureTransport` deny statement** is the standard TLS enforcement answer, expressed in the queue policy rather than anywhere else.

- **KMS key policy grants to the publishing service principal.** SNS, EventBridge, or S3 publishing to an encrypted queue needs `kms:GenerateDataKey` and `kms:Decrypt` on the key. Messages silently not arriving after encryption was enabled is a recurring scenario.

- **Visibility timeout must exceed the consumer's processing time**, including Lambda timeout. A mismatch produces duplicate processing, which for a remediation action means the action executes more than once.

- **Dead-letter queue plus an alarm on its depth** is the answer for detecting poison messages and silently failing consumers. A queue with no DLQ discards nothing but also surfaces nothing.

- **`ApproximateAgeOfOldestMessage`** is the answer when a question asks how to detect that a consumer has stopped processing, since queue depth alone can look normal on a low-volume queue.

- **`sqs:PurgeQueue` is a destructive action.** Restricting it and alerting on it in CloudTrail is the answer for protecting a queue holding security events, since a single call empties it with no recovery.

- **SQS message operations are not in CloudTrail.** If a question asks how to determine which principal consumed a specific message, there is no direct answer, and the closest available evidence is KMS usage under a customer managed key plus consumer-side application logging.

- **FIFO for exactly-once processing** when duplicate execution is unacceptable, accepting the throughput reduction and the per-message-group partitioning it forces.

- **Interface VPC endpoint plus `aws:SourceVpce`** is the answer for keeping queue traffic off the internet and restricting where producers and consumers may connect from.

## Limitations

- No message-level audit trail. `SendMessage` and `ReceiveMessage` do not appear in CloudTrail, so attribution for what entered or left a queue depends entirely on application logging and, indirectly, on KMS events.

- One consumer per message. Multiple independent consumers of the same event require multiple queues fed by SNS or EventBridge, not multiple readers on one queue.

- At-least-once delivery on standard queues means duplicates are normal operation, not an error condition, so every consumer needs idempotency regardless of how carefully timeouts are tuned.

- Retention is capped at 14 days, so SQS is a buffer and never an archive. Long-term retention of events belongs in S3 or a log store.

- Message size is capped at 256 KB. The extended client library stores the payload in S3 and enqueues a pointer, which moves the real data access control to bucket policy and introduces an S3 dependency into the message path.

- Server-side encryption covers the message body but not message attributes, so identifiers placed in attributes for filtering purposes are stored unencrypted.

- FIFO throughput is far lower than standard and is bounded per message group, so achieving both ordering and throughput requires a partitioning key design that frequently conflicts with the ordering requirement itself.

- Deleted messages are unrecoverable and there is no replay. A consumer that deletes a message before successfully processing it has destroyed the event, which is a different failure mode from Kinesis where the record persists regardless of consumer behavior.

- Queue policies grant standing access with no expiry, so cross-account access persists until someone edits the policy.