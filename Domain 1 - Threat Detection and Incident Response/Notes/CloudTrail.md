# AWS CloudTrail

## What Is The Service

AWS **CloudTrail** is the **authoritative record of “who did what, when, and from where”** in your AWS accounts. It captures **control-plane API activity** (management events) and, when you opt in, **data-plane access** to specific resources (data events). Trails deliver **tamper-resistant logs to S3** (with optional integrity validation and **KMS** encryption), can stream to **CloudWatch Logs** for near–real-time alerting, and expose a **searchable Event History** in the console for quick **lookups**. With **CloudTrail Lake**, you can retain and query activity across accounts/Regions using SQL, and you can even **ingest third-party audit data** into the same lake for a unified story.

**Why it matters:** incidents, audits, and investigations don’t accept hunches—they demand **evidence**. **CloudTrail** is that evidence. When **Blizzard-OnCall** asks *“who disabled encryption on this bucket?”* or *“why did Snowy-Deploy assume that role at 02:11?”*, **CloudTrail** gives you a **verifiable timeline** you can prove, automate against, and archive.

---

## Cybersecurity and Real-World Analogy

### Security analogy.

Think of **CloudTrail** as the **badge-swipe plus CCTV ledger** for the **Winterday** campus. Every door (API) you open, every cabinet (resource) you touch, every time a master key (assume role) is used—there’s a stamped entry with **who, what, when, where, and how** (identity, action, timestamp, source IP/user-agent, request parameters, response).

### Real-world analogy.

It’s the **flight data recorder** for your AWS control plane. **CloudWatch** is your live cockpit gauges; **CloudTrail** is the black box you pull when you need to **reconstruct** exactly what happened and in what order—with checks to prove the recording wasn’t tampered with.

---

## How It Works

### Event Types (The Fundamentals)

#### • Management events (control plane)

Default activity like  
`CreateBucket`, `RunInstances`, `PutBucketPolicy`, `PassRole`, `UpdateFunctionConfiguration`.  
These answer *“who changed configuration or permissions?”*

- **Read-only** (e.g., `Describe*`, `List*`) and **Write-only** events can be included/excluded per trail.

#### • Data events (data plane)

High-volume object-level or record-level access, disabled by default for cost/noise reasons. Examples:

- **S3 object-level**: `GetObject`, `PutObject` on selected buckets/prefixes.  
- **Lambda**: `InvokeFunction` per function or by pattern.  
- **DynamoDB**: `GetItem`, `PutItem` (when enabled via Lake data sources).  
  *These answer “who accessed this specific piece of data?”*

#### • Insights events

**CloudTrail** can learn a baseline of your management-event patterns and emit **anomaly findings** when usage spikes or deviates (e.g., sudden surge in `ConsoleLogin` failures, or unusual `AuthorizeSecurityGroupIngress`). Route these to **EventBridge** for investigation.

#### • Event History (90 days)

Every account has a rolling 90-day searchable history for **management events** in the console/API, even without creating a trail.

## Trails (Delivery & Integrity)

- **Single-account or Organization trail**  
  Create a trail per account or promote to an **organization-wide** trail so all **Winterday** accounts (current and future) are covered with one configuration.

- **Multi-Region**  
  A multi-Region trail ensures management events from **all Regions** land in a single **S3 location**, avoiding blind spots from Region drift.

### Delivery

- **S3**: durable storage of JSON event files (optionally **encrypted with KMS**).  
- **CloudWatch Logs** (optional): near-real-time streaming for alerts and searches.  
- **EventBridge** (optional): route certain events to automation.

### Integrity Validation

Trails can produce **digest files** (hashes) and are **signed** so you can **cryptographically verify** files haven’t been altered. This matters in regulated environments and incident response.

## CloudTrail Lake (Search, Retain, Correlate)

- **Managed event data store** for long-term retention and **SQL queries** over **CloudTrail** events without standing up infrastructure.
- **Data sources** include your trails (management/data events), **AWS service-linked sources** (e.g., **IAM Identity Center**), and **custom sources** (partner/third-party audit feeds) so all roads lead to one query surface.
- **Lake queries** support joins/filters/time windows, great for questions like *“all actions by Snowy-Deploy within ±30 minutes of the Blizzard-Checkout deployment.”*

## Selectors, Scope, and Noise Control

- **Event selectors** let you include/exclude:
  - Read vs Write management events,  
  - Specific S3 buckets/prefixes (data events),  
  - Specific Lambda **ARNs** (invocations),  
  - Specific DynamoDB tables (via Lake) or broader resource patterns.

- **Advanced event selectors** add conditions by resource **ARN**, name patterns, or even specific fields to reduce cost and noise.

## Identities and Correlation

- **Who** is captured as IAM user/role, federated principal, or **assumed role session** with tags and `SourceIdentity` (if used).
- **From where** includes source IP, **user agent**, and **AWS service that made the call**.
- **How** includes request parameters and the response (**success/failure**).  
Pair this with **IAM Access Analyzer** and **Detective** to move from *what happened → who’s allowed to do this → how far it spread*.

