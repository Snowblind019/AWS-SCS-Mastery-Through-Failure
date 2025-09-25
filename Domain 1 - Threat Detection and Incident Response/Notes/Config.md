# AWS Config

## What Is The Service
AWS Config is your source of truth for who changed what, when, and to what across your AWS resources—plus whether those resources currently comply with the rules you care about (security baselines, governance, tagging, encryption, least-privilege, drift control, etc.). It continuously records configuration state, builds a time-ordered history (point-in-time snapshots + diffs), and evaluates that state against managed or custom rules. When something drifts, Config can flag it, open a finding, and even auto-remediate using Systems Manager Automation.

Why this matters: production doesn’t only fail because of bugs; it fails because of misconfigurations. An S3 bucket goes public, a security group opens 0.0.0.0/0, an IAM policy grows wildcards, a KMS key gets disabled, or a CloudTrail trail stops logging. Config is the paper trail + compliance engine that turns all of that into evidence and action—so “it slipped through” becomes “we detected it in 2 minutes, rolled it back, and have proof.”

---

## Cybersecurity And Real-World Analogy

**Security Analogy.** Think of a well-run SOC facility:

- Cameras record state over time (that’s Config’s configuration history and timelines).
- Guards run checklists each hour (that’s Config rules evaluating compliance).
- If a door is propped open, the alarm triggers and a playbook runs (that’s auto-remediation via SSM Automation).
- Auditors ask, “show me evidence this door was never unlocked last quarter” (that’s point-in-time snapshots and Advanced Queries).

**Real-world Analogy.** Your car’s black box + inspection checklist:

- The black box captures what changed and when (speed, steering inputs → for us: resource properties).
- The inspection checklist says what “good” looks like (tire pressure, brake pads → for us: encryption on, tags present, least-privilege).
- If a check fails, you fix it, log it, and sign off.

---

## How It Works

### Core Components

| Component                  | What it is                                                                 | Why it matters                                                                                 |
|---------------------------|-----------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Configuration Recorder    | The on/off switch for recording resource changes.                          | Without it, no history. You can include/exclude resource types.                                 |
| Delivery Channel          | Where histories/snapshots go (usually S3 + optional SNS notifications).    | S3 = audit evidence; SNS = trigger downstream workflows.                                        |
| Configuration Items (CIs) | The discrete “events” for changes (new resource, property updated, relationship changed). | You pay per CI; this is your timeline.                                                          |
| Rules (Managed / Custom)  | Evaluations of resource state: triggered on change or periodic.            | Defines “compliance.” Output is COMPLIANT / NON_COMPLIANT with annotations.                     |
| Conformance Packs         | Bundles of rules + docs as a versioned baseline (CIS/NIST/Foundational), deployable across accounts/regions. | Make governance repeatable and auditable.                                                       |
| Aggregators               | Cross-account/region rollups (often Organization-wide).                    | Single pane for Snowy-Org instead of 50 consoles.                                               |
| Remediation               | SSM Automation documents invoked when a rule fails.                        | Turns drift detection into drift correction.                                                    |
| Advanced Query            | SQL-like queries across current recorded state (and relationships).        | Fast answers to governance questions (“show untagged EBS volumes”).                             |

### Recording And History

- **What gets recorded.** You select resource types (or “record all supported”). Config then logs create/update/delete events and relationships (e.g., “this ENI is attached to this instance,” “this role is bound to this policy”).
- **Granularity.** Config stores full configuration snapshots and diffs—you can open a timeline for any supported resource and see exactly when a property changed.
- **Sourcing.** Config is independent of CloudTrail but complementary: CloudTrail answers “who called which API from where?”; Config answers “what does the resource look like now and how did it evolve?”
- **Retention strategy.** Keep Config data in S3 with lifecycle policies (e.g., 90 days hot, then archive to Glacier). That gives you long-horizon evidence without runaway cost.

### Rules

