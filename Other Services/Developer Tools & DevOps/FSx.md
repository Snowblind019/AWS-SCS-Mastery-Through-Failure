# Amazon FSx

Amazon FSx is a family of four managed file systems, each wrapping a different established file storage technology: FSx for Windows File Server for SMB and NTFS, FSx for Lustre for high-performance parallel workloads, FSx for NetApp ONTAP for multiprotocol enterprise storage, and FSx for OpenZFS for Linux NFS workloads. What makes FSx different from S3 in security terms is that it is not an API service. Every variant deploys as ENIs in your subnets, speaks a file protocol on a network port, and authorizes access using the file system's own model, NTFS ACLs against Active Directory, POSIX permissions, ZFS ACLs, or ONTAP export policies, rather than IAM. IAM governs whether you can create, modify, delete, or back up the file system; it does not govern whether a mounted client can read a file. That split is the whole security story: a compromised EC2 instance with network reachability to the file system inherits whatever the file protocol grants it, and no IAM policy will stop it. The thing to hold onto is that FSx is network-reachable storage with a non-IAM authorization model, so security groups plus the file system's own permission model do the work IAM does everywhere else.

## How it works

- **Deployment.** All variants are provisioned into a VPC with subnet placement and security groups on their ENIs. Single-AZ deployments use one subnet; Multi-AZ deployments use two with a floating endpoint. There is no public endpoint option for any variant, which removes an entire exposure class that S3 and RDS have.

- **FSx for Windows File Server.** SMB 2.0 through 3.1.1, joined to AWS Managed Microsoft AD or a self-managed Active Directory. Authorization is NTFS ACLs mapped to AD users and groups, plus SMB share permissions. Encryption in transit uses SMB encryption and can be enforced so unencrypted sessions are rejected. It supports file access auditing, emitting Windows access events for file and folder access attempts to CloudWatch Logs or Kinesis Data Firehose, which is the only variant with a native per-file audit trail. Shadow Copies provide user-recoverable previous versions.

- **FSx for Lustre.** The Lustre parallel filesystem, mounted with a client kernel module. Authorization is POSIX UID and GID permissions with no directory service integration, so identity is whatever the mounting host asserts. It links to S3 buckets for import and export, meaning data flows between an IAM-governed store and a POSIX-governed one, and the export writes back with the file system's service-linked role rather than the user's identity. Encryption in transit is automatic between supported instance types within the same VPC. Scratch file systems have no replication and lose data on server failure by design.

- **FSx for NetApp ONTAP.** Multiprotocol: NFS, SMB, and iSCSI against the same data. A storage virtual machine provides the tenancy boundary and can be joined to Active Directory for SMB. Authorization uses NFS export policies, NTFS ACLs, or unified ACLs depending on the protocol. It brings ONTAP-native features that matter for security: SnapLock for WORM retention, snapshots restorable without a full recovery, SnapMirror replication to another file system or to an on-premises ONTAP system, and FabricPool tiering of cold data to a capacity pool.

- **FSx for OpenZFS.** NFS with ZFS semantics. Authorization is POSIX permissions and NFS export configuration by client CIDR. Snapshots are instantaneous and space-efficient, and child volumes provide separation within a file system.

- **Encryption at rest.** All variants encrypt at rest unconditionally, with an AWS managed key by default or a customer managed KMS key selected at creation. The key cannot be changed afterward for an existing file system.

- **Encryption in transit.** SMB encryption for Windows and ONTAP SMB, Kerberos-based encryption for ONTAP NFS, automatic in-VPC encryption for Lustre between supported instance types, and NFS over TLS support varying by variant and client. This is the least uniform area across the family and the one most likely to be assumed rather than verified.

- **Backup.** Native FSx backups plus AWS Backup integration, with backups encrypted using the file system's key. AWS Backup adds vault policies and Vault Lock for immutable retention, which is the ransomware control. ONTAP adds SnapLock for WORM at the volume level.

- **Network controls.** Security groups on the file system ENIs are the reachability control, and they should reference the client security group rather than a CIDR. Required ports differ by variant: 445 and several AD-related ports for SMB, 2049 and related for NFS, and a set of Lustre-specific ports. On-premises access goes over Direct Connect or VPN, and cross-VPC access requires peering or Transit Gateway.

- **Logging.** CloudTrail covers the FSx control plane: file system creation, modification, deletion, backup operations, and tag changes. File-level access is invisible to CloudTrail on every variant. Windows File Server file access auditing is the one native per-file log. VPC Flow Logs show which clients connected but not what they did. ONTAP has its own auditing configurable through the ONTAP CLI and API.

## FSx variants and adjacent storage compared

