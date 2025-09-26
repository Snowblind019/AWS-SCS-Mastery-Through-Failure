# AWS Backup Logs

---

## What Is It

**AWS Backup** is a fully managed backup service that centralizes and automates the backup of data across AWS services like EBS, RDS, DynamoDB, EFS, FSx, EC2, and more.

While AWS Backup handles **orchestration, scheduling, and retention**, the **logging and monitoring components** ensure **visibility, compliance, and alerting** across all backup operations.

**AWS Backup Logs** capture:

- Who initiated a backup
- What was backed up or restored
- Whether it succeeded or failed
- How it performed over time

This visibility is **critical for**:

- Forensics and incident response
- Compliance and audits (SOC2, HIPAA, PCI)
- Detecting unauthorized restores or deletions
- Verifying backup reliability and SLAs

---

## Cybersecurity and Real-World Analogy

### **Cybersecurity Analogy**

Imagine you're the **security lead in a hospital**. You must ensure medical records are backed up every night — and that *no unauthorized personnel can restore or delete them*.

AWS Backup Logs are like **security camera footage + access logs** for your server room:
- When were backups triggered?
- Who restored data?
- What failed and why?

Without this visibility, you’re **flying blind in a critical compliance zone**.

### **Real-World Analogy**

Think of AWS Backup Logs as **bank vault receipts**:

- Deposit (Backup)
- Withdrawal (Restore)
- Break-in attempt (Unauthorized action)
- Vault malfunction (Backup or restore failure)

They provide the **audit trail** that shows what happened, when, and by whom — *so nothing gets lost in the dark*.

---

## Where the Logs Come From

AWS Backup **does not generate logs directly** — instead, it integrates with:

| **Service**               | **Purpose**                                                                 |
|---------------------------|------------------------------------------------------------------------------|
| **AWS CloudTrail**        | Captures management events: API calls like `StartBackupJob`, `DeleteVault`  |
| **Amazon CloudWatch**     | Emits backup job metrics: `BackupJobStatus`, `RestoreJobStatus`             |
| **AWS Backup Audit Manager** | Evaluates compliance against frameworks (PCI, HIPAA, SOC2)              |

These services provide the **centralized observability** needed for security and operations teams.

---

## What Gets Logged?

| **Action Type**        | **Logged Via**            | **Examples**                                                                 |
|------------------------|---------------------------|------------------------------------------------------------------------------|
| Backup Job Start       | CloudTrail                | `StartBackupJob`, resource ID, timestamp, principal ARN                      |
| Backup Job Completion  | CloudTrail + CloudWatch   | Status (SUCCESS/FAILED), duration, logs                                      |
| Restore Job Start      | CloudTrail                | Who triggered restore and on what resource                                   |
| Restore Completion     | CloudWatch Metrics        | Duration, bytes restored, success/failure                                    |
| Vault Access           | CloudTrail                | Creation/deletion of backup vaults                                           |
| Backup Plan Changes    | CloudTrail                | Modifying backup schedules/lifecycle rules                                   |
| Backup Copy Activity   | CloudTrail + CW Logs      | Cross-account or cross-region copies                                         |
| Policy Evaluation      | Backup Audit Manager      | Whether backups meet defined compliance expectations                         |

## Example CloudTrail Event – Backup Job

```json
{
  "eventTime": "2025-09-23T15:42:12Z",
  "eventName": "StartBackupJob",
  "awsRegion": "us-west-2",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "Snowy"
  },
  "requestParameters": {
    "backupVaultName": "CriticalVault",
    "resourceArn": "arn:aws:ec2:us-west-2:123456789012:volume/vol-01abcde123456",
    "iamRoleArn": "arn:aws:iam::123456789012:role/AWSBackupRole"
  },
  "responseElements": {
    "backupJobId": "abcd-1234-backup-job"
  }
}
```

---

## How to Monitor Logs in Practice

### 1. **CloudTrail Integration**

- Enable CloudTrail (if not already enabled)
- Filter logs for:
  - `StartBackupJob`
  - `StartRestoreJob`
  - `DeleteRecoveryPoint`
  - `CreateBackupVault`
- Filter by:  
  `eventSource = backup.amazonaws.com`

### 2. **CloudWatch Alarms**

- Use CloudWatch Metrics:
  - `BackupJobStatus`
  - `RestoreJobStatus`
- Create Alarms for:
  - **FAILED** or **EXPIRED** states
  - Ex: Trigger alert if daily RDS backup fails

### 3. **Backup Audit Manager**

- Define compliance controls (e.g., “All RDS DBs must be backed up every 24h”)
- Enable **Audit Frameworks** (HIPAA, PCI, etc.)
- Export daily reports for auditors or internal compliance teams

---

## Use Cases for Security Teams

### Detect Gaps in Coverage
CloudTrail logs show resources that were created but never backed up.

### Investigate Restoration Events
Trace if a restore was triggered during a data breach or insider event.

### Prove Backup Integrity to Auditors
Backup Audit Manager provides formal reports showing coverage and conformance.

### Track Deletion Attempts
CloudTrail captures `DeleteRecoveryPoint` and `DeleteBackupVault` with user and timestamp.

---

## Pricing Model (Logging-Specific)

| **Service**            | **Logging Cost Detail**                                                         |
|------------------------|----------------------------------------------------------------------------------|
| **CloudTrail**         | Free for Management Events (most AWS Backup actions); Data events cost extra     |
| **CloudWatch Logs**    | Charged per **ingested GB** and **stored GB/month**                              |
| **Backup Audit Manager** | Charged **per evaluation** (based on number of resources and selected frameworks) |
| **AWS Backup**         | Backup jobs priced separately (by size, frequency, vault tier, region, etc.)     |

---

## Final Thoughts

Even the best backup architecture is **meaningless without observability**.

**AWS Backup Logs** — integrated via CloudTrail, CloudWatch, and Backup Audit Manager — give you:

- **Visibility**
- **Traceability**
- **Accountability**

They let you answer questions like:

- *Are our backups actually happening?*
- *Is someone restoring or deleting data without approval?*
- *Can I prove to an auditor that we’re compliant?*

In security, **logging your backups is just as important as performing them**.