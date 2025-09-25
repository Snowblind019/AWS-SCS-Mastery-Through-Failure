# AWS Macie

## What Is The Service

Amazon Macie is AWS’s data security and privacy service for S3. It automatically **discovers**, **classifies**, and **protects** sensitive data (PII, financial, health, credentials, secrets) and surfaces **where that data lives**, **who can access it**, and **how risky the bucket’s posture is**. It’s built to answer the questions that keep teams up at night:

- Where is our sensitive data actually stored?
- Is any of it public or overly exposed?
- What kinds of personal data do we hold, and in which regions/accounts/buckets?
- What changed this week—and do we need to act?

**Macie** does two big things well:

1. **Sensitive data discovery** in S3 objects (full jobs or automated sampling), using a mix of **managed** and **custom data identifiers**.  
2. **Bucket-level posture assessment** (public access, shared access, encryption status, policy risks), so you can spot “time bombs” even before scanning object contents.

For **Snowy’s** teams, that means moving from “I think we’re fine” to **evidence-backed visibility**, then wiring findings into automated guardrails.

---

## Cybersecurity And Real-World Analogy

**Security analogy.** Think of **Macie** like a confidential-documents unit in a security program:

- One crew **walks the halls** (your S3 estate) checking **doors and signs** (bucket access policies, encryption).  
- Another crew **opens sample boxes** (automated discovery) and **audits contents** with a trained eye (data identifiers).  
- When they find a box labeled “misc.zip” full of passports and **SSNs** in a shared closet, they **flag it** and **call the owner** with a remediation checklist.

**Real-world analogy.** It’s inventory + inspection at a warehouse:

- The inventory board says what **aisles exist and who can enter** (bucket posture).  
- Inspectors **spot-check** boxes to find mislabeled high-value items (sensitive data discovery).  
- Findings turn into **tickets** so the right team re-shelves, locks, or repackages the goods.

---

## How It Works

### Two modes of discovery

1. **Automated Sensitive Data Discovery (ASDD)**  
   *Macie* continuously and intelligently **samples objects** across your S3 estate (per account/Region). It learns where your sensitive data tends to be, surfaces trends, and reduces the operational grind of scheduling scans everywhere. This is the “set it and keep watch” mode—great for ongoing posture awareness.

2. **One-time or Scheduled Classification Jobs**  
   You define **scope** (accounts, buckets, prefixes, tags), **filters** (file types, last-modified, size), and run targeted scans. Use this for:
   - Mergers/audits,  
   - Before enabling external sharing,  
   - After incident indicators,  
   - When a specific **Winterday** bucket smells risky.

**Result:** *Macie* produces **findings with location, type, examples (redacted), and owner context** so you can act.

## What Macie Looks For

- **Managed data identifiers:** common **PII** (names, addresses, phone numbers), **national IDs** (e.g., SSNs), **financial** (credit cards, bank numbers), **healthcare** terms, **auth tokens** (access keys), **secrets** patterns, and more.  
- **Custom data identifiers:** your own **regex + context** rules (e.g., *Winterday-CustomerID*, *Snowy-MemberCode*, internal invoice formats).  
- **File types:** broad coverage (text formats, **JSON/CSV**, many office docs, PDFs, archives where supported). You can include/exclude types for cost and speed.  
- **Allow lists** (via regular expressions or exact values) help suppress **known benign matches** (e.g., test **SSNs**) to reduce noise.

## Bucket Posture & Exposure Analysis

*Macie* continuously evaluates:

- **Public access** (ACLs, policies, Block Public Access state),  
- **Shared access** (cross-account),  
- **Encryption status** (default encryption, **KMS** usage),  
- **Policy risks** (broad **Principal** or wildcard conditions).

You get **bucket-level findings** and an **estate view** so **Snowy** can prioritize *“Blizzard-Data-Share is public + unencrypted”* ahead of lower-risk issues.

## Findings, Classification Details, And Triage

*Macie* findings include:

