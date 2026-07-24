# Amazon Aurora

Amazon Aurora is AWS's cloud-native relational database engine, wire-compatible with MySQL and PostgreSQL but rebuilt so that compute and storage are separate services rather than one instance with an attached volume. A cluster is a set of stateless compute instances pointed at a shared, distributed storage volume that replicates six copies of every write across three Availability Zones, self-heals damaged segments, and streams continuously to S3. From a security perspective this changes where the controls live: encryption at rest, backups, snapshots, and replica consistency are properties of the cluster volume, not of any individual instance, so a replica can never be less encrypted or less current than its writer. Everything else you would secure on RDS still applies, including KMS key selection at creation, TLS enforcement through parameter groups, IAM database authentication, Secrets Manager rotation, and subnet plus security group isolation. The thing to hold onto is that Aurora security is RDS security applied at the cluster level, so encryption and snapshot decisions are made once at creation and inherited by everything downstream.

## How it works

- **Decoupled storage layer.** The cluster volume is a purpose-built distributed store, not EBS. It autoscales in 10 GB increments up to 128 TiB, requires no provisioned IOPS, and tolerates the loss of an entire AZ plus one additional copy without losing write availability. You do not patch, mount, or reach it directly.

- **Cluster endpoints.** A writer endpoint always resolves to the current primary, a reader endpoint load-balances across replicas, and custom endpoints group instances by role. Security groups and the DB subnet group are attached to the cluster, so network policy is uniform across instances.

- **Encryption at rest.** Chosen at cluster creation with either the AWS managed key `aws/rds` or a customer managed KMS key. It covers the volume, automated backups, snapshots, replicas, and exported logs. It cannot be enabled, disabled, or re-keyed in place. The only path is snapshot, copy the snapshot with the target key, restore.

- **Encryption in transit.** TLS is available by default but not mandatory until you force it. For Aurora PostgreSQL set `rds.force_ssl=1` in the cluster parameter group; for Aurora MySQL set `require_secure_transport=ON`. Clients validate against the regional RDS CA bundle.

- **IAM database authentication.** Enabled per cluster, then mapped to a database user granted `rds_iam` (PostgreSQL) or the `AWSAuthenticationPlugin` (MySQL). The client calls `generate-db-auth-token` and uses the token as the password. Tokens live 15 minutes, the IAM policy action is `rds-db:connect` on a `dbuser` ARN, and no static credential is stored anywhere. Throughput is capped at roughly 200 new connections per second, so it suits admin and Lambda access more than high-churn connection pools.

- **Secrets Manager integration.** Master and application credentials can be stored as a Secrets Manager secret with managed rotation, either a Lambda rotation function or fully managed master user password rotation. Access is controlled by the secret's resource policy and the KMS key policy, and every retrieval is a CloudTrail event.

- **Network isolation.** The cluster lives in a DB subnet group spanning at least two AZs. Keep those subnets private, control ingress with a security group referencing the application security group rather than CIDR ranges, and leave the public accessibility flag off. That flag plus a permissive security group is the only way an Aurora cluster becomes internet-reachable.

- **Aurora Replicas.** Up to 15 per region, sharing the same storage volume, so replication lag is typically well under 100 milliseconds and no data is copied between instances. Failover promotes an existing replica in under 30 seconds using tier priority, with no standby to synchronize.

- **Aurora Global Database.** One primary region and up to five read-only secondary regions, replicated through the storage layer with typical lag under one second. Supports managed planned failover and unplanned detach and promote. Each region needs its own KMS key, since KMS keys are regional.

- **Backup and recovery.** Continuous backup to S3 with no backup window and no performance penalty, retention of 1 to 35 days, and point-in-time recovery to any second in that window. Aurora MySQL additionally supports Backtrack, an in-place rewind of the cluster volume that does not create a new cluster.

- **Audit and activity logging.** Engine audit logs (`server_audit` for MySQL, `pgaudit` for PostgreSQL) plus error, general, and slow query logs can be exported to CloudWatch Logs. Database Activity Streams push near real-time, KMS-encrypted SQL activity to Kinesis Data Streams for SIEM ingestion, and in synchronous mode will block database activity if the stream cannot be written, which is the compliance-grade option.

- **RDS Proxy.** A managed connection pooler in front of the cluster. It pulls credentials from Secrets Manager, can require IAM authentication, enforces TLS to the client, and preserves connections across failover. It is the standard answer for Lambda connection exhaustion combined with credential hygiene.

- **Aurora Serverless v2.** Scales in fine-grained Aurora Capacity Units and inherits the full security surface: KMS encryption, VPC isolation, IAM authentication, snapshots, and audit logging all behave the same as provisioned.

