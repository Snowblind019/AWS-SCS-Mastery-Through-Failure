# Amazon Redshift

Amazon Redshift is AWS's managed columnar data warehouse for OLAP workloads: petabyte-scale joins, aggregations, and reporting queries over structured and semi-structured data. Its security significance comes from what it concentrates. A warehouse is where PII, transaction history, clickstream, and behavioral data from a dozen upstream systems get joined into one queryable surface, which makes it the single highest-value target in most estates and also the place where an over-permissioned analyst does the most damage without ever triggering a network alert. Redshift's controls come from two separate systems that must be reasoned about together: AWS-layer controls covering VPC placement, KMS encryption, IAM authentication, and the roles used for `COPY` and `UNLOAD`, and SQL-layer controls covering database users, roles, `GRANT` statements, row-level security policies, dynamic data masking, and column privileges. Neither one alone is sufficient, and the exam consistently tests whether you know which layer a given requirement lands in. The thing to hold onto is that IAM governs who can reach the warehouse and where data can move to or from, while SQL grants govern what a connected principal can actually see.

## How it works

- **Provisioned clusters and Serverless.** A provisioned cluster is a leader node plus compute nodes with fixed capacity, placed in a cluster subnet group in your VPC. Redshift Serverless replaces that with a namespace holding the database, its objects, its encryption key, and its IAM role associations, plus a workgroup holding compute capacity and network configuration. The namespace and workgroup split is what makes Serverless useful for departmental isolation.

- **Network placement.** Clusters and workgroups sit in your VPC with security groups on the endpoint. Publicly accessible is a flag that, combined with a route to an internet gateway, exposes the endpoint. Enhanced VPC routing forces all `COPY` and `UNLOAD` traffic through your VPC rather than the AWS network, which is what lets endpoint policies and flow logs see and control data movement to S3.

- **Encryption at rest.** Enabled with an AWS managed key or a customer managed KMS key, covering all user data, temporary and spill storage, and snapshots. On provisioned clusters encryption can be enabled or the key changed on an existing cluster through a background migration, unlike RDS and Aurora where it is creation-time only. Redshift also supports a hardware security module configuration for key management.

- **Encryption in transit.** TLS is available and enforced through the `require_SSL` parameter in the cluster parameter group, with clients configuring `sslmode` in the JDBC or ODBC connection string. `COPY` and `UNLOAD` to S3 use HTTPS.

- **Authentication.** Native database users with passwords, IAM authentication through `GetClusterCredentials` or `GetCredentials` for Serverless returning temporary credentials, federated single sign-on through SAML or IAM Identity Center with the IdP identity mapped into a Redshift session, and Secrets Manager for storing and rotating credentials that must remain passwords.

- **SQL-layer authorization.** Role-based access control with `CREATE ROLE` and `GRANT`, schema, table, and column-level privileges, row-level security policies attached to tables and evaluated per session, and dynamic data masking applying a masking expression to a column based on the querying role. Dynamic data masking is what lets one physical column render as full value, partial value, or redacted depending on who is asking.

- **IAM roles for data movement.** `COPY`, `UNLOAD`, Spectrum, and federated query use IAM roles associated with the cluster or namespace, referenced explicitly in the SQL statement. These roles are the exfiltration surface: an `UNLOAD` writes query results to S3, so the role's write scope determines where warehouse data can land.

- **Redshift Spectrum.** Queries data in S3 without loading it, using the Glue Data Catalog. Authorization comes from the Spectrum IAM role plus Lake Formation permissions on the external schema, which is a different enforcement path from internal tables.

- **Data sharing.** Producer clusters share live data with consumer clusters or accounts without copying, governed by datashare objects and, for cross-account sharing, an authorization step by the producer and an association by the consumer. Lake Formation can govern datashares for finer-grained control.

- **Snapshots and backups.** Automated snapshots on a schedule with configurable retention, manual snapshots persisting until deleted, all inheriting cluster encryption. Snapshots can be shared with specific accounts and copied cross-Region with a destination-Region key. Cross-Region snapshot copy for an encrypted cluster requires a snapshot copy grant.

- **Audit logging.** Three log types written to S3 or CloudWatch Logs: connection log for connection and disconnection attempts, user log for changes to database users, and user activity log capturing every query executed with the submitting user. User activity logging requires the `enable_user_activity_logging` parameter and is off by default. CloudTrail covers control plane operations only.

## Redshift versus adjacent analytics and storage options