- **Type** (`SensitiveData:S3Object` / `Policy:S3Bucket`),  
- **Where** (account, Region, bucket, prefix, object),  
- **What** (category and **count** of sensitive data types),  
- **Samples** (contextual excerpts, masked),  
- **Timestamps & owners** (bucket tags, AWS account).  

You can **publish findings** to **EventBridge**, pipe to **Security Hub**, notify via **SNS**, or hand them to **SSM Automation** for known fixes (e.g., lock down access, enable encryption, move the object).

## Multi-Account, Multi-Region

- Pick a **Macie administrator account** (e.g., *Snowy-Security*).  
- Enroll **member accounts** (or integrate with AWS Organizations for auto-enrollment).  
- Centrally manage **ASDD**, **jobs**, **findings**, and **suppression rules**.  
- Mirror the setup across Regions where you store data.

---

## Pricing Model

Keep this as a **shape** rather than memorizing numbers (pricing evolves):

| Area                          | What drives cost                                  | Design guidance                                                                 |
|------------------------------|---------------------------------------------------|----------------------------------------------------------------------------------|
| Automated Sensitive Data Discovery | Estate size (buckets/objects sampled) with tiered/object-based pricing | Use **ASDD** broadly for trend awareness; tune object sampling with scoping if needed. |
| Classification Jobs          | **GB scanned** per job (data inspected)           | Target scans (prefix/tags/last-modified); exclude compressions/duplicates when possible. |
| Bucket posture               | Usually included with **Macie enablement**        | Keep on; it’s cheap signal for high-value posture gaps.                          |

## Cost Control Quick Wins

- Start **ASDD** in prod and high-risk accounts first; expand as needed.  
- Scope one-time jobs by **prefix/tags/last-modified** to avoid cold archives.  
- Use **allow lists** and **custom identifiers** to reduce false positives and re-scans.  
- Export **reports** and disable scans on buckets you’ve archived to Glacier Deep Archive.

## Comparisons You’ll Actually Use

| Tool            | Best at                                      | How Macie fits                                                                 |
|----------------|-----------------------------------------------|---------------------------------------------------------------------------------|
| **GuardDuty**   | Threat **detections** (anomalies/IoCs over logs) | *Macie* focuses on **data classification & exposure** in S3; send findings into **Security Hub** alongside **GuardDuty**. |
| **Security Hub**| Posture **aggregation** & standards scoring  | *Macie* feeds findings to **Hub**; Hub gives you a unified queue & workflows.  |
| **Config**      | **Resource state** + compliance rules        | Use **Config** to enforce **encryption/BPA/tagging**; use *Macie* to prove content sensitivity. |
| **Inspector**   | **Vuln scanning** (EC2/ECR/Lambda)           | Different layer: Inspector = compute/software; *Macie* = data.                 |
| **Glue / Athena**| **ETL** & **query** data lakes              | Don’t use them to classify sensitive data. Use *Macie* first, then move/transform safely. |

---

## Operational & Security Best Practices

1. **Turn on ASDD** in your prod and sensitive **Winterday** accounts/Regions first; watch trends for 2–4 weeks.  
2. **Tag ownership** on buckets (`Service`, `DataOwner`, `PII`, `Retention`) so findings route to the right team.  
3. **Build custom identifiers** for your domain (customer IDs, invoice numbers, internal forms). Pair with **allow lists** to mute known test data.  
4. **Scope jobs tightly** for investigations: filter by **prefix, last-modified, file type, size**; avoid re-scanning stale archives unless required.  
5. **Route findings** to **EventBridge → SSM Automation** for standard fixes (enable encryption, block public access, quarantine to a private bucket).  
6. **Suppress noise** with time-boxed **suppression rules** (e.g., for sanctioned test buckets) and revisit them monthly.  
7. **Measure posture:** track % of buckets with **BPA enabled**, % encrypted with **KMS**, and **open exposure** findings over time. Put these on a **Snowy-Data Security dashboard**.  
8. **Treat PII in dev** like prod or redact it at ingestion. Create a *Macie* job that **fails the pipeline** if **PII** shows up in non-prod inputs.  
9. **Document ownership & escalation:** when *Macie* finds **SSNs** in `Blizzard-raw-drops/`, who fixes it within 24h? Bake this into your **on-call runbooks**.  
10. **Prove it for audits:** export *Macie* sensitive data discovery results and bucket posture reports monthly to **S3**; attach to audit evidence packs.