- **Managed rules.** Prebuilt checks such as s3-bucket-public-read-prohibited, restricted-ssh, cloudtrail-enabled, iam-policy-no-statements-with-admin-access, rds-storage-encrypted, ebs-encrypted-volume, ec2-instance-no-public-ip, etc.
- **Custom rules.** A Lambda function evaluates a resource or set of resources; return COMPLIANT / NON_COMPLIANT with a reason. Trigger on change (configuration change notifications) or periodically (cron).
- **Parameters.** Many managed rules accept parameters (e.g., allowed port lists, required tag keys).
- **Noise control.** Scope rules with resource filters, tags, or resource IDs. For periodic rules, pick intervals that fit your RTO/RPO and audit needs without over-evaluating.

### Conformance Packs

- Package multiple rules + remediation docs into one named, versioned configuration (YAML).
- Deploy pack “Winterday-CIS-Baseline” across all accounts/regions with Organization integration.
- Track overall compliance of the pack, not just individual rules, and export to S3 for auditors.

### Remediation

- Tie a rule’s NON_COMPLIANT result to an SSM Automation document—either AWS-provided (e.g., disable S3 public ACL) or custom (your playbook).
- Use automatic or admin-approved remediation modes.
- Pass rule parameters into the runbook (e.g., required tag keys, target KMS key IDs).

**Design note.** Keep remediation idempotent and safe. For risky operations, require a change ticket or approved mode.

### Relationships And Drift

- Config tracks resource relationships (ENI ↔ EC2, IAM Role ↔ Policy, ALB ↔ Target Group).
- You can visualize lineage for a resource and see drift after a deployment (e.g., “Blizzard-Checkout ALB now points at the wrong target group”).
- Combine with CloudFormation drift detection to separate “expected” changes (stack updates) from “out-of-band” changes (manual edits).

### Multi-account / Region

- Use Aggregators to centralize into Snowy-Observability (or a “Security Tooling” account).
- With AWS Organizations, enable All Features; delegate Config to a central account; deploy recorders, rules, and conformance packs Organization-wide.
- Apply guardrails (e.g., via SCPs) to ensure recorders stay on and delivery channels remain intact.

---

## Pricing Models

You primarily pay for Configuration Items (CIs) recorded and for rule evaluations. Conformance packs incur a per-account/region monthly charge. You’ll also pay for S3 storage (history/snapshots), SNS if used, and SSM Automation/downstream actions. Keep this high-level model in mind:

| Cost Area               | What drives cost                                             | Practical guidance                                                                 |
|------------------------|--------------------------------------------------------------|------------------------------------------------------------------------------------|
| Configuration recording| Number of CIs (changes captured)                             | Record only needed resource types in dev; “all supported” in prod/security-critical. |
| Rule evaluations       | Count of evaluations (change-triggered + periodic)           | Prefer change-triggered where possible; tune periodic frequency.                   |
| Conformance packs      | Count of packs × accounts × regions (monthly)                | Consolidate rules into a few packs; de-duplicate.                                  |
| Storage (S3)           | GB stored for histories/snapshots                            | Lifecycle to Glacier; compress; partition by account/region/date.                  |
| Remediation            | SSM Automation runs and any invoked services                 | Batch non-urgent remediations; throttle concurrency.                               |

**Cost Control Quick Wins.**

1. Narrow resource types in lower environments.  
2. Use parameters and scopes to avoid sweeping evaluations.  
3. Favor event-driven rules over aggressive periodic checks.  
4. Lifecycle S3 history aggressively.

## Operational And Security Best Practices

