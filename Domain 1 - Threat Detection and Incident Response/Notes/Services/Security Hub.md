# AWS Security Hub

## What Is The Service

**AWS Security Hub** is the *posture and findings hub* for your AWS estate. It **aggregates**, **normalizes**, and **prioritizes** security signals from AWS services (**GuardDuty**, **Inspector**, **Macie**, **IAM** Access Analyzer, **Config**, **Detective**, etc.) and many third-party tools, evaluates your accounts against **security standards** (e.g., *AWS Foundational Security Best Practices, CIS, PCI DSS, NIST 800-53*), and gives you one place to **triage**, **track**, and **automate** response.

**Why it matters**: without a hub, teams bounce between consoles, CSV exports, and ad-hoc scripts. With **Security Hub**, Snowy gets **one queue**, **one score**, and **one automation fabric** to turn *“we have alerts all over”* into:

> “Winterday’s org is 92% compliant; here are 14 high-priority, **deduped** findings routed to **Blizzard-OnCall** with playbooks attached.”

It’s not a **SIEM** replacement; it’s the **AWS-native control tower** for cloud security posture and findings lifecycle.

---

## Cybersecurity and Real-World Analogy

### **Security analogy.**
Imagine the **Winterday** SOC with different sensors: doors, cameras, badge readers, smoke detectors. **Security Hub** is the **dispatch desk** that:

- **Collects** all alarms in one pane,
- **Scores** the building against the safety checklist,
- **Flips** the right playbook for each alarm,
- **Tracks** which alarms are resolved and which controls are still failing.

### **Real-world analogy.**
Think of airline ops control: multiple aircraft (accounts), multiple telemetry feeds (services), and a **central ops board** showing fleet status, open issues, and compliance against flight rules.  
You don’t fix engines in that room—but **you decide what to fix first and who rolls with what checklist.**

---

## How It Works

### Core Model

- **Findings aggregation.**  
  Security Hub ingests findings from AWS services and partners. Everything is converted into the **AWS Security Finding Format (ASFF)** — a normalized JSON schema so you’re not reconciling twenty different dialects.

- **Standards & controls.**  
  You enable one or more **standards** (e.g., *AWS Foundational Security Best Practices v1.0/1.0.1, CIS AWS Foundations, PCI DSS, NIST 800-53*).  
  Each standard is a set of **controls** mapped to signals from services like Config, IAM, GuardDuty, etc.  
  Controls evaluate to: `PASSED / FAILED / WARNING / NOT_AVAILABLE / SUPPRESSED`

- **Security score.**  
  Per standard and overall:  
  `passed controls / enabled controls`.  
  It’s a **directional KPI**; don’t game it — use it to guide work.

- **Workflow states.**  
  Findings carry `Workflow.Status` (e.g., `NEW`, `NOTIFIED`, `RESOLVED`, `SUPPRESSED`) and `RecordState`; you can drive these via automation.

- **Automation hooks.**  
  `Custom actions`, `EventBridge rules`, `BatchUpdateFindings`, and `Insights` power routing and bulk operations.

## The Moving Pieces

| Piece               | What It Is                                      | Why Snowy Cares                                              |
|---------------------|--------------------------------------------------|---------------------------------------------------------------|
| **ASFF**            | Normalized JSON for all findings                | Same fields to filter/route across tools                      |
| **Standards & Controls** | Policy-as-checks mapped to services          | Turn best practices into scored, **trackable** items          |
| **Insights**        | Saved queries over findings                     | “Show only Highs in prod from GuardDuty this week”            |
| **Custom Actions**  | Buttons/labels that fire **EventBridge**        | One-click “Quarantine,” “Open Ticket,” “Assign to Blizzard”   |
| **Admin/Member**    | Org-wide aggregator model                       | Single pane across **Winterday** accounts/Regions             |
| **Finding Aggregator** | Multi-Region view                            | Centralize triage; keep data local for sovereignty if needed  |

## Data Flow

1. A service (e.g., **GuardDuty**) emits a finding → **Security Hub ingests + normalizes it (ASFF)**.
2. Security Hub **evaluates controls** (e.g., Config check fails → control FAILED).
3. Snowy triages via **Insights** (saved filters), **tags**, **standards view**, and **severity**.
4. Automation kicks in: a **Custom Action** or **EventBridge** rule calls **SSM Automation**, **Lambda**, or your SOAR to contain/fix.
5. `Workflow.Status` is updated to track progress; score moves as controls pass/fail.

## Where Security Hub Fits