| Option | Protocol | Identity and authorization | Native per-file access audit | WORM or immutability | Encryption in transit | Cross-Region replication |
|---|---|---|---|---|---|---|
| FSx for Windows File Server | SMB | Active Directory plus NTFS ACLs | Yes, file access auditing to CloudWatch or Firehose | Through AWS Backup Vault Lock | SMB encryption, enforceable | AWS Backup copy |
| FSx for Lustre | Lustre client | POSIX UID and GID, no directory service | No | Through AWS Backup Vault Lock | Automatic in-VPC for supported instances | S3 linkage plus backup copy |
| FSx for NetApp ONTAP | NFS, SMB, iSCSI | AD for SMB, export policies for NFS, SVM tenancy | ONTAP auditing, configured natively | SnapLock, plus AWS Backup Vault Lock | SMB encryption and Kerberos for NFS | SnapMirror |
| FSx for OpenZFS | NFS | POSIX permissions and export rules | No | Through AWS Backup Vault Lock | Varies by client and configuration | AWS Backup copy |
| Amazon EFS | NFS | POSIX plus IAM authorization for NFS and access points | No, but IAM authorization is logged in CloudTrail | Through AWS Backup Vault Lock | TLS via the EFS mount helper | EFS Replication |
| Amazon S3 | HTTPS API | IAM, bucket policies, access points | CloudTrail data events | Object Lock, compliance mode | Always TLS | S3 Replication |

## What gets tested

- **IAM does not authorize file access on FSx.** If a requirement is per-user access control on a share, the answer is Active Directory and NTFS ACLs for Windows or ONTAP SMB, and POSIX or export policies elsewhere. An IAM policy answer is wrong for the data plane on every variant.

- **EFS versus FSx for identity-driven access.** EFS supports IAM authorization for NFS clients and access points enforcing a POSIX identity, which FSx does not. A question requiring IAM-based file access control on NFS points at EFS.

- **Security groups are the only reachability control.** There is no public access setting to disable and no bucket policy equivalent. Restricting which instances can mount is a security group referencing the client security group.

- **Windows File Server file access auditing** is the answer when the requirement is recording who opened, modified, or deleted which file. No other variant has a native equivalent, and CloudTrail never sees file operations.

- **KMS key is set at creation and immutable.** Changing keys means creating a new file system and migrating, the same pattern as RDS and Aurora.

- **AWS Backup Vault Lock in compliance mode** is the ransomware answer, since it makes backups undeletable even by an administrator. Snapshots alone are insufficient if the attacker holds permissions to delete them.

- **Lustre scratch file systems are not durable.** Any scenario mentioning persistent or sensitive data on Lustre scratch is describing a design flaw, and the answer is a persistent deployment plus backups, or moving the data to S3 after the job.

- **The Lustre to S3 link crosses authorization models.** Data exported from Lustre to S3 is written by the file system's role, so the S3 objects carry no relationship to the POSIX user who created the file, which breaks attribution.

- **ONTAP SnapLock** is the answer when WORM retention is required at the file system level rather than through a backup vault, and SnapMirror is the answer for replication to an existing on-premises ONTAP estate.

- **AD compromise is FSx for Windows compromise.** A domain administrator credential grants access to every share, so the answer to limiting blast radius is AD tiering, separate AD instances per sensitivity level, and least privilege in group membership, not an AWS-layer control.

## Limitations

- No IAM-based data plane authorization on any variant. Access control lives in AD, POSIX, or ONTAP, which means it is managed outside AWS tooling and is invisible to IAM Access Analyzer, Config resource compliance, and most cloud posture management.

- File-level access is not in CloudTrail. Only FSx for Windows File Server produces a native per-file audit log, and it must be explicitly enabled.

- The KMS key is fixed at creation, so key rotation to a different CMK requires building a new file system and migrating the data.

- Encryption in transit is inconsistent across the family and across client configurations. Lustre encrypts automatically only between supported instance types in the same VPC, and NFS encryption depends on client support and configuration rather than being enforced by the service.

- Lustre has no directory service integration, so identity on the file system is whatever UID the mounting host presents. A compromised host can present any UID.

- Lustre scratch file systems have no replication and are explicitly designed to lose data on failure, which is a correct design for temporary compute scratch and a disaster for anything else.

- Multi-AZ is not available for every variant and configuration, and single-AZ deployments lose availability with the AZ.

- Cross-VPC, cross-account, and on-premises access all require network plumbing, peering, Transit Gateway, Direct Connect, or VPN, each of which widens the set of hosts that can reach the file system.

- ONTAP's depth is also its cost: it brings a second management plane with its own CLI, its own credentials, and its own audit configuration that AWS-native tooling does not see.

- File systems are stateful and long-lived. Unlike S3 where lifecycle policies expire objects, cleaning up sensitive data from a share is a filesystem operation someone has to remember to perform, and deleted files may persist in snapshots and previous versions.