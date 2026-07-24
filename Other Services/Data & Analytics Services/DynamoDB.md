# Amazon DynamoDB

Amazon DynamoDB is a fully managed NoSQL key-value and document database with no instances, no endpoints inside your VPC by default, and no database engine to patch. Every interaction is a SigV4-signed HTTPS API call, which changes the entire security model: there are no database users, no passwords, no listening port, and no SQL parser to inject into. Authorization is IAM and only IAM, which means access control is expressed in policy documents alongside the rest of your account rather than in a separate credential store, and it can be narrowed with condition keys down to specific partition key values and specific attributes. Encryption at rest is always on and cannot be disabled, transport is always TLS, and every control plane and data plane call is a CloudTrail event. The thing to hold onto is that DynamoDB has no network perimeter to defend, so the entire security posture reduces to IAM policy scope, KMS key choice, and whether you have constrained the API path with VPC endpoint conditions.

## How it works

- **IAM is the only authorization mechanism.** Actions map one to one with API calls: `dynamodb:GetItem`, `PutItem`, `UpdateItem`, `DeleteItem`, `Query`, `Scan`, `BatchGetItem`, `TransactWriteItems`. Resources are table ARNs, index ARNs, and stream ARNs. There is no grant statement, no role inside the database, and no way to authenticate without an AWS identity.

- **Fine-grained access control through condition keys.** `dynamodb:LeadingKeys` restricts a principal to items whose partition key matches a value, most commonly the caller's identity, which is how per-tenant isolation is enforced in a shared table. `dynamodb:Attributes` combined with `dynamodb:Select` restricts which attributes may be read or written, and `dynamodb:ReturnValues` prevents a write from leaking the prior item state back to the caller.

- **Encryption at rest.** Always enabled, with three key options: an AWS owned key at no cost and no visibility, the AWS managed key `aws/dynamodb`, or a customer managed KMS key. Encryption covers the table, local and global secondary indexes, streams, on-demand backups, PITR backups, and global table replicas. It cannot be turned off, but unlike RDS the key type can be changed on a live table.

- **Encryption in transit.** TLS is enforced by the endpoint. There is no plaintext option and no configuration to get wrong. Requests are additionally SigV4-signed, so integrity and identity are bound to every call.

- **VPC gateway endpoint.** DynamoDB is reached over the public AWS endpoint by default even from inside a VPC. A gateway endpoint adds a route table entry keeping traffic on the AWS network at no cost, and it enables two enforcement points: an endpoint policy limiting which tables and actions can be reached through it, and an IAM or resource policy condition on `aws:SourceVpce` or `aws:SourceVpc` that denies access arriving from anywhere else. That pairing is the anti-exfiltration control.

- **Resource-based policies.** Tables and streams support resource policies, which is what enables cross-account access without assuming a role in the table owner's account, and what allows a `Deny` that no identity policy in another account can override.

- **DynamoDB Streams and Kinesis Data Streams.** Streams emit an ordered, item-level change record with configurable views including old image, new image, or both, retained 24 hours and consumed by Lambda or KCL. Kinesis Data Streams for DynamoDB is the alternative sink with longer retention. For security this is the mechanism for change forensics, near real-time detection on sensitive item mutations, and shipping an immutable change log to an Object Lock bucket.

- **Backup and recovery.** Point-in-time recovery gives per-second restore across a rolling 35 day window. On-demand backups are retained until deleted. Both restore to a new table, never in place, and both inherit encryption. AWS Backup integration adds vault-level controls and Vault Lock for WORM retention.

- **Global tables.** Multi-Region, multi-active replication with last-writer-wins conflict resolution. Each replica is encrypted with a key in its own Region, so a customer managed key deployment needs a key per Region and the replication traffic stays on the AWS backbone.

- **TTL.** A designated epoch-timestamp attribute triggers background deletion, typically within 48 hours of expiry. Deletions appear in Streams flagged as system deletes, which is how session and token expiry becomes auditable rather than silent.

- **Monitoring surface.** CloudTrail logs control plane calls by default and data plane calls (item-level reads and writes) only when data events are explicitly enabled, which is the usual gap. CloudWatch Contributor Insights surfaces the most accessed keys and is how you spot scraping or a hot tenant. Config tracks table configuration drift such as encryption key changes or PITR being disabled.

## DynamoDB versus adjacent data stores

