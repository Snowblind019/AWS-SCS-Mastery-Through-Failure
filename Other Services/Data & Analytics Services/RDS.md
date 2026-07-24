# Amazon RDS

Amazon RDS is AWS's managed relational database service, supporting MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, and Db2, with Aurora as a separate cloud-native engine sharing much of the same control plane. AWS operates the host, the OS, and the engine patching; you own everything above and around it, which under the shared responsibility model means network placement, encryption key selection, credential management, parameter configuration, and audit logging are all yours. That split is where the exam lives and where real breaches come from, because RDS ships with several defaults that are convenient rather than secure: an instance can be made publicly accessible with a checkbox, TLS is available but not required until you force it in a parameter group, and encryption at rest is a creation-time decision that cannot be reversed. The instance itself is a real host with an ENI in your subnet, so unlike DynamoDB or S3 it has a network attack surface governed by security groups. The thing to hold onto is that RDS security decomposes into four independent layers, network reachability, at-rest encryption chosen once at creation, credential mechanism, and audit configuration, and each is enforced by a different control that must be set deliberately.

## How it works

- **Instance and subnet placement.** An instance lives in a DB subnet group spanning at least two AZs. Its ENIs sit in those subnets, so the security group on the instance is the primary reachability control. The `PubliclyAccessible` flag determines whether the endpoint resolves to a public IP; combined with a route to an internet gateway and a permissive security group, that is the classic exposure path.

- **Encryption at rest.** Selected at instance creation with an AWS managed key or a customer managed KMS key. Covers the underlying storage, automated backups, snapshots, read replicas, and logs. It cannot be enabled, disabled, or re-keyed in place. The only path is snapshot, copy the snapshot specifying the target key, restore from the copy. Oracle and SQL Server additionally support engine-native TDE through an option group, which is a separate mechanism from RDS storage encryption.

- **Encryption in transit.** TLS is available on every engine but optional by default. Enforcement is a parameter group setting: `rds.force_ssl` for PostgreSQL, `require_secure_transport` for MySQL and MariaDB, an option group setting for Oracle and SQL Server. Clients validate against the regional or global RDS CA bundle, and CA rotation is a recurring operational event that breaks clients pinning an expired bundle.

- **IAM database authentication.** Supported on MySQL, MariaDB, and PostgreSQL, not on Oracle, SQL Server, or Db2. A client calls `generate-db-auth-token` and presents the token as the password. Tokens expire in 15 minutes, authorization is the `rds-db:connect` action on a `dbuser` ARN, and no static password exists. Throughput is limited to roughly 200 new connections per second, so it fits administrative and serverless access rather than a high-churn connection pool.

- **Secrets Manager integration.** Master credentials can be managed directly by RDS with fully managed rotation, or stored as a secret with a rotation Lambda. The secret's resource policy and its KMS key policy are the access controls, and every retrieval is a CloudTrail event.

- **Kerberos and directory authentication.** RDS integrates with AWS Managed Microsoft AD for Kerberos authentication on PostgreSQL, MySQL, Oracle, SQL Server, and Db2, which is the answer when the requirement is domain-joined authentication rather than IAM.

- **RDS Proxy.** A managed connection pooler placed between the application and the instance. It retrieves credentials from Secrets Manager, can require IAM authentication, terminates TLS, and preserves connections across failover. It is the standard remedy for Lambda connection exhaustion combined with credential hygiene.

- **High availability.** Multi-AZ instance deployment replicates synchronously to a standby in another AZ with automatic failover in one to two minutes. Multi-AZ DB cluster deployment adds two readable standbys and cuts failover to under 35 seconds. Read replicas are asynchronous, can be cross-Region, and are promoted manually, making them a DR mechanism rather than an availability one.

- **Backups and snapshots.** Automated backups within a retention window of 0 to 35 days enable point-in-time recovery. Manual snapshots persist until deleted. Both inherit the instance's encryption. Snapshots can be shared with specific accounts or made public, and cross-Region copy requires a key in the destination Region. AWS Backup adds vault-level policy and Vault Lock for immutable retention.

- **Logging.** Engine logs including error, general, slow query, and audit logs can be published to CloudWatch Logs. Database Activity Streams provide near real-time, KMS-encrypted, engine-level activity to Kinesis for Oracle, SQL Server, and Aurora, with synchronous mode blocking database activity if the stream cannot be written. CloudTrail records only control plane API calls such as `ModifyDBInstance` and `CreateDBSnapshot`, never SQL.

- **Patching and maintenance.** AWS applies OS and engine patches within a maintenance window you define. Blue/Green Deployments create a synchronized staging environment for testing an engine or schema upgrade and switching over with minimal downtime.

