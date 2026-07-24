# Amazon EFS

Amazon EFS is a managed, elastic, POSIX-compliant NFS file system that many clients mount concurrently across Availability Zones in a Region. Unlike the FSx family, EFS is the one AWS file service that bridges the POSIX permission model and IAM: alongside standard UID, GID, and mode bits, it supports IAM authorization for NFS clients, file system policies that behave like bucket policies, and access points that force a specific POSIX identity and root directory on every client that uses them regardless of what the client asserts. That combination is what makes EFS defensible in a multi-tenant or container environment, because an access point removes the client's ability to choose its own identity, which is the fundamental weakness of NFS. The residual risk is the same as any shared file system: a mount target is an ENI on a network port, so anything with network reachability and a permitted identity can mount the whole file system, and there is no per-file access log to tell you what it read. The thing to hold onto is that access points plus IAM authorization are what turn EFS from trust-the-client NFS into something with an enforceable identity, and everything else, encryption, backups, network scope, follows normal AWS patterns.

## How it works

- **Mount targets.** One ENI per Availability Zone, each with its own security group and IP address. Clients mount over NFSv4.1 or 4.2 on port 2049. The security group on the mount target is the network reachability control, and it should reference the client security group rather than a CIDR range.

- **POSIX permissions.** Standard UID, GID, and mode bits enforced by the file system. Absent an access point, the mounting client asserts its own UID, so a root user on a compromised instance presents whatever identity it likes, which is the inherent NFS weakness.

- **Access points.** An application-specific entry point that enforces a POSIX user and group, and a root directory, on every operation through it. The client cannot override either. This is the mechanism for giving each container, task, or tenant an isolated subtree with a fixed identity, and it is what makes EFS usable for Lambda and Fargate workloads without trusting the workload.

- **IAM authorization for NFS clients.** The `elasticfilesystem:ClientMount`, `ClientWrite`, and `ClientRootAccess` actions can be required for mounting and writing, evaluated against the file system policy and the client's IAM role. Combined with a file system policy denying access without IAM authentication and without TLS, this produces an NFS file system where mounting requires an AWS identity.

- **File system policy.** A resource-based policy on the file system, evaluated like a bucket policy. The standard hardened policy denies all actions when `elasticfilesystem:AccessedViaMountTarget` is false, denies when `aws:SecureTransport` is false, and grants mount and write only to named roles, optionally scoped to a specific access point with the `elasticfilesystem:AccessPointArn` condition.

- **Encryption at rest.** Enabled at file system creation with an AWS managed key or a customer managed KMS key, and immutable afterward. Encryption of new file systems is on by default, and it can be enforced organization-wide with an SCP or IAM condition on `elasticfilesystem:Encrypted`.

- **Encryption in transit.** Not automatic. It requires the client to mount with TLS, which in practice means using `efs-utils` and the `-o tls` option, or specifying TLS in the ECS or EKS volume configuration. Enforcement is a file system policy denying access when `aws:SecureTransport` is false, since the service will otherwise accept plaintext NFS.

- **Storage classes and lifecycle.** Standard and One Zone, each with an Infrequent Access and an Archive tier. Lifecycle management moves files between tiers by last access time, and Intelligent-Tiering moves them back on access. One Zone stores data in a single Availability Zone and is therefore not durable against AZ loss.

- **Replication.** EFS Replication maintains a read-only replica in another Region or another AZ, with a recovery point objective typically under fifteen minutes. Failover promotes the replica by deleting the replication configuration, which makes it writable.

- **Backup.** AWS Backup integration with vault policies and Vault Lock for immutable retention. EFS has no snapshot mechanism of its own, so AWS Backup is the only recovery path, and Vault Lock in compliance mode is the ransomware control.

- **Logging.** CloudTrail records control plane operations including file system, mount target, and access point creation and deletion, policy changes, and, when IAM authorization is in use, the `ClientMount` authorization decisions. There is no per-file access log. VPC Flow Logs show which clients connected to a mount target. Deeper visibility requires OS-level auditing such as `auditd` on the client, which is the client's responsibility rather than the file system's.

## EFS versus adjacent shared storage

