# SNS versus SQS

SNS and SQS are both managed messaging services with nearly identical AWS security primitives, resource policies, KMS encryption, VPC endpoints, and CloudTrail coverage, but they solve opposite problems and fail in opposite ways. SNS is push-based publish-subscribe: a publisher sends once to a topic and SNS immediately fans the message out to every subscription in parallel, with no storage. SQS is pull-based queuing: a producer sends a message to a queue where it persists until a consumer polls, processes, and deletes it. The security consequence of that difference is durability. An SNS message whose subscriber is offline or misconfigured is retried and then lost, so a security alert that nobody was listening for never existed. An SQS message stays in the queue for up to fourteen days, survives consumer outages, and lands in a dead-letter queue after repeated processing failures, which is what makes it suitable when losing an event is unacceptable. The thing to hold onto is that SNS distributes and SQS preserves, so any pipeline that must both notify broadly and never lose an event uses SNS fanning out to SQS rather than choosing between them.

## How it works

- **SNS delivery model.** Publish to a topic, SNS pushes to every subscription simultaneously. Subscribers can be SQS queues, Lambda functions, HTTPS endpoints, email, SMS, mobile push, and Firehose. There is no storage; delivery is attempted with per-protocol retry policies and then abandoned unless a subscription dead-letter queue exists.

- **SQS delivery model.** Send to a queue, messages persist with a configurable retention of 60 seconds to 14 days. Consumers call `ReceiveMessage`, which makes the message invisible to other consumers for the visibility timeout, process it, then call `DeleteMessage`. A message not deleted before the timeout expires becomes visible again, which is why at-least-once processing and idempotent consumers are the norm.

- **The fan-out pattern.** One SNS topic with multiple SQS queue subscriptions. Each queue serves a different consumer with its own retention, its own visibility timeout, its own dead-letter queue, and its own processing rate. This is the canonical architecture for security event pipelines: one queue feeds real-time remediation, another feeds a SIEM, another buffers for batch analytics, and a slow or failing consumer on one queue does not affect the others.

- **Raw message delivery.** By default SNS wraps the payload in an envelope with metadata. Enabling raw message delivery on an SQS or Firehose subscription passes the payload through unwrapped, which matters because filter policies and downstream parsers behave differently in each case.

- **Access control.** Both use a resource policy: a topic policy on SNS, a queue policy on SQS. For SNS the risky permission is `sns:Subscribe`, since a new subscription is a standing copy of everything published. For SQS the risky permissions are `sqs:ReceiveMessage`, which lets a principal consume and delete messages another consumer needed, and `sqs:SendMessage`, which lets a principal inject messages a trusted consumer will act on.

- **Encryption.** Both support server-side encryption with an AWS managed or customer managed KMS key, and both enforce TLS on the API. In an SNS to SQS fan-out with customer managed keys, the SNS service principal needs `kms:GenerateDataKey` and `kms:Decrypt` on the queue's key, which is the single most common cause of a silently broken pipeline.

- **Failure handling.** SNS has per-subscription dead-letter queues, which are SQS queues holding messages that exhausted delivery attempts. SQS has redrive policies with a `maxReceiveCount` sending messages to a dead-letter queue after repeated failed processing, plus a redrive capability to move them back once the consumer is fixed.

- **Ordering and deduplication.** SNS FIFO topics and SQS FIFO queues provide strict ordering within a message group and exactly-once publishing with a deduplication window. FIFO topics deliver only to FIFO queues, so a FIFO fan-out is closed to Lambda, HTTPS, and email subscribers.

- **Filtering.** SNS filter policies evaluate message attributes or JSON body fields before delivery, so a subscriber never receives non-matching messages. SQS has no server-side filtering; a consumer receives everything in its queue and discards what it does not want, which means the filtering decision must happen upstream at the topic.

- **Logging.** CloudTrail records configuration operations on both. `Publish`, `SendMessage`, `ReceiveMessage`, and `DeleteMessage` are data plane events, not logged unless data events are enabled on the trail. CloudWatch metrics cover delivery failures for SNS and, for SQS, `ApproximateAgeOfOldestMessage` and `ApproximateNumberOfMessagesVisible`, which are the practical signals that a consumer has stopped working.

## Side-by-side comparison

