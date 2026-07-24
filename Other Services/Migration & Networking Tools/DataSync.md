# AWS DataSync

AWS DataSync is a managed data transfer service for moving large datasets between on-premises storage and AWS, between AWS storage services, and across Regions and accounts. It handles the parts that make a hand-rolled `rsync` or scripted copy unreliable at scale: parallelized multi-threaded transfer, incremental sync based on metadata and checksums, in-transit encryption, per-file integrity verification, retry logic, and metadata and permission preservation. Its security relevance is that it is the sanctioned, auditable, encrypted path for exactly the transfers that most often go wrong, moving forensic evidence, migrating regulated data, or replicating logs to an immutable archive. Every task run is logged, every file can be validated, and the transfer can be kept entirely on the AWS private network through VPC endpoints so nothing touches the public internet. The two things that determine its security posture are the agent, a VM you run in your own environment that reads the source and therefore holds access to it, and the IAM roles DataSync assumes to read the source location and write the destination. The thing to hold onto is that DataSync is a governed, verifiable transfer channel whose trust concentrates in the agent that reads your source data and the location access roles that let it write to your destination.

## How it works

- **Agent.** For on-premises and other non-AWS sources, a DataSync agent runs as a VM on VMware, Hyper-V, KVM, or as an EC2 instance, reads the source NFS, SMB, HDFS, or object store, and streams to AWS. The agent holds credentials or mount access to the source, which makes the host it runs on a sensitive asset. Transfers purely between AWS services do not require an agent.

- **Locations.** A source and a destination, each a configured location: NFS, SMB, HDFS, or a self-managed object store on premises, and S3, EFS, FSx for Windows, FSx for Lustre, FSx for ONTAP, or FSx for OpenZFS in AWS. Each AWS location is accessed through an IAM role DataSync assumes, and an S3 location role needs read or write on the bucket plus KMS permissions if the bucket is encrypted.

- **Tasks.** A task pairs a source and destination location with options: what to transfer, what to verify, what to preserve, and how to handle files that no longer exist at the source. A task can run on demand or on a schedule, and it records the results of each execution.

- **Transfer modes and options.** Options control whether to transfer only changed data, whether to preserve POSIX permissions, ownership, and timestamps, whether to keep deleted files at the destination or remove them, and whether to overwrite. Metadata and permission preservation is what makes DataSync suitable for evidence and compliance transfers where the original attributes matter.

- **Integrity verification.** DataSync can verify data integrity during transfer and, optionally, perform a full checksum comparison of the entire source and destination after transfer. That post-transfer verification is the chain-of-custody control for forensic and compliance use, producing evidence that what arrived matches what left.

- **Filtering.** Include and exclude filters transfer only specific paths, file types, or patterns, which is both an efficiency control and a compliance control when only a subset of a share may leave its current location.

- **Encryption.** TLS in transit for every transfer. At rest, the destination's own encryption applies: SSE-S3 or SSE-KMS for S3, KMS for EFS and FSx. Writing to an SSE-KMS bucket requires the DataSync location role to have `kms:GenerateDataKey` and `kms:Decrypt` on the key, which is the common cause of a task failing partway through.

- **Network.** DataSync can route through VPC interface endpoints so agent-to-AWS and service-to-service traffic stays on the AWS private network with no public internet exposure. The agent activates against a specific endpoint type, public, VPC, or FIPS, and that choice fixes the network path for the agent's life.

- **Cross-account and cross-Region.** Tasks can move data between accounts and Regions, which is the pattern for centralizing backups or replicating to a DR Region. Cross-account S3 requires the destination bucket policy to trust the DataSync role.

- **IAM surface.** `datasync:*` actions govern creating locations, tasks, and agents and starting executions. The location roles are the higher-value element, since they carry the actual data access, and the agent activation ties a specific agent to your account.

- **Logging.** CloudTrail records control plane operations including task creation, execution start, and location changes. Task execution results and per-file transfer logs can be sent to CloudWatch Logs at configurable verbosity, which is where a failed or tampered transfer becomes visible, and the log level is what you raise when a compliance transfer needs a per-file record.

## DataSync versus adjacent transfer and migration options