## Aurora versus sibling data services

| Service | Data model | Encryption at rest | Native identity-based auth | Query-level activity capture | Cross-Region story |
|---|---|---|---|---|---|
| Aurora | Relational, MySQL or PostgreSQL compatible | KMS, set at creation, immutable | IAM DB auth plus native DB users | Database Activity Streams, engine audit logs | Global Database, single writer region |
| RDS (MySQL, PostgreSQL, MariaDB) | Relational, same engines | KMS, set at creation, immutable | IAM DB auth plus native DB users | Engine audit logs only, no DAS for open source engines | Cross-Region read replica, manual promotion |
| RDS for Oracle or SQL Server | Relational, commercial engines | KMS, TDE also available | Kerberos and native, IAM DB auth not supported | Database Activity Streams supported for Oracle and SQL Server | Cross-Region read replica |
| DynamoDB | Key-value and document | KMS, always on, key can be changed | IAM only, no database users | CloudTrail data events for item-level access | Global Tables, multi-Region multi-active writes |
| DocumentDB | MongoDB-compatible document | KMS, set at creation, immutable | Native users, IAM auth supported on newer versions | Profiler and audit logs to CloudWatch | Global Clusters, single writer region |
| Redshift | Columnar analytics warehouse | KMS or HSM, can be toggled with cluster resize | IAM credentials via GetClusterCredentials | Connection, user, and user activity logs to S3 or CloudWatch | Cross-Region snapshot copy, no synchronous global cluster |

## What gets tested

- **Encryption cannot be added to an existing unencrypted cluster.** The exam answer is always: take a snapshot, copy the snapshot specifying the KMS key, restore from the copy. The same three-step path is the answer for changing keys.

- **Encrypted snapshot sharing requires sharing the key.** A snapshot encrypted with the AWS managed `aws/rds` key cannot be shared with another account at all. Copy it to a customer managed key first, add the target account as a key user in the key policy, then share the snapshot.

- **Cross-Region snapshot copy needs a key in the destination Region.** KMS keys do not cross Regions. Expect a distractor that reuses the source Region key ARN.

- **IAM DB auth versus Secrets Manager.** Pick IAM DB auth when the requirement is that no long-lived database password exists anywhere. Pick Secrets Manager when the requirement is automatic rotation of an application credential the app still presents as a password. Pick RDS Proxy when the question adds Lambda scaling or connection exhaustion on top of either.

- **CloudTrail does not see SQL.** CloudTrail records control plane calls such as `CreateDBCluster` and `ModifyDBInstance`. For who ran which query, the answer is Database Activity Streams or the engine audit log exported to CloudWatch Logs.

- **Forcing TLS is a parameter group change**, not a security group or IAM change. Watch for the cluster parameter group versus DB parameter group distinction: `rds.force_ssl` and `require_secure_transport` are cluster-level.

- **RPO and RTO ladder.** Multi-AZ replica failover for in-Region availability at roughly 30 seconds, cross-Region Aurora Global Database for sub-second RPO and roughly one minute RTO on managed failover, snapshot copy for cheap non-urgent DR. Backtrack answers "undo a bad write on Aurora MySQL without restoring."

- **Public exposure.** Security Hub and Config flag the combination of a publicly accessible cluster and an open security group. Remediation is disabling public accessibility and scoping the security group to the application security group, not adding a NACL.

- **Synchronous Database Activity Streams** is the choice when the requirement says database activity must never proceed unaudited. Asynchronous favors availability over completeness.

## Limitations

- One writer per cluster and one writer Region in a Global Database. There is no multi-Region multi-active write model; write forwarding from secondary Regions still sends the write to the primary and adds latency.

- Encryption at rest is fixed at creation. You cannot enable it, disable it, or rotate to a different KMS key without a snapshot copy and restore, which means a new cluster endpoint.

- Storage tops out at 128 TiB per cluster volume, and 15 Aurora Replicas per Region.

- No operating system access, no superuser, and only the extensions and plugins AWS ships for the engine version. Workloads needing custom modules or filesystem access belong on EC2.

- Backtrack is Aurora MySQL only, is not a substitute for backups, and must be enabled at creation.

- Database Activity Streams support depends on engine and version, and synchronous mode can stall the database if the Kinesis stream is unavailable, which is the intended failure mode but still an availability trade.

- Secondary Global Database Regions are read-only until promoted, and promotion of an unplanned failover breaks replication, requiring the global cluster to be rebuilt.

- Serverless v2 does not support every provisioned feature set, and scaling to zero introduces cold-start latency on the first connection.

- IAM database authentication throughput is limited, so it does not replace a connection pool for high-volume application traffic.