# Amazon EBS

Amazon EBS provides block storage volumes attached to EC2 instances, presented to the operating system as raw disks. Its security character comes from three properties that make it unlike any other AWS storage service. It persists independently of the instance, so a terminated workload can leave its disk behind. It is trivially cloneable through snapshots, which are incremental, cheap, restorable in any Region, and shareable with other AWS accounts, which makes snapshot permissions a direct data exfiltration path that involves no network traffic and touches no security group. And its encryption is a creation-time property with no in-place conversion, which means the answer to almost every EBS encryption question is snapshot, copy with the target key, restore. The service also carries the primary forensic workflow in AWS incident response: isolate the instance, snapshot the volume, restore a copy in a sandbox account, and analyze the copy while the original stays untouched. The thing to hold onto is that the volume is only half the asset, since the snapshot is a portable, shareable, restorable copy of the same data governed by an entirely separate set of permissions.

## How it works

- **Volume scope and attachment.** A volume lives in one Availability Zone and attaches only to instances in that same AZ. Moving a volume across AZs or Regions means snapshotting and restoring. Multi-Attach on io1 and io2 allows one volume to attach to multiple Nitro instances simultaneously, which requires a cluster-aware filesystem and is a data integrity hazard otherwise.

- **Volume types.** gp3 and gp2 for general purpose SSD, io1 and io2 for provisioned IOPS with io2 Block Express supporting the largest sizes and highest durability, st1 and sc1 for throughput-optimized and cold HDD. Type affects performance and cost, not the security model.

- **Encryption at rest.** Enabled at volume creation using an AWS managed key `aws/ebs` or a customer managed KMS key. It covers data at rest on the volume, data in transit between the volume and the instance, all snapshots of the volume, and all volumes restored from those snapshots. Encryption cannot be enabled or disabled on an existing volume, and the key cannot be changed in place. The path is snapshot, copy the snapshot specifying the new key, create a volume from the copy.

- **Encryption by default.** An account and Region setting that forces every new volume and every snapshot copy to be encrypted, with a configurable default KMS key. This is the single most effective EBS control and it is off unless enabled, which makes enabling it across every account and Region a standard baseline task.

- **Snapshots.** Incremental point-in-time copies stored in S3-backed storage managed by AWS, not in your buckets. Only changed blocks are stored, but a restore always produces a complete volume. Snapshots can be copied within a Region, across Regions, and across accounts, and copying is where the key change happens.

- **Snapshot sharing.** `ModifySnapshotAttribute` grants another account permission to use a snapshot, or makes it public. An unencrypted snapshot can be made public, which is the classic catastrophic exposure. An encrypted snapshot cannot be made public at all, which is itself a control, and sharing an encrypted snapshot with a specific account additionally requires granting that account use of the customer managed key. Snapshots encrypted with the AWS managed `aws/ebs` key cannot be shared with another account.

- **Data Lifecycle Manager.** Policy-driven snapshot and AMI creation, retention, cross-Region copy, and cross-account sharing, targeted by tags. It also supports snapshot archive tiering and can enforce retention rules, which is how snapshot sprawl is controlled without a Lambda.

- **Snapshot Lock.** Snapshots can be locked in governance or compliance mode for a specified duration, preventing deletion during the lock period. Compliance mode locks cannot be shortened or removed after a cooling-off period, which makes it the WORM control for EBS backups and the ransomware protection for AMI and snapshot estates.

- **Recycle Bin.** Retention rules that hold deleted snapshots and AMIs for a defined period so they can be restored, which protects against accidental and malicious deletion.

- **EBS direct APIs.** `ListSnapshotBlocks`, `ListChangedBlocks`, and `GetSnapshotBlock` read snapshot data directly without creating a volume or attaching an instance. This is how modern backup and forensic tooling reads snapshots efficiently, and it is also a data access path that bypasses the usual "restore a volume and mount it" pattern, so `ebs:GetSnapshotBlock` deserves its own IAM attention.

- **AWS Backup integration.** Centralized backup plans, cross-account and cross-Region copy, and Vault Lock for immutable retention, which is the alternative to Snapshot Lock when backups span multiple services.

- **Logging.** CloudTrail records `CreateVolume`, `AttachVolume`, `DetachVolume`, `CreateSnapshot`, `CopySnapshot`, `ModifySnapshotAttribute`, `DeleteSnapshot`, and the EBS direct API calls. With a customer managed key, the corresponding KMS `CreateGrant`, `GenerateDataKeyWithoutPlaintext`, and `Decrypt` calls also appear, which is a second signal that a volume or snapshot was accessed.

## EBS versus adjacent storage options