## Encryption & Access

- **S3 object encryption via SSE-KMS** is recommended; scope your **KMS key policy** to the **CloudTrail** service principal plus **Snowy-Security** roles.
- Lock down the **S3 bucket policy** so only **CloudTrail** can `PutObject` and only specific principals can read.
- Use **S3 Object Ownership** and bucket **ACLs off** to simplify permissions.
- If streaming to **CloudWatch Logs**, grant the trail’s role `logs:PutLogEvents` narrowly.

## Integrations (Action the Evidence)

- **EventBridge**: route sensitive events to **SSM** Automation, Lambda, ticketing, or Incident Manager (e.g., `StopLogging` on a trail, `PutBucketAcl` public, `kms:DisableKey`).  
- **Security Hub**: posture checks (**CloudTrail** on, multi-Region, log validation) show up as controls; failures become findings you can track/automate.  
- **Detective**: pivot from a **GuardDuty** finding into the entity’s **CloudTrail** activity timeline.  
- **Athena/S3** (classic pattern): partition and query raw **CloudTrail** in S3 if you prefer your own lake, or use **CloudTrail Lake** for managed simplicity.

---

## Pricing Model

Treat the numbers as shape; exact $ varies by Region and may change.

| Area              | What Drives Cost                                                | Practical Guidance                                                                 |
|-------------------|------------------------------------------------------------------|-------------------------------------------------------------------------------------|
| **Trail delivery**| Management events included; data events billed per event; Insights per anomaly event | Be selective with data events (only sensitive buckets/functions). Use prefixes/ARN filters. |
| **CloudWatch Logs** | Ingestion + storage + queries                                 | Stream only what you need for alerts; keep S3 as system of record.                 |
| **CloudTrail Lake** | Ingested events and stored GB, plus query costs               | Use Lake for long retention and cross-acct queries; expire old data with retention policies. |
| **S3 storage**    | GB stored for trail files + digests                             | Lifecycle to Glacier for long-term, immutable retention.                           |
| **KMS**           | Requests for encryption/decryption (usually small vs. data events) | Use a dedicated CMK; audit via CloudTrail itself.                                  |

**Rule of thumb:** enable organization, multi-Region trails for management events everywhere; opt in data events surgically where object/record access truly matters (e.g., `s3://snowy-pii/`, `arn:aws:lambda:...:function:Winterday-Checkout`).

---

## Operational & Security Best Practices

1. **Org-wide, multi-Region, KMS-encrypted.**  
   Create an **organization trail**, multi-Region, **SSE-KMS**, with **log file validation** enabled.

2. **Separate buckets.**  
   Use a central **Snowy-Audit** account with a dedicated S3 bucket for **CloudTrail**; lock the bucket policy to **CloudTrail** and your security roles.

3. **Turn on Insights** for anomaly detection on management events; route to **EventBridge**.

4. **Data events: be intentional.**  
   Start with S3 objects for sensitive prefixes and Lambda invoke on high-risk functions; expand as needed.

5. **Protect the trail itself.**  
   **CloudWatch** alarms and **EventBridge** rules on `StopLogging`, `DeleteTrail`, `KMS` key disable, or S3 bucket policy changes.

6. **Retention & immutability.**  
   **S3 Object Lock** (compliance/WORM) for regulated trails; lifecycle to Glacier Deep Archive after your hot window.

7. **Tag for ownership.**  
   Tag trails and Lake stores with `Service`, `Team`, `Environment`, and route alerts to **Blizzard-OnCall** by tag.

8. **Session context matters.**  
   Standardize `sts:TagSession` and `SourceIdentity` so audit lines up with humans, CI jobs, and change tickets.

9. **Query paths.**  
   Keep Athena queries or Lake SQL snippets ready: by principal, by IP, by action, by resource, ± time window around an incident.

10. **Test recoverability.**  
    Quarterly, restore a day’s logs from Glacier and verify digests; show evidence for audit.

## Comparisons You’ll Actually Use

| Tool          | Best At                                 | How It Pairs with CloudTrail                                                          |
|---------------|------------------------------------------|----------------------------------------------------------------------------------------|
| **CloudWatch**| Live signals (metrics/logs/alarms)       | **CloudTrail** provides immutable who-did-what; **CloudWatch** tells you what hurts now |
| **Config**    | Resource state over time + compliance    | **CloudTrail** proves the API call that changed state; **Config** shows the resulting state |
| **Detective** | Investigation pivots across entities     | **CloudTrail** is a major input; **Detective** speeds the graph view                   |
| **Security Hub**| Posture & findings hub                 | Ensures **CloudTrail** is enabled/healthy; centralizes related alerts                 |
| **GuardDuty** | Threat detections                        | Uses **CloudTrail** (and others) as signal; you pivot into **CloudTrail** details for evidence |

---

## Queries & Automations You’ll Reuse

### CloudTrail Lake — Example Queries (SQL-like)

#### All actions by a role in a window

