# Amazon Timestream

Amazon Timestream is AWS's purpose-built time series database, offered as two distinct products that share a name and almost nothing else. Timestream for LiveAnalytics is the serverless, AWS-native engine with a memory and magnetic tier, its own SQL dialect, and no instances to manage. Timestream for InfluxDB is a managed InfluxDB deployment running on instances in your VPC, for teams already invested in InfluxQL, Flux, and the InfluxDB ecosystem. The security models differ accordingly: LiveAnalytics is an API service authorized purely by IAM with no network surface beyond an optional PrivateLink endpoint, while InfluxDB is a database instance with an endpoint, security groups, and native InfluxDB tokens for data plane authentication. For security work, the value in either is the same: time series is where you see rate-of-change signals, failed authentication trends, egress volume patterns, and resource consumption anomalies, at a fraction of the cost of keeping the equivalent raw logs hot. The thing to hold onto is that which Timestream you mean determines whether the control surface is IAM alone or IAM plus network plus a separate token system.

## How it works

- **Data model (LiveAnalytics).** Records consist of a timestamp, dimensions (the identifying attributes such as instance ID, Region, or account), a measure name, and a measure value. Multi-measure records group several values under one timestamp, which is both cheaper and faster to query than the single-measure form.

- **Storage tiering (LiveAnalytics).** A memory store holds recent data for fast writes and point queries, and a magnetic store holds older data for cheaper long-term querying. Retention is configured per table for each tier, and data aging out of the magnetic store is deleted permanently and automatically. There is no archive tier and no recycle bin.

- **Ingestion paths (LiveAnalytics).** The `WriteRecords` API directly, IoT Core rules, Kinesis Data Firehose, Lambda, CloudWatch metric streams, and Telegraf. Every path is an IAM-authorized API call with TLS enforced.

- **Scheduled queries (LiveAnalytics).** A precomputed aggregation written back to a destination table on a schedule, used to keep dashboards cheap. The scheduled query runs under an IAM role you supply and can notify an SNS topic on failure, and that role is a standing read grant on the source table.

- **Authorization (LiveAnalytics).** Purely IAM. Actions include `timestream:WriteRecords`, `timestream:Select`, `timestream:DescribeEndpoints`, and `timestream:SelectValues`, scoped to database and table ARNs. There are no database users. Note that `DescribeEndpoints` is required by every SDK call, so a policy omitting it breaks all access regardless of other grants.

- **Encryption (LiveAnalytics).** Always on at rest with an AWS managed key by default or a customer managed KMS key specified per database. TLS on every API call with no plaintext option.

- **Network (LiveAnalytics).** No VPC placement. The service is reached through a public API endpoint or an interface VPC endpoint, and an `aws:SourceVpce` condition is what restricts access to your network.

- **Timestream for InfluxDB.** Provisioned as a DB instance into a VPC subnet with a security group, optionally Multi-AZ with a standby. Storage is encrypted with KMS. Authentication is InfluxDB's own token model, with the initial admin token issued at creation and stored in Secrets Manager. IAM governs instance lifecycle, not data access.

- **Logging.** CloudTrail records control plane operations for both products: database and table creation, retention changes, KMS key configuration, scheduled query definitions, and instance lifecycle. For LiveAnalytics, `WriteRecords` and query calls are data plane operations and are not individually logged, so query attribution is unavailable. InfluxDB engine logs can be delivered to an S3 bucket.

- **Query and visualization integration.** Grafana has native plugins for both. QuickSight connects to LiveAnalytics. Athena can reach LiveAnalytics through a federated connector. Each of these introduces its own identity and caching layer on top of the Timestream authorization model.

## Timestream versus adjacent time series and metrics stores

