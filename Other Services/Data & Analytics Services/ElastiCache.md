# Amazon ElastiCache

Amazon ElastiCache is a managed in-memory data store offered in three flavors: ElastiCache for Valkey, ElastiCache for Redis OSS, and ElastiCache for Memcached. Unlike DynamoDB or S3, it is not an API service. It runs as nodes with ENIs inside your VPC, reachable on a TCP port, which means it carries a real network attack surface and is governed by subnet placement and security groups rather than by IAM alone. That distinction drives everything about securing it: the AWS control plane is IAM-authorized, but the data plane historically was not, so a caller who can reach the port can often read and write every key in the cache. What lives in that cache is usually the most sensitive short-lived data in the architecture, session state, JWTs, revocation lists, rate-limit counters, and cached query results containing PII. The thing to hold onto is that ElastiCache is the one managed data store where network reachability, not policy, is the primary control, so private subnets plus a tightly scoped security group plus encryption plus authentication all have to be configured deliberately because several of them default off.

## How it works

- **Deployment model.** A cluster is one or more nodes placed into a cache subnet group spanning your chosen AZs. Each node gets an ENI in a subnet and a DNS endpoint. Redis OSS and Valkey use port 6379, Memcached uses 11211. Serverless caches remove node sizing but keep the same VPC-attached, endpoint-based model.

- **Cluster topology.** Cluster mode disabled gives one shard with a primary and up to five replicas. Cluster mode enabled shards the keyspace across up to 500 shards, each with its own primary and replicas. Multi-AZ with automatic failover promotes a replica when the primary fails, typically in well under a minute.

- **Network isolation.** Security groups on the cluster ENIs are the enforcement point. Reference the application tier security group as the source rather than a CIDR block. There is no public endpoint option in a VPC deployment, but a cluster in a public subnet with a permissive security group and a route to an internet gateway is reachable, which is the classic exposure.

- **Encryption in transit.** TLS is optional and, for older engine versions, can only be enabled at cluster creation. Newer Valkey and Redis OSS versions support enabling it in place with a rolling migration through a transitioning mode that accepts both TLS and plaintext while clients cut over. Memcached supports TLS only on version 1.6.12 and later.

- **Encryption at rest.** Covers the persisted data on disk, backups, and swap. Uses an AWS managed key by default or a customer managed KMS key. Applies to Valkey, Redis OSS, and Memcached 1.6.12 and later, and for node-based clusters is set at creation and cannot be toggled afterward.

- **Redis AUTH and Valkey AUTH.** A shared-secret token presented on connect. It is a single credential for the whole cluster with no user concept, so it grants all-or-nothing access. It requires in-transit encryption to be enabled, and it should live in Secrets Manager with rotation. ElastiCache supports a two-token rotation flow so the old and new tokens are both valid during the cutover.

- **Role-Based Access Control.** The stronger data plane control on Valkey and Redis OSS 6 and later. You define users with passwords and an access string that limits which commands and which key patterns each user can touch, group them into a user group, and attach the group to the cluster. This is how you get least privilege inside the cache, for example a read-only user restricted to a key prefix, or a user with dangerous commands removed.

- **IAM authentication.** Valkey and Redis OSS 7 and later support authenticating a user with a short-lived IAM-generated token instead of a stored password, tying cache access to an IAM role and removing the long-lived secret entirely. The IAM user's name must match the ElastiCache user name, and the RBAC access string still governs what that user can do once connected.

- **Backups.** Manual and automatic snapshots write to an ElastiCache-managed S3 location and inherit the cluster's at-rest encryption. Snapshots can be exported to your own S3 bucket, which is a data-egress path worth policy-controlling, and can be copied across Regions with a destination-Region KMS key.

- **Logging.** There is no per-key access log. Valkey and Redis OSS emit slow logs and an engine log to CloudWatch Logs or Firehose, which capture slow and errored commands but not routine reads. CloudTrail records control plane changes such as cluster creation, security group modification, and parameter group changes. VPC Flow Logs are the only record of who connected. Memcached offers none of this.

- **Parameter groups.** Where you disable or rename dangerous commands such as `FLUSHALL`, `KEYS`, and `CONFIG` on Valkey and Redis OSS, and where cluster behavior including timeouts and eviction policy is set. Changes are tracked by Config and CloudTrail.

## ElastiCache engines and adjacent stores