| Option | Workload | Encryption at rest | Row and column controls | Query-level audit | Where data lives |
|---|---|---|---|---|---|
| Amazon Redshift | OLAP, large joins and aggregations | KMS or HSM, changeable on existing cluster | RLS policies, column grants, dynamic data masking | User activity log, off by default | Managed cluster storage plus RMS |
| Athena with Lake Formation | Ad hoc SQL over S3 | Per-bucket SSE-S3 or SSE-KMS | Lake Formation row, column, and cell filters | CloudTrail plus Lake Formation events | S3, in place |
| Aurora and RDS | OLTP, transactional | KMS, creation-time, immutable | Engine GRANT and RLS | DAS or engine audit logs | Managed instance storage |
| OpenSearch | Search and log analytics | KMS | Fine-grained access control, document and field level | Domain audit logs | Domain storage |
| EMR | Distributed processing | KMS per security configuration | Lake Formation or Ranger | Engine dependent | S3 or HDFS |
| DynamoDB | Key-value at scale | Always on, key changeable | IAM condition keys on partition key | CloudTrail data events | Managed, no instances |

## What gets tested

- **Which layer enforces the requirement.** Restricting an analyst to certain rows is a Redshift RLS policy, not an IAM policy. Restricting which S3 buckets an `UNLOAD` can write to is an IAM role scope, not a `GRANT`. Distractors routinely swap the two.

- **Dynamic data masking is the answer for showing different views of the same column to different roles**, when the alternative offered is maintaining separate redacted tables or views.

- **Enhanced VPC routing is the answer for controlling and monitoring data egress** from the warehouse, because it forces `COPY` and `UNLOAD` through your VPC where endpoint policies, flow logs, and NAT controls apply. Without it that traffic bypasses your network entirely.

- **User activity logging is disabled by default** and requires a parameter group change plus a reboot. If a question asks how to know which queries an analyst ran, the answer includes enabling it, not "check CloudTrail."

- **Unlike RDS and Aurora, encryption can be enabled on an existing Redshift cluster** and the key can be changed. This is a genuine engine-specific distinction and a common trap when the same question stem is reused across services.

- **Cross-Region snapshot copy of an encrypted cluster requires a snapshot copy grant** in the destination Region, which is a Redshift-specific mechanism rather than the plain key-in-destination-Region pattern.

- **IAM authentication over stored passwords.** `GetClusterCredentials` issues temporary credentials tied to an IAM identity and can auto-create the database user and assign groups, which is the answer when static credentials must be eliminated and federated identity must map to database roles.

- **Datashare authorization is two-sided.** The producer authorizes, the consumer associates. A one-sided answer is wrong, and Lake Formation is the answer when the shared data needs column or row filtering.

- **Public accessibility plus an open security group** is the exposure Config and Security Hub flag, and the remediation is disabling the flag and scoping the security group, with enhanced VPC routing as the companion control for egress.

- **Spectrum authorization runs through Lake Formation and the Spectrum IAM role**, not through internal Redshift grants, so an answer about `GRANT` on an external schema is incomplete.

## Limitations

- Not built for low-latency point lookups or high-concurrency OLTP. Sub-100 millisecond single-row reads belong in DynamoDB or Aurora, and transactional write patterns will perform poorly regardless of tuning.

- Row-level security and dynamic data masking are evaluated per session against the current role. A user with a superuser or overly broad role bypasses both, so the controls depend on disciplined role design rather than being enforced beneath the SQL layer.

- User activity logs contain full query text, which means they contain any literal values in queries, including credentials pasted into SQL and PII in `WHERE` clauses. The audit log is itself sensitive data requiring encryption and restricted access.

- `UNLOAD` is a legitimate feature and an exfiltration primitive. Any role permitted to run it can write warehouse contents to S3 at scale, and the only meaningful control is the scope of the associated IAM role plus enhanced VPC routing.

- Concurrency is bounded by workload management queues and slots. A single expensive query can starve others, and the isolation between tenants sharing a cluster is resource-level rather than security-level.

- Cross-Region and cross-account data sharing move governance responsibility to two parties. Revoking a datashare stops future reads but does nothing about data the consumer has already materialized.

- Serverless namespaces isolate objects and keys but share the Redshift service surface, and per-workgroup network configuration is the only network boundary between them.

- Snapshot restore creates a new cluster or namespace, so recovery involves new endpoints and a fresh application of security groups, parameter groups, and IAM role associations.