| Option | Direction and sources | Agent required | Integrity verification | Metadata preservation | Scheduling | Best fit |
|---|---|---|---|---|---|---|
| AWS DataSync | On-prem and AWS storage, both ways, cross-Region and account | Only for non-AWS sources | Yes, optional full checksum | Yes, POSIX and SMB metadata | On demand or scheduled | Bulk file transfer and periodic sync |
| S3 Replication | S3 bucket to S3 bucket | No | Managed, not user-verified | Object metadata | Continuous | Ongoing S3-to-S3 replication |
| AWS Transfer Family | SFTP, FTPS, FTP into and out of S3 and EFS | No | Protocol dependent | Limited | Client-driven | Exposing a managed SFTP endpoint |
| Snowball | Physical device for offline transfer | The device | Yes | Yes | One-time | Petabyte-scale or low-bandwidth sites |
| AWS Backup | AWS service resources | No | Managed | Full, resource-level | Policy-driven | Backup and restore, not general transfer |
| Storage Gateway | On-prem access to AWS storage | The gateway VM | Managed | Yes | Continuous | Hybrid access, not bulk migration |

## What gets tested

- **DataSync is the answer for moving on-premises NAS or HDFS data to AWS with encryption, integrity checking, and logging.** A scripted `rsync` over VPN is the distractor, lacking verification, retry, and audit.

- **Post-transfer full checksum verification** is the chain-of-custody answer for forensic evidence and regulated data, producing proof that source and destination match.

- **Metadata and permission preservation** is the answer when POSIX ownership, ACLs, or timestamps must survive the transfer, such as moving a file share whose permissions carry meaning.

- **VPC endpoints keep the transfer off the public internet**, which is the answer for a compliance requirement that data never traverse the internet, and the agent's endpoint type is chosen at activation.

- **The location role plus KMS key policy** is why a task fails when the destination bucket is SSE-KMS encrypted. The role needs `kms:GenerateDataKey` and `kms:Decrypt`, and this is a recurring troubleshooting scenario.

- **DataSync to S3 with Object Lock** is the pattern for landing logs or evidence in immutable storage, combining the verified transfer with compliance-mode retention at the destination.

- **DataSync versus S3 Replication.** Replication is continuous and S3-to-S3 only. DataSync is scheduled or on-demand, spans on-premises and multiple storage services, and offers user-verifiable integrity. A nightly sync from S3 to FSx is DataSync, not replication.

- **The agent host is a sensitive asset.** It reads the source data, so it belongs in a controlled network segment such as a DMZ with restricted access, and compromising it means access to whatever the source location exposes.

- **Filtering limits what leaves.** When only part of a share may be transferred for compliance reasons, include and exclude filters are the control, not a separate copy of the permitted subset.

## Limitations

- Not real-time. DataSync moves data in scheduled or on-demand batches, so it is a synchronization and migration tool, not a streaming or continuous replication mechanism. Sub-minute freshness requirements need a different service.

- The agent is a standing piece of infrastructure with access to the source data. It must be patched, network-restricted, and monitored, and its compromise exposes the source it reads.

- Cross-account and SSE-KMS transfers depend on correctly aligned IAM roles, bucket policies, and key policies across the boundary, which is fiddly and fails partway through a large transfer when misconfigured.

- Integrity verification of a very large dataset adds significant time and read cost, since a full checksum comparison re-reads both source and destination, so the chain-of-custody guarantee has a real performance price.

- Cost is per gigabyte transferred, which for repeated full syncs of large datasets adds up, and outbound transfers from AWS incur data transfer charges on top.

- Transfer performance depends on the agent host resources and the source storage throughput, so a slow NAS or an undersized agent VM bottlenecks the job regardless of AWS-side capacity.

- The endpoint type is fixed at agent activation. Changing from a public to a VPC endpoint means reactivating the agent, which is disruptive to an established transfer schedule.

- It preserves source permissions but does not translate them across incompatible models cleanly. Moving SMB ACLs to an S3 object store or POSIX permissions to a Windows file system involves mapping that may not round-trip faithfully.

- DataSync moves data but does not classify it. Knowing that a share contains regulated data and should be filtered or encrypted at the destination is upstream work, and DataSync will faithfully transfer sensitive data it was pointed at regardless.