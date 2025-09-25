# AWS WAF Logs

---

## What Is The Service

**AWS WAF Logs** provide detailed, per-request visibility into how AWS Web Application Firewall evaluates, accepts, or blocks web traffic. While AWS WAF is the shield protecting your apps from malicious requests (e.g., SQL injection, XSS, bots, etc.), its **logs are the black box recorder** — capturing *every single request* processed, along with rule evaluations and final decisions.

WAF Logs are not enabled by default — but once turned on, they give your security team **line-by-line details on what WAF is doing**, including which rule group triggered, what part of the request was suspicious, and how the action was taken (allow, block, count).

This is essential for:

- **Threat hunting and forensics**  
- **False positive/negative tuning**  
- **Security analytics and dashboards**  
- **SIEM correlation**  
- **Proving compliance with NIST, PCI, ISO, HIPAA**

---

## Cybersecurity and Real-World Analogy

### **Cybersecurity Analogy**  
Imagine AWS WAF is your security guard at the front door of a data center. He checks ID, scans for suspicious bags, looks for banned patterns, and allows or denies entry.  
Now imagine he **writes down every person he evaluated**, what rules they tripped (or didn’t), what part of the bag looked shady, and what decision he made — in a ledger.

That’s WAF logging. It’s the *audit trail* of security enforcement at the HTTP request level.

### **Real-World Analogy**  
Think of a bouncer at a club who doesn't just say “yes” or “no” — he logs:
- What time each person came
- What they were wearing
- What they said at the door
- Whether they were flagged as underage or drunk
- Which security rule (dress code, ID mismatch) was triggered

**Now multiply that by millions of requests per hour.**

---

## Where the Logs Go

AWS WAF doesn’t write to CloudWatch Logs directly. Instead, **logs are streamed to**:

- **Amazon Kinesis Data Firehose**, which can deliver logs to:
  - **Amazon S3** — long-term storage, batch analysis  
  - **Amazon Redshift** — for structured querying  
  - **Amazon OpenSearch** — for real-time dashboards  
  - **3rd-party SIEMs** — like Splunk, Datadog, Sentinel  

You configure a **Logging Configuration** per WebACL, and Firehose handles the delivery.

---

## What’s in a WAF Log Entry?

Each log record is a **JSON document** with full metadata about the request. Key fields:

- `timestamp`: When the request hit  
- `httpRequest`:
  - `clientIp`
  - `country`
  - `uri`, `method`, `headers`, `args`, `body`
- `action`: `ALLOW` | `BLOCK` | `COUNT`
- `terminatingRuleId`: Which rule caused the final decision
- `terminatingRuleType`: `ManagedRuleGroup` | `RateBasedRule` | etc.
- `ruleGroupList[]`: All rule groups evaluated (and which rules matched)
- `labels[]`: Tags applied to this request (e.g., `bot:known`)
- `captchaResponse`: If CAPTCHA was used
- `responseCodeSent`: Final HTTP status
- `nonTerminatingMatchingRules[]`: Rules that matched but didn't affect the final decision (e.g., COUNT mode)

---

## Key Use Cases

| Use Case                      | WAF Logs Help You...                                     |
|-------------------------------|-----------------------------------------------------------|
| **Tune rules**                | Identify false positives/negatives, add exclusions        |
| **Investigate attacks**       | Review exact payloads of XSS/SQLi probes                  |
| **Rate abuse detection**      | See spike patterns by IP, user-agent, path                |
| **User behavior modeling**    | Track which endpoints are hit, by whom, from where        |
| **WAF + GuardDuty correlation**| Enrich threat intel pipelines                            |
| **PCI/NIST compliance**       | Prove web-layer protections were in place and effective   |

---

## Logging + Analytics Best Practices

- Use **OpenSearch** with dashboards for live analysis  
- Set **filters in Kinesis** to log only BLOCK or COUNT actions if you want to reduce volume  
- Pair with **Athena** for querying logs in S3 (cheap + serverless)  
- **Redact sensitive fields** (like POST body passwords) using `RedactedFields`  
- Tag requests with **labels** and correlate with other AWS services (e.g., CloudTrail, GuardDuty)

---

## Pricing Model

Logging itself is free — but you pay for:

| Component         | Cost Basis                         |
|-------------------|-------------------------------------|
| Kinesis Firehose  | Per GB ingested and delivered       |
| S3 Storage        | Per GB-month                        |
| OpenSearch/Redshift | Standard pricing                  |
| Athena            | Per TB scanned per query            |

> 💡 High-traffic APIs with full-body logging can generate **massive logs**, so control scope carefully.

---

## Real-Life Example

Let’s say **Snowy’s team** just deployed a new API protected by WAF. After going live, some users report 403 errors when uploading photos. Snowy turns on WAF logging to investigate.
The logs show:
- The requests were `POST`s with large bodies  
- A managed rule group for SQL injection was blocking them  

- The triggering payload included strings like `filename="select.jpg"` which falsely matched SQLi patterns  

Using this insight, Snowy:
- Creates an **exclusion** for that rule when the path starts with `/upload`  
- Verifies the fix using logs in **COUNT** mode  

- Later builds a **Grafana dashboard** showing top IPs triggering blocks, top rule groups matched, and total actions per minute  

Now the team can *monitor* WAF behavior, *tune it*, and *prove* it’s doing its job.

---


## Final Thoughts

WAF Logs are your **deep inspection microscope** for web-layer traffic.  
They’re not just logs — they’re **evidence**, **debug tools**, and **threat intelligence feeds** rolled into one.

Without them, WAF is a black box.  
With them, you gain:

- Insight into attack attempts  
- Clarity on why users are blocked  
- Feedback loop for rule tuning  
- A SIEM-friendly data source  
- Proof of protection for auditors  

If you’re using WAF, **logs should always be part of the plan** — they close the loop between detection and action.