1. Turn it on everywhere (prod first). Recording + a minimal “critical” rule set beats perfection six months from now.  
2. Baseline with conformance packs. Start with Foundational Security Best Practices (or CIS/NIST) and tune.  
3. Separate “detect” vs “enforce.” High-confidence rules → auto-remediate; nuanced checks → ticket/approval.  
4. Parameterize rules. Make “allowed CIDR,” “required tags,” and “required KMS keys” inputs, not hard-coded.  
5. Annotate reasons. Ensure managed/custom rules provide human-readable annotation for faster triage.  
6. Tag scope. Use tags to limit rule scope and to map findings to Service/Team/Environment for ownership.  
7. Test remediation in a sandbox. Validate SSM docs idempotency, rollback paths, and guardrails.  
8. Aggregate centrally. One “Snowy-Observability” dashboard for org-wide posture; pipe exports to S3 for auditors.  
9. Integrate with incidents. Forward critical Config violations to EventBridge → Incident Manager with runbook links.  
10. Prove it. Periodically run Advanced Queries to produce audit evidence (e.g., “All EBS volumes encrypted as of March”). Save the query and export.

---

## Real-life example (end-to-end, winter names)

**Scenario.** A well-meaning change to Snowy-Data-Share accidentally flips an S3 bucket ACL to allow public reads. Within minutes:

1. **Recording & detection**  
   - The Configuration Recorder captures the ACL change as a CI.  
   - Managed rule s3-bucket-public-read-prohibited triggers on change and marks NON_COMPLIANT with annotation: “Public READ detected via ACL.”
2. **Signal → action**  
   - An EventBridge rule sees the NON_COMPLIANT evaluation for Snowy-Data-Share.  
   - It invokes SSM Automation (AWS-provided runbook to remove public read/write) with automatic remediation.
3. **Remediation**  
   - The runbook removes the offending ACL grants, verifies Block Public Access settings, and writes a change note.
4. **Evidence & communication**  
   - Config updates compliance to COMPLIANT.  
   - A message goes to Blizzard-OnCall: “Auto-remediated public ACL on Snowy-Data-Share at 14:06; change source: PutBucketAcl.”  
   - The timeline for the bucket shows: before → after, with the exact property diffs.
5. **Governance**  
   - The Winterday-S3-Baseline conformance pack score improves by one resource.  
   - An Advanced Query snapshot for the weekly audit exports “All S3 buckets, public = false, encryption = true, BPA = enabled” to S3.

Net effect: misconfig drift existed for minutes, was fixed automatically, and left behind a clean audit trail.

### Common Managed Rules

| Area     | Rule (example)                                                                 | What it checks                                                  |
|----------|---------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Network  | restricted-ssh, vpc-sg-open-only-to-authorized-ports                           | SGs don’t expose 22/3389 or unauthorized ports to the world.    |
| Storage  | s3-bucket-public-read-prohibited, s3-bucket-server-side-encryption-enabled     | No public buckets; SSE enabled.                                 |
| Identity | iam-user-mfa-enabled, iam-policy-no-statements-with-admin-access               | MFA on, avoid admin wildcards.                                  |
| Data     | rds-storage-encrypted, ebs-encrypted-volume                                    | At-rest encryption on databases and volumes.                    |
| Audit    | cloudtrail-enabled, cloud-trail-log-file-validation-enabled                    | Trails exist and have integrity protection.                     |

### Alarm/Response Pairing (Config + automation)

| Drift detected    | Rule type                 | Remediation example                                 | Response mode |
|------------------|---------------------------|-----------------------------------------------------|---------------|
| Public S3 access | Managed (change-triggered)| SSM: Disable public ACL/Policy, enable BPA          | Auto          |
| Open SSH to world| Managed (change-triggered)| SSM: Replace SG rule with approved CIDR set         | Approval      |
| Unencrypted EBS  | Managed (periodic)        | SSM: Snapshot → create encrypted volume → reattach (planned) | Ticket        |
| Missing tags     | Custom (periodic)         | SSM: Apply tags from CMDB / notify owner            | Auto/Approval |

---

## Final Thoughts

AWS Config is the backbone of governance and drift control in AWS. It doesn’t replace CloudTrail or CloudWatch—it complements them by maintaining a clean record of resource state over time, measuring compliance against what “good” means in your shop, and closing the loop with automation when something slips. If you enable it broadly, choose a sensible baseline (conformance packs), and wire high-confidence rules to safe remediation, you transform misconfigurations from “surprises in production” into “brief entries in a timeline with a fix and a receipt.”