| Option | Data plane authentication | Encryption in transit | Encryption at rest | Per-key or per-command authorization | Replication and failover | Access audit trail |
|---|---|---|---|---|---|---|
| ElastiCache for Valkey | RBAC users, IAM auth, or AUTH token | Yes, optional | Yes, KMS | Yes, RBAC access strings | Multi-AZ with automatic failover | Engine and slow logs, no read log |
| ElastiCache for Redis OSS | RBAC users, IAM auth on 7+, or AUTH token | Yes, optional | Yes, KMS | Yes, RBAC access strings | Multi-AZ with automatic failover | Engine and slow logs, no read log |
| ElastiCache for Memcached | SASL on 1.6.12+, otherwise none | Only on 1.6.12+ | Only on 1.6.12+ | No | None, nodes are independent | None |
| ElastiCache Serverless | RBAC and IAM auth | Always on | Always on | Yes, RBAC access strings | Managed across AZs | Engine and slow logs |
| MemoryDB | RBAC users, IAM auth | Always on | Always on, KMS | Yes, RBAC access strings | Multi-AZ, durable multi-AZ transaction log | Engine and slow logs |
| DynamoDB with DAX | IAM for DynamoDB, DAX cluster in VPC | Optional on DAX | Yes | IAM condition keys on the table | Multi-AZ | CloudTrail data events on the table |

## What gets tested

- **Redis or Valkey over Memcached whenever the question mentions encryption, authentication, replication, or compliance.** Memcached before 1.6.12 has no auth and no encryption of any kind, so it is disqualified from any sensitive-data scenario.

- **AUTH token versus RBAC versus IAM auth.** AUTH is one shared secret for the whole cluster. RBAC gives per-user command and key-pattern restrictions and is the answer when least privilege inside the cache is required. IAM authentication is the answer when the requirement is no long-lived credential stored anywhere.

- **In-transit encryption is a prerequisite for AUTH and RBAC.** A distractor will offer auth without TLS, which the service will not accept and which would put the token on the wire in cleartext.

- **Encryption settings on node-based clusters are largely creation-time.** Where in-place enablement is not supported the answer is create a new cluster with encryption enabled and migrate, which mirrors the Aurora and RDS pattern.

- **There is no read audit log.** If the requirement is knowing which client accessed the cache, the answer is VPC Flow Logs plus CloudTrail for configuration changes, not "enable ElastiCache logging." Anything asking for key-level access history has no clean answer, which itself is the point being tested.

- **Public exposure remediation** is moving the cluster to private subnets and scoping the security group to the application security group. NACLs are a supporting control, never the primary answer.

- **AUTH token rotation** uses the two-token flow: set the new token as an addition, roll clients, then delete the old one. A single hard swap is the wrong answer because it drops live connections.

- **Snapshot export to S3** is the exfiltration path to lock down with bucket policy and KMS key policy, and cross-Region snapshot copy needs a key in the destination Region.

- **MemoryDB over ElastiCache** when the requirement includes durability of the data itself, since MemoryDB persists to a multi-AZ transaction log and ElastiCache is a cache that can lose data on failure.

## Limitations

- No per-operation access logging. You cannot reconstruct which client read which key, which makes forensics after a cache compromise dependent on application-side logging.

- AUTH is a shared secret with no identity behind it. Every application sharing the cluster shares the same blast radius unless RBAC is configured.

- RBAC and IAM authentication are unavailable on Memcached entirely, and IAM authentication requires recent Valkey or Redis OSS versions.

- On node-based clusters, at-rest encryption and, on older engine versions, in-transit encryption are fixed at creation. Changing them means a new cluster and an application cutover.

- TLS adds handshake latency and CPU overhead, which matters for a store chosen specifically for sub-millisecond response, so client connection pooling becomes a correctness requirement rather than an optimization.

- Cross-VPC and on-premises access requires peering, Transit Gateway, or a VPN. There is no endpoint service equivalent to a gateway endpoint.

- The cache holds decrypted, application-readable data in memory. At-rest encryption protects snapshots and disk, not the live keyspace, so an authenticated attacker reads plaintext regardless of KMS configuration.

- Automatic failover can drop in-flight connections and, for Redis OSS and Valkey with asynchronous replication, can lose recent writes. Treat cached authorization decisions and revocation lists as best-effort unless backed by a durable store.

- Memcached has no replication at all, so node loss means total loss of that node's keyspace with no failover path.