| Option | Ingestion model | Authentication | Encryption at rest | Retention control | Network surface | Query attribution |
|---|---|---|---|---|---|---|
| Timestream for LiveAnalytics | API writes, serverless | IAM only | KMS, customer managed key optional | Per table, memory and magnetic tiers | API endpoint, PrivateLink optional | None, data plane not in CloudTrail |
| Timestream for InfluxDB | InfluxDB line protocol | InfluxDB tokens, admin token in Secrets Manager | KMS | InfluxDB retention policies | Instance in your VPC with security groups | Engine logs to S3 |
| Amazon Managed Service for Prometheus | Remote write from a collector | IAM SigV4 | KMS, customer managed key optional | Fixed at 150 days | API endpoint, PrivateLink optional | None |
| CloudWatch metrics | Push from agents and services | IAM | AWS managed | 15 months with rollup, not configurable | API endpoint | CloudTrail on API, not per datapoint |
| OpenSearch | Indexed documents | Fine-grained access control, IAM, SAML | KMS | Index lifecycle management | Domain in VPC or public | Domain audit logs, full |
| DynamoDB with a time-ordered key | API writes | IAM, with condition keys | Always on, key changeable | TTL attribute | API endpoint, gateway endpoint optional | CloudTrail data events available |

## What gets tested

- **Which Timestream.** LiveAnalytics has no VPC placement and no database users, InfluxDB has both. A question mentioning security groups, subnets, or database credentials for Timestream is about the InfluxDB variant.

- **IAM is the only data plane control on LiveAnalytics.** There is no row, dimension, or measure-level authorization, so isolating tenants means separate databases or separate tables with separate IAM scoping, and even that is coarse.

- **`timestream:DescribeEndpoints` is required for every operation.** A least-privilege policy that grants only `Select` or `WriteRecords` fails, and this specific gotcha appears in policy-troubleshooting questions.

- **Retention expiry is permanent deletion.** There is no backup, no snapshot, and no point-in-time restore for LiveAnalytics. If a requirement involves retaining telemetry beyond the magnetic store window or recovering deleted data, the answer is exporting to S3 with `UNLOAD` before expiry.

- **Customer managed KMS key per database** is the answer for key control and for the ability to render data unreadable by disabling the key. It is set per database and is the crypto-shredding mechanism in a service with no other deletion controls.

- **Interface VPC endpoint plus an `aws:SourceVpce` condition** is the answer for keeping LiveAnalytics traffic off the internet, since there is no security group to attach.

- **No query-level audit.** If a question requires knowing who queried telemetry containing sensitive dimensions, LiveAnalytics cannot answer it, which points toward OpenSearch or a service with data event logging.

- **Scheduled query IAM roles are standing grants.** A scheduled query role reads the source table on every run without a human in the loop, so its scope and its failure SNS topic are both worth reviewing.

- **Timestream for InfluxDB admin token handling.** The initial token is written to Secrets Manager, and rotating it is an InfluxDB-side operation, not an AWS-managed rotation. That gap is the credential-management answer.

- **Metrics are not logs.** Timestream shows the shape of an anomaly over time. It does not carry the identity, request context, or payload needed to attribute it, which still requires CloudTrail or the source logs.

## Limitations

- No backup, snapshot, or point-in-time restore for LiveAnalytics. Data lost to a retention misconfiguration or an accidental table deletion is unrecoverable, and the only preventive control is restricting `timestream:DeleteTable` and `timestream:UpdateTable`.

- No data plane logging on LiveAnalytics, so there is no record of who read or wrote what. Attribution depends entirely on the identity of the calling role and whatever the caller logs itself.

- Authorization granularity stops at the table. Dimensions cannot be used as an access boundary, so a table holding telemetry for multiple tenants is readable in full by anyone with `Select` on it.

- High-cardinality dimensions degrade query performance and inflate cost. Putting user IDs, request IDs, or source IPs in dimensions is an anti-pattern, which limits how far time series can substitute for logs during an investigation.

- Query cost is per gigabyte scanned, so a dashboard without a bounded time predicate or without binning is expensive and can become an unintentional denial of budget.

- Records arriving with timestamps outside the memory store's retention window are rejected unless magnetic store writes are enabled, which makes late-arriving or replayed telemetry a real failure mode during recovery from an ingestion outage.

- Timestream for LiveAnalytics availability and new-customer access have changed since launch, so confirm current AWS guidance before designing new architecture on it rather than on Managed Prometheus or Timestream for InfluxDB.

- Timestream for InfluxDB inherits InfluxDB's own security model, including token scoping and organization structure, which is a second authorization system to design, audit, and rotate alongside IAM.

- Not suitable for transactional workloads, updates, or deletes of individual records. Writes are effectively append-only, and correcting bad data means writing a new value rather than fixing the old one.