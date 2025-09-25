# Data Retention & Lifecycle Management of Logs and Security Findings  

---

## Why This Matters

In security, time is rarely on your side.  
Attackers don’t always trigger alarms right away.  
Sometimes, you don’t realize there’s been a breach until weeks or months later.  
If your logs have expired, or your findings were deleted after 30 days, there’s no trail left to follow.  

- No “what happened.”  
- No “who did it.”  
- No timeline, no impact analysis, no root cause.  
- You’re flying blind.

Detection and investigation rely on historical data.  
If you delete too early, you lose the ability to trace.  
If you keep everything forever, you drown in cost and clutter.

**The goal is balance:**  
Long enough to retain forensic value.  
Smart enough to avoid waste.

---

## Key Concepts

### 1. Retention Policies

Defines how long something is kept before it’s:
- Archived (moved to colder storage)
- Deleted
- Transitioned automatically

**Apply to:**
- CloudTrail
- VPC Flow Logs
- S3 Access Logs
- Lambda Logs
- Security Findings (GuardDuty, Inspector, Macie, etc.)
- Snapshots & Artifacts (disk images, memory dumps)

Think of this as your **black box flight recorder** window.

### 2. Data Classification

Not all logs are created equal.

**Critical security logs:**
- API calls  
- Authentication events  
- Network activity  
→ **Should be kept longer** (1–7 years)

**Low-signal noise:**
- Debug logs  
- Verbose app traces  
→ **Can be downsampled or expired faster**

**Examples:**

| Log Type           | Recommended Retention    |
|--------------------|--------------------------|
| CloudTrail         | 1–3 years (minimum 90d)  |
| VPC Flow Logs      | 90d–1 year               |
| GuardDuty Findings | Default 90d — *export!*  |
| Macie Findings     | 30–90d                   |
| App Logs           | Depends on system risk   |

---

## Best Practices for Lifecycle Management

### A. Store Logs in Immutable, Centralized Locations

Send logs to **S3 buckets** with:
- Object Locking (WORM)
- Versioning enabled
- KMS encryption
- Deletion control via bucket policy

**Forward to a SIEM** (e.g., Splunk, Elastic, Datadog)  
Apply **access logging** on top of logs themselves

> This makes logs tamper-proof and audit-friendly.

### B. Apply Lifecycle Rules (Example: S3)

In a centralized S3 logging bucket:

- **Day 0–30:** Store in S3 Standard  
- **Day 31–365:** Transition to S3 Glacier  
- **After 1 year:** Move to Deep Archive  
- **After 7 years:** Delete (unless regulated)

> This keeps costs low while retaining detection capabilities.

### C. Keep API & Auth Logs Longer

**CloudTrail** is your #1 source of forensic truth.  
Shows *who did what, when, from where*

- **Never** set less than 90 days retention  
- **Ideal:** 1–7 years depending on industry

Enable:
- Log file integrity validation
- CloudTrail to S3 + Glacier
- Access control + SCPs to prevent deletion

### D. Export Security Findings Proactively

Findings from **GuardDuty, Macie, Security Hub, Inspector** often have short retention (30–90 days by default).

**Set up automatic exports:**
- S3 (structured JSON or CSV)
- SIEM / Security Data Lakes
- Ticketing systems (for audit trails)

Use **EventBridge + Lambda** to automate exports when new findings are generated.  
> Don’t rely on the default retention — or you’ll lose valuable evidence.

### E. Align Retention with Incident Timelines

Ask yourself:

- If detection is T+0  
→ Investigation starts at T+5  
→ RCA at T+14  
→ Report due at T+30  
→ Retrospective hunting at T+60...

Then what good is a **7-day log retention**?

> Your logs should outlive your incident lifecycle.  
> **Minimum recommendation:** 180 days for all logs, 1+ year for security/audit logs.

### F. Regulatory and Legal Requirements

Don’t just think security — think **compliance**:

| Standard   | Requirement                    |
|------------|--------------------------------|
| PCI DSS    | 1 year (with 3 months online)  |
| HIPAA      | 6 years (access logs)          |
| SOX        | 7 years                        |
| ISO 27001  | Org-defined, must be documented |

Map your retention policy to each framework by data type.

### G. Versioning + Integrity = Trust

- Turn on **S3 Object Versioning** to prevent silent overwrites  
- Enable **Object Lock** (governance or compliance mode)  
- Use **CloudTrail log validation** for anti-tampering  
- Hash logs when exporting (`sha256sum > audit.hash`)

> **Trustworthy logs = legally defensible logs.**  
> And **defensible logs = real security posture.**

---

## Lifecycle Policy Example: CloudTrail Logs

- **Bucket:** `org-cloudtrail-logs`  
- **Versioning:** Enabled  
- **Object Lock:** Compliance mode  
- **Lifecycle Policy:**  
  - Days 0–90: S3 Standard  
  - Days 91–365: S3 Glacier  
  - Days 366–2555 (7 years): Glacier Deep Archive  
- **Log Validation:** Enabled  
- **Access Control:** IAM policy + SCPs preventing delete or overwrite

---

## Final Thoughts

Logs are your **time machine**.  
Findings are your **breadcrumbs**.  

If you delete them too early, you lose the ability to:
- Detect advanced threats  
- Investigate root cause  
- Prove what happened  
- Respond with confidence  
- Comply with laws  

**Retention is not optional.**  
It’s a core part of cloud security architecture.

> Get it wrong, and your black box is just a black hole.  
> If you want to see, prove, and learn — keep your logs around long enough to do so.