| Layer               | Tooling                                      | Role of Security Hub                                         |
|---------------------|-----------------------------------------------|----------------------------------------------------------------|
| **Prevention & Config** | Organizations, SCPs, IAM, Config             | Consume control signals; show pass/fail                         |
| **Detection**       | GuardDuty, Inspector, Macie, Access Analyzer  | Aggregate findings; normalize & prioritize                     |
| **Investigation**   | Detective, CloudTrail Lake                    | Link out from a finding to investigate deeply                  |
| **Response**        | EventBridge, SSM, Lambda, SOAR                | Launch playbooks via custom actions/rules                      |
| **Governance**      | Standards, conformance packs                 | Track adherence and progress with a score                      |

## Integrations You’ll Actually Use

### First-Party Signals

| Service               | Examples of What Flows into Hub                                            |
|------------------------|-----------------------------------------------------------------------------|
| **GuardDuty**          | Threat findings (*IAM, EC2, DNS, S3, EKS, RDS*, malware protection)         |
| **Inspector**          | EC2/ECR/Lambda **vulnerability findings** with CVE metadata & fix paths     |
| **Macie**              | Sensitive data discovery + bucket exposure risks                            |
| **IAM Access Analyzer**| Public/cross-account access to resources (*S3/KMS/IAM/SQS/etc.*)            |
| **Config**             | Resource **misconfig** findings mapped to controls                          |
| **Detective**          | Linked investigations (you **pivot from Hub** to Detective)                 |

> **Third-party**: EDR, CSPM, WAF, CNAPP, DLP, ticketing.  
> Partners push into **ASFF**; you keep **one queue**.

---

## Pricing Model

Security Hub pricing is based on **ingested finding events** and **control evaluations**, with volume **tiering**.  
There’s **no extra charge** for **Insights**, **Custom Actions**, or **score**.

| Cost Driver            | What Changes the Bill                                   | How to Control It                                                  |
|------------------------|----------------------------------------------------------|---------------------------------------------------------------------|
| **Finding ingestion**  | Volume from GuardDuty/Inspector/Macie/partners           | Suppress known-benign patterns at source; tune partners; aggregate wisely |
| **Control evaluations**| Enabled controls × accounts × Regions                    | Enable where you operate; disable unused; scope dev selectively     |
| **Multi-Region agg.**  | Aggregator across Regions                                | Only aggregate needed Regions; preserve data locality               |

> **Rule of thumb**: Turn Hub on for **prod** and sensitive **Winterday** accounts/Regions first.  
> Confirm **signal-to-noise**, then expand.

---

## Operational & Security Best Practices (Snowy’s Checklist)

1. **Delegate an admin** (e.g., *Snowy-Security*) and **auto-enable** members via Organizations.  
2. **Pick standards intentionally.**  
   Start with *AWS Foundational Security Best Practices*; add *CIS* or *NIST* only if needed.  
3. **Create ownership-oriented Insights.**  
   - High severity, prod, service=Blizzard  
   - New findings in last 24h for Winterday-Data  
   - FAILED controls mapping to encryption/public exposure  
4. **Wire Custom Actions to playbooks.**  
   - `Quarantine-EC2`, `Lock-S3`, `Rotate-Keys`, `Open-Jira`  
   - **Backed by EventBridge → SSM/Lambda**  
5. **Normalize tags.**  
   Ensure resources carry: `Service`, `Team`, `Environment`, `DataOwner`, `Sensitivity`  
6. **Set workflow rules.**  
   - `High` → page Blizzard-OnCall  
   - `Medium` → 24h SLA ticket  
   - `Low` → weekly digest  
7. **Suppress with discipline.**  
   Suppressions must be **time-boxed** + justified. Revalidate monthly.  
8. **Close the loop.**  
   Fix the **control**, not just findings.  
9. **Dashboards that matter.**  
   MTTD/MTTR, recurring failure trends, posture drift.  
10. **Audit pack.**  
   Monthly: overall score, deltas, open Highs, SLAs, and exports.

## Hands-On: Insights & Automation (Copy/Paste Starters)

### Example Insights You’ll Click

- **High severity in prod (last 24h):**

```sql
SeverityLabel = HIGH AND ResourceTags.Environment = "prod" AND CreatedAt >= -24h
```

- **New external-exposure risks:**

```sql
Title CONTAINS "public" OR Title CONTAINS "cross-account" AND WorkflowStatus = "NEW"
```

- **Recurring vulnerability on Snowy services:**

```sql
ProductName = "Inspector" AND SeverityLabel IN (MEDIUM, HIGH) AND ResourceTags.Service STARTS_WITH "Snowy-" AND CreatedAt >= -7d
```

### Custom Actions → EventBridge → SSM