## Findings You’ll See

| Category                | Example finding                          | Why it matters               | Typical action                                                  |
|------------------------|-------------------------------------------|------------------------------|-----------------------------------------------------------------|
| `SensitiveData:S3Object`| `Multiple.Personal` or `Credentials` in object | **PII/keys in the wild**     | Quarantine/move, notify owner, add deny policy, scrub/redact    |
| `Policy:S3BucketPublic` | Public read/write or policy wildcard     | **External exposure risk**    | Enable **Block Public Access**, fix bucket policy/**ACL**       |
| `Policy:S3BucketShared` | Cross-account overly broad               | **Data leakage across accounts** | Restrict principals, introduce access patterns (role assumption)|
| `Policy:S3BucketEncryption` | Missing default encryption           | **At-rest exposure**         | Enforce **KMS** default encryption, rotate keys if needed       |

## Real-Life Example

**Scenario.** A partner integration is about to pull weekly reports from `s3://blizzard-exports/reports/`.  
Before granting access, **Snowy** wants assurance there’s no **PII** in that path and that the bucket isn’t exposed.

1. **Baseline posture**  
   *Macie* shows Block Public Access: **enabled**, default encryption: **KMS**, no public or wildcard sharing on `blizzard-exports`. ✅ Good.

2. **Targeted job**  
   **Snowy** runs a **classification job** scoped to `reports/` with filters:
   - Include: `*.csv`, `*.parquet`, `last-modified <= 30 days`  
   - Exclude: files > `512 MB` (handled separately)  
   - Custom identifier: `Winterday-CustomerID` (regex + context words)

3. **Findings**  
   - One `SensitiveData:S3Object` finding flags **names + phone numbers** in `reports/2025-09-w38-raw.csv` (unexpected—should have been anonymized).  
   - No credentials or national IDs detected.  
   - No policy findings on the bucket.

4. **Action**  
   **EventBridge** routes the finding to **SSM Automation**:
   - Quarantines the object to a **private quarantine bucket**  
   - Notifies **Blizzard-Data** owner group with the **Macie** evidence excerpt  
   - Opens a ticket to fix the **ETL** (apply anonymization before export)

5. **Close the loop**  
   The team patches **ETL**, re-runs the job, and *Macie* reports **no sensitive data** in `reports/`.  
   The partner role is granted **prefix-scoped read access** with **KMS condition keys**.

**Outcome:** Data was protected *before* external sharing; evidence is stored with the incident ticket for audit.

### Scoping A Job

| Filter         | Why you use it              | Example                               |
|----------------|-----------------------------|----------------------------------------|
| Prefix         | Limit blast radius          | `s3://winterday-datalake/raw/2025/`   |
| Tags           | Route by owner/domain       | `DataOwner = Blizzard-Finance`        |
| File types     | Avoid binaries you don’t need| Include `CSV/JSON/PARQUET` only       |
| Last-modified  | Focus on recent data        | `>= now() - 30d`                       |
| Size bounds    | Keep scans fast             | `<= 512 MB` (large go to separate flow)|

### Building Custom Identifiers

| Element       | Purpose              | Example                                 |
|---------------|----------------------|------------------------------------------|
| Regex         | Match the token      | `\bSNOWY-\d{8}\b`                         |
| Context words | Reduce false positives| “customer id”, “member id”, “account ref”|
| Proximity     | Tighten match window | `± 50 chars`                             |
| Allow list    | Exempt test values   | `SNOWY-00000000`                         |

---

## Final Thoughts

*Macie* is how you **turn uncertainty into facts** about your S3 data. With **automated discovery** running as a background radar and **targeted jobs** for the risky corners, you get continuous signal on **what’s sensitive, where it lives, and how exposed it is**.  