## RDS versus adjacent database options

| Option | Operational surface | At-rest encryption | Identity-based auth | Query-level audit | Failover behavior |
|---|---|---|---|---|---|
| RDS (MySQL, PostgreSQL, MariaDB) | Managed instance in your VPC | KMS, creation-time, immutable | IAM DB auth plus native users and Kerberos | Engine audit logs to CloudWatch, no DAS | Multi-AZ, one to two minutes |
| RDS (Oracle, SQL Server, Db2) | Managed instance in your VPC | KMS plus optional engine TDE | Kerberos and native, no IAM DB auth | Database Activity Streams supported | Multi-AZ, one to two minutes |
| Aurora | Managed cluster, shared storage volume | KMS, creation-time, immutable | IAM DB auth plus native users | Database Activity Streams | Replica promotion, under 30 seconds |
| RDS Custom | Managed with customer OS and DB access | KMS, creation-time | Native, plus whatever you configure | Whatever you configure | Limited automation |
| Database on EC2 | You own everything | EBS encryption you configure | Whatever you configure | Whatever you configure | You build it |
| DynamoDB | No instances, API only | Always on, key changeable in place | IAM only, no DB users | CloudTrail data events plus Streams | Managed, transparent |

## What gets tested

- **Encryption cannot be added to an existing unencrypted instance.** The answer is always snapshot, copy with the KMS key, restore. The same three steps apply to changing keys and to encrypting a previously unencrypted read replica.

- **Encrypted snapshot sharing requires sharing the key.** A snapshot encrypted with the default `aws/rds` managed key cannot be shared with another account at all. Copy to a customer managed key, add the target account as a key user, then share. Cross-Region copy needs a key in the destination Region.

- **Public snapshot sharing is a top exposure pattern.** Encrypted snapshots cannot be made public, which is itself a control. Config rules and Security Hub flag public snapshots, and the remediation is `ModifyDBSnapshotAttribute` to remove the `all` value.

- **Forcing TLS is a parameter group change**, not a security group or IAM change, and it requires a reboot for a static parameter. Watch for the cluster parameter group versus DB parameter group distinction on Aurora.

- **IAM DB auth is unavailable on Oracle and SQL Server.** A question specifying one of those engines plus a no-static-password requirement points to Kerberos with AWS Managed Microsoft AD, or to Secrets Manager rotation with RDS Proxy.

- **CloudTrail does not record SQL.** For who ran which query, the answer is Database Activity Streams where supported, or the engine audit log exported to CloudWatch Logs. Synchronous DAS is the answer when activity must never proceed unaudited.

- **Multi-AZ versus read replica.** Multi-AZ is synchronous, same-Region, and for availability. Read replicas are asynchronous, can be cross-Region, and are for read scaling and DR. Multi-AZ DB cluster is the answer when the RTO requirement is tighter than a minute and readable standbys are wanted.

- **RDS Proxy** is the answer for Lambda connection exhaustion, for enforcing IAM authentication in front of an engine that supports it, and for shortening failover impact on application connections.

- **Public accessibility remediation** is disabling the flag and scoping the security group to the application security group. NACLs are supporting, never primary.

## Limitations

- Encryption at rest is fixed at creation. Enabling it, disabling it, or changing the key all require a snapshot copy and restore, which produces a new endpoint and an application cutover.

- No OS access and no superuser. Extensions, plugins, and modules are limited to the supported list per engine version. Workloads needing filesystem access, custom agents, or unsupported extensions require RDS Custom or EC2.

- IAM database authentication is engine-limited and connection-rate-limited, so it does not substitute for a connection pool on a high-traffic application.

- Automated backups are deleted when the instance is deleted unless a final snapshot is taken, and the retention ceiling is 35 days. Longer retention requires manual snapshots or AWS Backup.

- Multi-AZ failover takes one to two minutes on instance deployments and drops in-flight connections. Applications need retry logic; the failover is transparent to DNS, not to open sessions.

- Cross-Region read replicas replicate asynchronously, so a Region loss can lose recent transactions, and promotion is manual unless orchestrated.

- Engine audit logging capability varies substantially by engine. MySQL and PostgreSQL rely on plugins configured through parameter and option groups, and Database Activity Streams is not available for the open source engines on RDS.

- Restores always create a new instance, never in place, so recovery is a cutover involving new endpoints, security groups, parameter groups, and monitoring targets.

- Maintenance windows apply patches on AWS's schedule within your window. You control when, not whether, and a deferred patch eventually becomes mandatory.