# AWS Snowball Edge

AWS Snowball Edge is a ruggedized physical device from the AWS Snow family, used to move large datasets into AWS by shipping them rather than transferring over a network, and to run compute at the edge in disconnected or bandwidth-constrained environments. It comes in storage-optimized and compute-optimized variants, the latter able to run EC2 instances and Lambda functions locally. Its security relevance is concentrated in two scenarios cloud-native services cannot address: moving data from a site where the network cannot carry it in a reasonable time, and running processing in a genuinely air-gapped or intermittently connected location such as a remote facility, a ship, or an incident site with no trustworthy uplink. The device is built to be handled outside a data center, so its security model addresses physical threats that S3 and EC2 never have to: tamper-evident and tamper-resistant hardware, a Trusted Platform Module, encryption where the device never holds the plaintext keys, and a NIST-compliant erase after ingestion. The thing to hold onto is that Snowball Edge extends AWS security controls to physical media in transit and to disconnected compute, so its distinctive concerns are chain of custody, tamper evidence, and key handling for a device that leaves your control.

## How it works

- **Ordering and job types.** Ordered from the console as an import job (data into AWS), an export job (data out of AWS), or a local compute and storage job. The job definition specifies the destination or source S3 buckets and the KMS key, and the device is configured before it ships.

- **Key handling.** Data is encrypted with keys managed through KMS, and the device does not store the encryption keys persistently. Keys are provided to unlock the device for a session and are not written to disk, so a stolen device is encrypted data with no keys on it. The unlock manifest and unlock code are delivered separately, which is a deliberate separation of the device from the means to open it.

- **Tamper protection and TPM.** The chassis is tamper-evident and tamper-resistant, and a Trusted Platform Module measures the device's integrity so tampering in transit is detectable. This is what makes the physical chain of custody verifiable rather than assumed.

- **Local access interfaces.** Data is loaded and retrieved over the Snowball client, an S3-compatible endpoint on the device, or an NFS interface. Compute-optimized devices additionally expose EC2-compatible instances and can run Lambda through IoT Greengrass, so processing happens on the device against local data.

- **Local compute security.** EC2 instances on the device run from AMIs you prepare and preload. They carry whatever the AMI carries, so hardening, patching, and the credentials baked into or supplied to them are your responsibility, exactly as with EC2 in the cloud, but without the cloud's surrounding controls during disconnected operation.

- **Return and ingestion.** The device ships back with a provided label. On receipt AWS verifies integrity, decrypts using the KMS key, loads the data into the designated bucket, and performs a NIST 800-88 compliant erase of the device before it is reused. That erase is the control that lets the same physical device serve another customer without data remanence.

- **Object Lock at the destination.** Data landing in S3 can go into a bucket with Object Lock in compliance mode, which is the pattern for forensic and evidentiary data: an offline-collected dataset lands in immutable storage with a verifiable path from collection to archive.

- **IAM and roles.** IAM governs who can create Snow jobs and which S3 buckets a job may read or write, through a role the Snow service assumes. On-device operations use device-local credentials issued for the session rather than your standing IAM identity.

- **Disconnected operation.** Compute-optimized devices operate with no connection to an AWS Region, which is the entire point for air-gapped use, but also means none of the Region's logging, detection, or key services are reachable during that window. What happens on the device is recorded locally and reconciled on return.

- **Logging.** CloudTrail records the control plane: job creation, configuration, and status. On-device activity is logged locally and is not visible in CloudTrail until, and only to the extent, it is brought back. The audit trail therefore spans the shipping and custody records, the local device logs, and the cloud-side ingestion record.

## Snowball Edge versus adjacent transfer and edge options

| Option | Transfer mechanism | Works fully offline | Local compute | Physical tamper protection | Best fit |
|---|---|---|---|---|---|
| Snowball Edge | Shipped physical device | Yes | EC2 and Lambda on device | Yes, TPM and tamper-evident | Large offline transfer, disconnected edge compute |
| AWS Snowcone | Smaller shipped device | Yes | Limited EC2 | Yes | Small or portable edge collection |
| AWS DataSync | Network transfer with an agent | No, needs connectivity | No | Not applicable | Online bulk transfer with verification |
| S3 Transfer Acceleration | Network via edge and backbone | No | No | Not applicable | Global online uploads to S3 |
| AWS Outposts | Racked AWS hardware on premises | Needs a link to a Region | Full AWS services | Data center physical security | Persistent on-prem AWS presence |
| Direct Connect | Dedicated network circuit | No | No | Not applicable | Sustained high-bandwidth connectivity |

## What gets tested

- **Snowball Edge is the answer when the network cannot carry the data in a reasonable time**, or when the site is genuinely disconnected. If connectivity exists and is adequate, DataSync or Transfer Acceleration is the answer, since shipping a device is slower and more operationally heavy when the network is viable.

- **Compute-optimized for edge processing in disconnected environments**, such as running detection, inference, or forensic analysis on site with no uplink. Storage-optimized when the requirement is only bulk data movement.

- **The device holds no persistent keys**, so a lost or stolen device in transit is encrypted data an attacker cannot open without the separately delivered unlock code and manifest. This is the answer to physical interception concerns.

- **NIST 800-88 erase after ingestion** is the data remanence answer, and the chain-of-custody logging plus tamper evidence is the answer for evidentiary and compliance transfers.

- **Landing offline-collected data in an Object Lock bucket** completes the chain of custody for forensic use, pairing the device's tamper evidence with immutable destination storage.

- **Disconnected operation means no Region-side controls during the window.** A question about detection or key access during air-gapped operation is testing whether you understand that GuardDuty, CloudTrail, and live KMS are not reachable, and that on-device activity reconciles only on return.

- **On-device EC2 is your responsibility to harden.** The AMI carries its own posture, and there is no Inspector, Systems Manager, or patch pipeline reaching a disconnected device, so the image must be hardened before it ships.

- **Snowcone versus Snowball** is a size and portability distinction. A small, truly portable collection in a space-constrained or rugged setting points at Snowcone.

## Limitations

- Transfer latency is measured in days, since it involves shipping. It is faster than a slow network for very large datasets but far slower for anything the network could carry, so it is the wrong tool whenever adequate connectivity exists.

- During disconnected operation the device is outside every Region-side control. No live CloudTrail, no GuardDuty, no live KMS, no Config, so security monitoring during the offline window is only what the device records locally.

- On-device compute inherits none of the cloud's surrounding controls. EC2 instances run from AMIs you must harden and patch in advance, and credentials supplied to them operate without the cloud's guardrails.

- The physical chain of custody is a human process. Tamper evidence and TPM measurement detect interference, but the operational discipline of shipping, receiving, and custody documentation is what makes the evidentiary guarantee real.

- Keys and unlock materials must be handled correctly out of band. The security of a device in transit depends on the manifest and unlock code being delivered and stored separately from the device, which is a process that can be undermined by careless handling.

- Capacity and compute are fixed at the hardware level. A job that outgrows the device's storage or compute requires additional devices, and there is no elastic scaling in the field.

- Return and ingestion introduce a delay before data reaches S3 and becomes queryable, so it is unsuitable for anything needing timely availability.

- The device is a shared physical asset returned to AWS and wiped, so any assumption that data persists on it after the job is wrong, and the erase is a feature that also means the device is not an archive.

- Cost is job-based with charges for extended possession, so a device held at a remote site for a long deployment accrues cost, and planning the possession window is part of using it economically.