# Amazon FinSpace

Amazon FinSpace is a managed analytics platform built for financial services workloads, where the governing constraint is not just performance but regulatory recordkeeping: who accessed which dataset, when, under what entitlement, and whether the derived result can be reconstructed years later. The service has two distinct generations, and the distinction matters. The original offering combined a data catalog, entitlement engine, and managed Spark notebooks, and AWS has deprecated that dataset management and notebook capability. The current product is FinSpace with Managed kdb Insights, a managed environment for running kdb+ and q workloads, which is what a hedge fund or market-data team actually deploys today. Security-wise, Managed kdb Insights is a VPC-attached, cluster-based service: you provision an environment tied to a KMS key, place clusters into your subnets, mount shared and managed volumes, and control access through IAM and security groups. The thing to hold onto is that FinSpace is a managed compute environment for a specific engine, not a policy layer, so entitlements, encryption keys, and network placement are configured at the environment level and inherited by every cluster inside it.

## How it works

- **FinSpace environment.** The top-level resource. Created with a customer managed KMS key that is fixed for the life of the environment and encrypts everything the service stores, including databases, changesets, and volumes. Deleting the key makes the environment's data permanently unreadable, which is the intended crypto-shredding behavior and also the sharpest operational risk.

- **Managed kdb databases and changesets.** Data is loaded as changesets against a named database, each changeset being an immutable, versioned addition. Because changesets are append-only and versioned, the load history itself is the lineage record, which is what supports reconstruction requirements under SEC 17a-4 and FINRA recordkeeping rules.

- **Cluster types.** HDB clusters serve historical data, RDB clusters hold the current in-memory partition, gateway clusters route queries, GP clusters run general q processes, and Tickerplant clusters handle the real-time feed. Each is provisioned separately with its own capacity, cache configuration, and initialization script.

- **VPC attachment.** Clusters are placed into a VPC configuration specifying your VPC, subnets, security groups, and IP address type. Traffic to a cluster crosses your security groups, so reachability is a network control, not an IAM control. The environment also creates a transit gateway attachment for connectivity into the managed side.

- **Encryption.** At rest through the environment KMS key across all managed storage. In transit through TLS on the service API and on cluster connections. There is no unencrypted configuration.

- **IAM surface.** `finspace:*` actions govern environment, database, changeset, cluster, and scaling group operations. Access to run a query is not an IAM action, it is network reachability to the cluster plus whatever authentication the q code enforces, which is the same posture as ElastiCache.

- **Volumes and shared access.** Managed volumes provide shared filesystem storage across clusters in an environment, scoped by an availability zone identifier. Anything written to a shared volume is visible to every cluster mounting it, so volume mounting is an authorization decision.

- **Scaling groups and dedicated capacity.** Clusters can run on shared or dedicated capacity. Dedicated scaling groups give tenancy isolation, which is the answer when a control requires that compute not be shared.

- **Data views.** A cached, read-optimized snapshot of a database at a specific changeset, attached to HDB clusters. Views are what let multiple clusters read the same historical state without duplicating storage, and they pin a specific version, which is useful for reproducing a prior analysis.

- **Logging.** CloudTrail records control plane operations including environment creation, changeset ingestion, cluster creation, and key configuration. Cluster logs go to CloudWatch Logs. There is no per-query audit stream from the engine itself, so query-level attribution depends on gateway logging and application instrumentation.

- **Identity federation.** Environment and console access uses IAM, and organizations typically federate through IAM Identity Center or an external IdP such as Okta or Entra ID rather than provisioning users directly.

## FinSpace versus adjacent analytics platforms

| Service | Engine and workload | Where compute runs | Encryption key model | Data-level authorization | Query-level audit trail |
|---|---|---|---|---|---|
| FinSpace Managed kdb Insights | kdb+ and q, tick and time-series | Managed clusters attached to your VPC | Customer managed KMS key, fixed at environment creation | Network reachability plus q-level logic, no column or row policy engine | Cluster logs to CloudWatch, no native per-query log |
| Athena with Lake Formation | SQL over S3 | Serverless, AWS managed | SSE-KMS per bucket or catalog | Table, column, row, and cell filters enforced by Lake Formation | CloudTrail plus Lake Formation audit events |
| Redshift | SQL data warehouse | Cluster or serverless in your VPC | KMS or HSM, changeable via resize | GRANT, row-level security, dynamic data masking | Connection, user, and user activity logs |
| EMR | Spark, Hive, Presto | Cluster in your VPC | KMS per security configuration | Lake Formation or Ranger integration | Depends on the engine and integration |
| SageMaker Studio | Notebooks and ML | Managed instances, VPC-attached | KMS per domain and per volume | IAM plus whatever the data source enforces | CloudTrail on API, no query-level log |
| OpenSearch | Search and log analytics | Domain in your VPC or public endpoint | KMS at rest, node-to-node TLS | Fine-grained access control, index and field level | Audit logs to CloudWatch |

## What gets tested

- **The environment KMS key is immutable and load-bearing.** It cannot be changed after creation, and disabling or scheduling deletion of the key renders the environment inoperable. Any answer proposing a key rotation to a different CMK in place is wrong, the path is a new environment and a data migration.

- **Cluster access is a network control.** Restricting who can reach an HDB or gateway cluster is a security group and subnet question. IAM answers govern who can create, scale, or delete clusters, not who can run a query.

- **Changesets are immutable and versioned**, which is the recordkeeping answer for regulations requiring write-once retention and the ability to reproduce a historical state. Pair with a data view pinned to a changeset for reproducibility.

- **Dedicated scaling groups** are the answer when a compliance requirement forbids shared tenancy for compute.

- **CloudTrail covers the control plane only.** If the requirement is knowing which analyst ran which query against which dataset, expect the answer to involve gateway-level logging or application instrumentation, not a native FinSpace audit feature.

- **The legacy dataset management, entitlement, and notebook capability is deprecated.** Questions or documentation describing FinSpace column-level entitlements and managed Spark notebooks describe the retired generation. For governed, fine-grained data access on current architecture the answer is Lake Formation over S3, not FinSpace.

- **Cross-account and on-premises connectivity** goes through the VPC attachment and transit gateway, so hybrid access is a routing and security group design, with no public endpoint option to shortcut it.

## Limitations

- The dataset catalog, entitlement engine, and managed Spark notebooks from the original FinSpace product are deprecated. Anything relying on FinSpace for column-level or row-level entitlements needs a different service.

- No native per-query audit log from the engine. Meeting an access-attribution requirement means building it at the gateway or application layer.

- The environment KMS key cannot be changed, and losing it is unrecoverable. Key policy and key deletion protections are therefore a higher-severity control here than in most services.

- Managed kdb Insights is specific to kdb+ and q. It is not a general analytics platform, and there is no SQL interface for teams without q expertise.

- Regional availability is narrow relative to core services, which constrains data residency and DR design.

- Cost is driven by continuously running clusters and cached volumes rather than per-query consumption, so idle capacity is expensive and there is no scale-to-zero equivalent.

- kdb+ licensing is a separate commercial arrangement with KX. The AWS service does not remove that dependency.

- Cluster initialization scripts and q code run with whatever access the cluster has to mounted volumes and databases. There is no in-engine least-privilege model, so code review and volume scoping carry the weight that a policy engine would carry elsewhere.