| Dimension | Amazon SNS | Amazon SQS |
|---|---|---|
| Pattern | Publish-subscribe, push | Queue, pull |
| Consumers per message | Many, all subscriptions in parallel | One, whichever consumer receives it |
| Persistence | None, delivery attempt only | 60 seconds to 14 days, until deleted |
| Replay after consumer outage | No, message is lost | Yes, message waits in the queue |
| Ordering | FIFO topics only, SQS FIFO targets only | FIFO queues available |
| Server-side filtering | Yes, subscription filter policies | No, consumer filters |
| Failure handling | Subscription dead-letter queue | Redrive policy with `maxReceiveCount` plus DLQ and redrive back |
| Risky policy permission | `sns:Subscribe`, a standing egress copy | `sqs:ReceiveMessage` for theft, `sqs:SendMessage` for injection |
| Encryption at rest | KMS, optional | KMS, optional |
| Max message size | 256 KB, extended client for larger | 256 KB, extended client for larger |
| Latency profile | Milliseconds, immediate push | Poll interval dependent, long polling up to 20 seconds |
| Typical security role | Broadcast findings to many destinations | Durable buffer that must not lose an event |

## What gets tested

- **The requirement word tells you the service.** Broadcast, notify, fan out, multiple destinations means SNS. Buffer, retry, durable, must not lose, decouple a slow consumer means SQS. Both means SNS to SQS fan-out.

- **SNS to SQS fan-out is the answer for a security pipeline with multiple consumers of different speeds**, because each queue absorbs its own backlog independently and a failing SIEM ingester does not delay automated containment.

- **KMS key policy grants to the SNS service principal** are required for encrypted queues subscribed to an encrypted topic. Alerts that stopped arriving after encryption was enabled is a scripted exam scenario.

- **`sns:Subscribe` is the exfiltration permission on the SNS side.** Restrict it with `sns:Protocol` and `sns:Endpoint` conditions and by account. On the SQS side the equivalent exposure is `sqs:ReceiveMessage` granted too broadly, which lets a principal drain a queue another consumer depends on.

- **Message injection into a queue is a real attack path.** A consumer that trusts queue contents and acts on them, such as an automated remediation worker, will act on whatever an unauthorized `sqs:SendMessage` grant puts there. The queue policy plus message validation in the consumer is the answer.

- **Visibility timeout must exceed processing time.** A timeout shorter than the consumer's work causes duplicate processing, which for a remediation action means the action runs twice.

- **Dead-letter queues are not optional in security pipelines.** A finding that failed processing three times and vanished is an undetected incident, and the DLQ plus a CloudWatch alarm on its depth is the answer.

- **FIFO constrains the fan-out.** If ordering is required and the scenario also needs Lambda or HTTPS subscribers, the requirements conflict, and the answer is a FIFO queue with a consumer that forwards rather than a FIFO topic fanning out directly.

- **Neither service filters on the consumer side for SQS.** If a scenario wants different consumers receiving different subsets, the filtering belongs in SNS subscription filter policies or in EventBridge rules, not in the queue.

- **Data events must be enabled** for either service before message-level publish or receive activity appears in CloudTrail.

## Limitations

- SNS loses messages when subscribers are unreachable and no dead-letter queue is configured, which makes it unsuitable on its own for anything with a delivery guarantee requirement.

- SQS delivers to one consumer per message. Multiple independent consumers of the same event require multiple queues fed by SNS or EventBridge, not multiple readers on one queue.

- Both are at-least-once by default, so duplicate delivery is expected and consumers must be idempotent. FIFO reduces but does not eliminate this at the processing layer.

- Both cap messages at 256 KB. The extended client library stores the payload in S3 and sends a pointer, which relocates the actual access control to bucket policy and adds an S3 dependency to the message path.

- SQS has no server-side filtering and no content-based routing, so a queue receives everything sent to it and the consumer pays to read and discard what it does not need.

- SNS filter policies have complexity limits and require the payload to be JSON for body-scope filtering. Sophisticated routing belongs in EventBridge.

- Neither provides message-level audit by default. Without data events enabled, there is no record of who published or who consumed, only who configured.

- SQS message retention is capped at 14 days, so it is a buffer rather than an archive. Long-term retention of security events belongs in S3 or a log store.

- FIFO throughput is substantially lower than standard, and the constraint applies per message group, which forces a partitioning design that often conflicts with the ordering requirement that motivated FIFO in the first place.