| Option | Attachment model | Encryption change after creation | Portable copy mechanism | Cross-account sharing | Immutability control | Access audit |
|---|---|---|---|---|---|---|
| Amazon EBS | One AZ, one instance, or Multi-Attach on io1 and io2 | No, snapshot and copy required | Snapshots, incremental | Snapshot sharing plus KMS key grant | Snapshot Lock, Recycle Bin, Backup Vault Lock | CloudTrail on API, no per-block read log except direct APIs |
| EC2 instance store | Physically attached, ephemeral | Always encrypted on Nitro, no key control | None, data is lost on stop | None | Not applicable | None |
| Amazon EFS | Many instances, multi-AZ | No, immutable at creation | AWS Backup only | No native sharing | Backup Vault Lock | CloudTrail on mount authorization only |
| Amazon FSx | Many clients, protocol dependent | No, immutable at creation | Backups, ONTAP SnapMirror | Limited | SnapLock on ONTAP, Backup Vault Lock | Windows File Server only |
| Amazon S3 | API, no attachment | Yes, changeable per object | Replication, copy | Bucket policy, access points | Object Lock, compliance mode | CloudTrail data events, per object |

## What gets tested

- **Encryption cannot be added to an existing volume.** The answer is always snapshot, copy the snapshot with the KMS key, create a new volume, and swap it in. This mirrors RDS, Aurora, EFS, and FSx and is the most repeated pattern in the exam.

- **Encryption by default is an account and Region setting.** Enforcing it estate-wide combines that setting with an SCP denying `ec2:CreateVolume` and `ec2:CopySnapshot` when `ec2:Encrypted` is false, plus a Config rule for detection.

- **Public snapshots are the exposure to hunt.** The remediation is `ModifySnapshotAttribute` removing the `all` group. The preventive controls are encryption by default, since encrypted snapshots cannot be made public, plus an SCP denying `ec2:ModifySnapshotAttribute` where `ec2:Add/group` is `all`.

- **Cross-account snapshot sharing needs both the snapshot permission and the KMS key grant.** A snapshot encrypted with `aws/ebs` cannot be shared at all, and the fix is copying it to a customer managed key first.

- **Forensic workflow order matters.** Isolate the instance with a restrictive security group, capture memory before stopping if volatile evidence is needed, snapshot the volume, copy the snapshot to a forensics account, restore in an isolated VPC, and analyze the copy. Working on the original volume destroys evidence.

- **Snapshot Lock in compliance mode** is the answer when snapshots must survive a compromised administrator, alongside Backup Vault Lock for a multi-service estate. Recycle Bin is the answer for accidental deletion, not for a determined attacker.

- **`ebs:GetSnapshotBlock` and the direct APIs** are a read path that does not require creating a volume, so restricting `ec2:CreateVolume` alone does not prevent snapshot data access.

- **`kms:ViaService` scoping** limits a role's use of a key to EBS operations specifically, which is the standard hardening when a role needs decrypt only for volume attachment.

- **Detached volumes and orphaned snapshots** are both cost and data-retention findings. A volume detached from a terminated instance still holds whatever was on it, encrypted or not, and is invisible to most workload-oriented scanning.

- **`DeleteOnTermination`** on the root volume defaults to true and on attached volumes defaults to false, which is why data survives instance termination unexpectedly.

## Limitations

- Encryption is a creation-time property. There is no in-place enable, disable, or re-key, so every encryption change is a snapshot, copy, restore, and cutover.

- Snapshots are a copy of the data governed by an entirely separate permission set. Locking down the volume does nothing about a snapshot already shared with another account, and revoking sharing does not recall a copy that account already made.

- A volume is bound to one Availability Zone and, except for Multi-Attach, one instance. Moving data means snapshot and restore, which produces additional copies to track.

- There is no per-read or per-write audit of volume contents. Once a volume is attached, everything the instance does with it is invisible to AWS logging, so file-level activity depends on OS auditing.

- Multi-Attach shares a block device between instances with no coordination. Without a cluster-aware filesystem the result is silent corruption, and it is not a substitute for EFS.

- Encrypted snapshot sharing depends on KMS key policy across accounts, which is a frequent source of failed restores during a real incident when the forensics account cannot decrypt the snapshot it was given.

- Snapshot storage is incremental and deduplicated, which makes deletion semantics unintuitive: deleting a snapshot does not necessarily reclaim its apparent size, and retention policies are easier to get wrong than they look.

- Instance store volumes attached to the same instance are not EBS, are not snapshotted, and are lost on stop, which surprises teams assuming all attached storage is durable.

- Restoring a large snapshot produces a volume whose blocks are lazily loaded from S3, so first-touch performance is degraded until the volume is fully hydrated or initialized, which matters during a time-sensitive recovery.