```sql
SELECT eventTime, eventName, awsRegion, sourceIPAddress, userAgent, requestParameters, errorCode
FROM   event_data_store
WHERE  userIdentity.sessionContext.sessionIssuer.arn = 'arn:aws:iam::<acct>:role/Snowy-Deploy'
  AND  eventTime BETWEEN '2025-09-24T01:30:00Z' AND '2025-09-24T03:00:00Z'
ORDER BY eventTime;
```

#### Public S3 policy changes

```sql
SELECT eventTime, eventName, userIdentity.arn, requestParameters.bucketName
FROM   event_data_store
WHERE  eventSource = 's3.amazonaws.com'
  AND  eventName IN ('PutBucketPolicy','PutBucketAcl')
  AND  requestParameters.policy LIKE '%"Principal":"*"%' ;
```

#### Who assumed a sensitive role

```sql
SELECT eventTime, userIdentity.arn, sourceIPAddress, requestParameters.roleArn, requestParameters.roleSessionName
FROM   event_data_store
WHERE  eventName = 'AssumeRole'
  AND  requestParameters.roleArn = 'arn:aws:iam::<acct>:role/Blizzard-Prod-Admin';
```

### EventBridge Rules (Fast Containment)

- **Trail tamper attempt:**  
  If `eventName` in `[StopLogging, DeleteTrail]` → **SSM Automation** to re-enable logging, page **Blizzard-OnCall**, open ticket.

- **KMS key disable** tied to the **CloudTrail CMK** → deny via **SCP** exception and notify.

- **S3 public change:**  
  `PutBucketAcl`/`Policy` with public principal → auto-apply **Block Public Access** and attach restrictive policy in the same account.

### What’s in an Event

| Field                              | Meaning                                   |
|-----------------------------------|-------------------------------------------|
| `eventTime`                       | When it happened (UTC)                    |
| `eventSource`                     | Service (e.g., `ec2.amazonaws.com`)       |
| `eventName`                       | API (`AuthorizeSecurityGroupIngress`)     |
| `userIdentity.*`                  | Who did it (user/role, accountId, session tags) |
| `sourceIPAddress` / `userAgent`   | From where / how                          |
| `requestParameters` / `responseElements` | What was attempted / result        |
| `errorCode` / `errorMessage`      | Why it failed (if failed)                 |
| `resources[]`                     | ARNs touched (when provided)              |

## Data Events — Where to Start

| Resource      | Good First Scope                                               | Why                                                 |
|---------------|----------------------------------------------------------------|------------------------------------------------------|
| **S3**        | Buckets with PII, reports, exports (`s3://snowy-pii/`, `.../exports/`) | Object access is sensitive; audit exactly who read/wrote |
| **Lambda**    | Public/internet-facing functions; admin/automation functions   | Track `InvokeFunction` sources; detect abuse        |
| **DynamoDB** (via Lake) | Tables with customer records                       | Investigate who *read which partition keys*         |

---

## Real-Life Example (End-to-End, Winter Names)

### Scenario:

Score dropped in **Security Hub**; **Blizzard-Exports** objects were read from an unusual location. Snowy needs to know **who accessed what** and **what changed** around that time.

### A. Triage

- Open **CloudTrail Lake** and run a query for `GetObject` on `s3://blizzard-exports/reports/` between 14:00–15:00 UTC.
- Results show `AssumeRole → Snowy-Partner-Role` from source IP `203.0.113.77` performing bulk reads.

### B. Scope

- Query preceding `AssumeRole` events to see who assumed `Snowy-Partner-Role`.  
  It’s `Winterday-Sharing-Service` as expected, but the **session tags** are missing the usual `approvedChangeId`.  
  That’s off.

### C. Change Correlation

- Search management events ±30 minutes.  
  Find `PutBucketPolicy` adding a wildcard to allow an extra external principal on `blizzard-exports`.  
  Actor: `Snowy-Ops-TempRole`, source IP internal, change ticket absent.

### D. Contain & Fix

- **EventBridge** rule triggers **SSM** to apply **Block Public Access** and revert policy to the **conformance pack** baseline.
- Add a **trust policy condition** requiring `aws:SourceIdentity` and a specific `approvedChangeId` tag for `Snowy-Partner-Role`.

### E. Prove & Archive

- Export the relevant **CloudTrail** files and digests, verify integrity, attach to the incident.
- Audit note: **CloudTrail** shows the exact timeline—policy change at 13:48, reads at 14:06, revert at 14:12.

**Outcome:**  
Clear, defensible chain of events; automated rollback; preventive guardrails added. **CloudTrail** provided the **who/what/when/where/how** with cryptographic assurance.

---

## Final Thoughts

**CloudTrail** is the **evidence backbone** of AWS.  
Make it **organization-wide, multi-Region, KMS-encrypted**, and **integrity-validated**; be **surgical with data events**; wire **EventBridge** to catch trail tampering and risky changes; and keep **CloudTrail Lake** (or Athena) queries close at hand.  