#### Action Name: `Blizzard-Quarantine-EC2`

```text
EventBridge rule: if UserDefinedFields.Action = "QuarantineEC2"
→ run SSM doc AWS-ApplyQuarantineSG with instanceId from finding.
```

#### Action Name: `Winterday-Lock-S3`

- Runbook sequence:
  - Enable **Block Public Access**
  - Apply restrictive bucket policy (only `Snowy-Exports-Role`)
  - Quarantine flagged objects
  - Notify **DataOwner=Blizzard-Analytics**

## Standards Quick-Map

| Standard                            | What It’s For                        | Good First Targets                                   |
|-------------------------------------|---------------------------------------|------------------------------------------------------|
| **AWS Foundational Security BP**    | AWS-specific best practices           | Encryption, logging, least-privilege, network hygiene |
| **CIS AWS Foundations (v1.4+)**     | Baseline hardening & logging          | CloudTrail on, MFA on root, centralized logging      |
| **PCI DSS**                         | Cardholder data environments          | Strict logging, segmentation, encryption             |
| **NIST 800-53**                     | Gov/regulated alignment               | Control families mapped into AWS checks              |

> **Pro tip**: Pair with **Config Conformance Packs** to track enforcement → Hub tracks **outcomes**.

## Comparisons

| Tool         | Best At                              | How It Pairs With Security Hub                              |
|--------------|---------------------------------------|--------------------------------------------------------------|
| **GuardDuty**| Threat detections                     | Hub aggregates & routes; GuardDuty stays the detector        |
| **Inspector**| Vulnerabilities (EC2/ECR/Lambda)      | Hub prioritizes & automates fix routing                      |
| **Macie**    | Sensitive data in S3                  | Hub centralizes alerts w/ posture                            |
| **Config**   | Resource compliance                   | Hub turns signals into scored posture                        |
| **Detective**| Investigation graph                   | Hub → “Investigate in Detective” to scope impact             |
| **SIEM**     | Long-term correlation                 | Export to SIEM; Hub = posture + **actioning**               |

---

## Real-Life Example (End-to-End, Winter Names)

### Scenario
A routine score drop: **Security Hub falls from 92% → 87% overnight**. Snowy opens the **Blizzard-Posture** dashboard.

### A. Triage

- **Insight**: 9 High findings for `public S3 access` and 3 **Macie** `sensitive-data` flags under `s3://snowy-exports/`

### B. Linkage

- Findings reveal cross-account wildcard policy + objects w/ phone numbers.

### C. Action (One Click)

```text
Custom Action: Winterday-Lock-S3
EventBridge → SSM:
  - Enable Block Public Access
  - Restrict policy (only Snowy-Exports-Role)
  - Quarantine to private bucket
  - Notify DataOwner = Blizzard-Analytics (with Macie excerpts)
```

### D. Follow-Ups

- 2nd action → **Jira ticket** w/ ASFF attached.

### E. Recovery

- Controls → `PASSED`, score back to 92%.
- Add `Config rule param` to deny public policy
- Create **Hub suppression rule** (time-boxed)

### F. Evidence

- Monthly audit export → *before/after state*, *runbook output*, *Macie resolution*.

---

### Routing Matrix (Severity × Environment)

| Severity | Prod                                             | Staging/Dev                     |
|----------|--------------------------------------------------|---------------------------------|
| High     | Page **Blizzard-OnCall** + auto-runbook          | Open ticket; auto-runbook if safe |
| Medium   | Ticket in 24h; assign to **Service Owner**       | Weekly triage board             |
| Low      | Digest report; fix in posture sprint             | Digest report                   |

### Useful ASFF Fields (For Filters/Automation)

| Field                        | Why It’s Useful                                     |
|-----------------------------|-----------------------------------------------------|
| `Severity.Label`, `Severity.Normalized` | Route by impact                         |
| `ProductName`, `Types[]`     | Source/type grouping (*GuardDuty/Macie/Inspector*) |
| `Resources[]` (ARNs & tags)  | Ownership, environment, service                    |
| `Workflow.Status`            | Drive lifecycle (`NEW` → `NOTIFIED` → `RESOLVED`) |
| `RecordState`                | Active vs archived                                 |
| `CreatedAt / UpdatedAt`      | Freshness windows                                  |
| `UserDefinedFields`          | Routing hints (team, **JIRA**, runbook id)         |

---

## Final Thoughts

**Security Hub** doesn’t replace your detectors or your **SIEM** — it **connects** them.  
It turns scattered AWS security signals and best-practice checks into **one prioritized queue**, **a measurable posture score**, and **push-button automation**.