| Option | Protocol | Identity model | Enforced identity independent of client | Per-file access audit | Multi-AZ durability | Encryption in transit |
|---|---|---|---|---|---|---|
| Amazon EFS | NFS 4.1 and 4.2 | POSIX plus optional IAM authorization | Yes, through access points | No | Yes, Standard class | Optional, enforceable by policy |
| Amazon EFS One Zone | NFS 4.1 and 4.2 | Same | Same | No | No, single AZ | Same |
| FSx for Windows File Server | SMB | Active Directory plus NTFS ACLs | Yes, AD authenticated | Yes, file access auditing | Multi-AZ option | SMB encryption, enforceable |
| FSx for OpenZFS | NFS | POSIX plus export rules | No | No | Multi-AZ option | Varies by client |
| FSx for Lustre | Lustre | POSIX only, no directory service | No | No | Persistent deployments only | Automatic in-VPC on supported instances |
| Amazon S3 | HTTPS API | IAM, bucket policies, access points | Yes, IAM always | CloudTrail data events | Yes | Always TLS |
| Amazon EBS | Block, single attach or multi-attach | None, OS filesystem | No | No | Single AZ, snapshots cross-AZ | Volume encryption, no protocol layer |

## What gets tested

- **Access points are the multi-tenant isolation answer.** When containers, Lambda functions, or tenants share one file system and each must be confined to its own directory with a fixed identity, the answer is an access point per tenant with an enforced POSIX user and root directory, not directory permissions alone.

- **IAM authorization plus a file system policy** is the answer when the requirement is that only specific roles may mount, since POSIX permissions alone cannot express that. Scope grants with the `elasticfilesystem:AccessPointArn` condition so a role can only reach its own access point.

- **Encryption in transit is not on by default.** Enforcing it is a file system policy `Deny` with a `aws:SecureTransport` false condition, plus clients mounting through `efs-utils` with TLS. A security group answer is wrong.

- **EFS over FSx when IAM-based access control on NFS is required.** No FSx variant offers IAM authorization for the data plane, so this is the distinguishing feature.

- **The KMS key is set at creation and immutable**, which follows the same pattern as RDS, Aurora, and FSx. Changing keys means a new file system and a data migration.

- **AWS Backup with Vault Lock in compliance mode** is the answer for protecting file data against an administrator or ransomware, since EFS has no native snapshot and file deletion is immediate.

- **No per-file access log.** If a requirement is knowing which principal read which file, EFS cannot provide it, and the answer is either S3 with CloudTrail data events or client-side auditing. This is a genuine capability gap worth recognizing rather than working around.

- **One Zone is not multi-AZ durable**, so any scenario involving regulated data, audit artifacts, or anything not reproducible excludes it regardless of cost framing.

- **Lambda and EFS** requires the function to be VPC-attached, an access point, and `elasticfilesystem:ClientMount` in the execution role. That combination is a recurring architecture question.

- **`elasticfilesystem:ClientRootAccess`** is the permission allowing a client to operate as root on the file system, and granting it broadly defeats access point identity enforcement, so it is the permission to withhold.

## Limitations

- No per-file or per-operation access logging. CloudTrail sees the mount authorization at best, never the reads and writes, so post-incident answers about what was accessed depend on client-side auditing that must have been configured in advance.

- Without access points, NFS trusts the client's asserted UID and GID. A root user on any instance that can mount presents any identity it chooses, so POSIX permissions alone are not a security boundary against a compromised host.

- Encryption in transit is opt-in per mount. A client that mounts without TLS succeeds unless a file system policy explicitly denies it, and there is no service-level default enforcement.

- The KMS key cannot be changed after creation, and there is no snapshot-and-restore path as convenient as RDS, so re-keying means creating a new file system and copying data.

- Mount targets are per Availability Zone with their own security groups, so a security group change applied to one mount target and not the others produces inconsistent reachability that is easy to miss.

- Cross-VPC, cross-account, and on-premises access requires peering, Transit Gateway, Direct Connect, or VPN, each of which enlarges the set of hosts able to reach port 2049.

- Performance is elastic but not unlimited. Bursting throughput depletes on sustained load, and the resulting slowdown looks like an application problem rather than a storage limit, which complicates incident triage.

- Deleted files are gone immediately with no versioning and no recycle bin. Recovery depends entirely on AWS Backup having a recovery point, and the backup vault becomes the only line of defense against both accident and ransomware.

- One Zone loses data with its Availability Zone, and lifecycle transitions to Archive add retrieval latency that can surprise a workload expecting local-disk-like behavior.