| Service | Auth model | Encryption at rest | Item or row-level authorization | Change capture for audit | Network exposure |
|---|---|---|---|---|---|
| DynamoDB | IAM only, no DB users | Always on, key type changeable in place | IAM condition keys on partition key and attributes | DynamoDB Streams or Kinesis, item-level before and after images | API endpoint only, gateway endpoint optional |
| Aurora and RDS | Native DB users, plus optional IAM DB auth | Set at creation, immutable, snapshot copy to change | GRANT and row-level security inside the engine | Database Activity Streams, engine audit logs | Instance in a VPC subnet with security groups |
| DocumentDB | Native users, IAM auth on newer versions | Set at creation, immutable | Role-based inside the engine | Profiler and audit logs to CloudWatch | Cluster in a VPC subnet |
| Amazon Keyspaces | IAM only, or service-specific credentials | Always on, AWS owned or customer managed key | IAM on keyspace and table, no per-row conditions | No native item-level stream | API endpoint, interface endpoint optional |
| ElastiCache | Redis AUTH or IAM auth, or none | Optional, enabled at creation | Redis ACLs by key pattern | No durable change log | Cluster in a VPC subnet |
| S3 | IAM plus bucket and access point policies | Always on, SSE-S3, SSE-KMS, or DSSE-KMS | Prefix and object level via policy and access points | CloudTrail data events, S3 event notifications | Endpoint, gateway or interface endpoint optional |

## What gets tested

- **Per-tenant isolation in a shared table is `dynamodb:LeadingKeys`.** Expect distractors offering a separate table per tenant (works but does not scale and is not the asked-for answer) or a Lambda authorizer doing the filtering in code (moves enforcement out of IAM).

- **CloudTrail does not log item-level access by default.** If the requirement is knowing who read a specific item, the answer is enabling DynamoDB data events in a trail, not "CloudTrail already covers this."

- **Preventing exfiltration to a table in an attacker's account** is a VPC endpoint policy restricting which table ARNs are reachable, combined with an SCP or IAM condition on `aws:SourceVpce`. A security group answer is always wrong, DynamoDB has no ENI in the gateway endpoint case.

- **Encryption cannot be disabled**, so any question asking how to turn it off is testing whether you know it is unconditional. The valid change is switching between AWS owned, AWS managed, and customer managed keys, which is done in place with no downtime.

- **Customer managed key over AWS managed key** when the requirement mentions key policy control, key rotation on your schedule, CloudTrail visibility into individual `Decrypt` calls, or the ability to make data unreadable by disabling the key.

- **Streams to Lambda is the answer for real-time reaction to a data change**; PITR is the answer for recovering from a bad write or a ransomware-style mass delete; AWS Backup with Vault Lock is the answer when retention must be immutable and enforced against an administrator.

- **Blocking `Scan` is an IAM deny, not a table setting.** Scan abuse and data scraping detection maps to Contributor Insights plus a CloudWatch alarm on consumed read capacity.

- **Global tables need a KMS key in every replica Region.** A single-Region key ARN in a multi-Region answer is a distractor, the same trap as Aurora Global Database.

- **`dynamodb:ReturnValues` restriction** is the control that stops an `UpdateItem` caller from reading the prior value of an attribute they are not permitted to read.

## Limitations

- No joins, no foreign keys, and no cross-table referential integrity. Transactions exist through `TransactWriteItems` and `TransactGetItems` but are capped at 100 items or 4 MB and are limited to a single account and Region.

- Item size is capped at 400 KB, so large payloads must live in S3 with a pointer stored in the item. That splits the security boundary across two services and two sets of policies.

- Query flexibility is fixed by key design. Access patterns must be known before the table is modeled, and adding a new one usually means a new index or a migration rather than a new query.

- IAM condition keys operate on keys and attribute names, not attribute values. There is no way to express "deny reading items where classification equals restricted" through policy alone.

- Gateway endpoints work only for traffic originating inside the VPC and cannot be reached from on-premises over Direct Connect or VPN. That case requires an interface endpoint or routing through a proxy.

- Streams retain 24 hours only. Anything needing longer forensic retention must be forwarded to Kinesis, Firehose, or S3.

- Global tables resolve write conflicts with last-writer-wins and no application-level arbitration, which is a data-integrity consideration, not just a consistency one.

- Restores always create a new table, so recovery involves an application cutover and a fresh set of IAM policies and alarms pointed at the new ARN.