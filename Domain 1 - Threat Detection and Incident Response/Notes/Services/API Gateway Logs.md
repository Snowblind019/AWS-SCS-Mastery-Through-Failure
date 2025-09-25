# Amazon API Gateway Logs

---

## What Is The Service

Amazon API Gateway Logs refer to the structured log output you can enable to monitor, audit, debug, and secure your APIs when using Amazon API Gateway. While API Gateway is the managed front-door to expose backend services (like Lambda, EC2, or HTTP endpoints), its **logging features** give you a **high-fidelity stream of what’s happening at the API level** — every request, every response, latency, errors, headers, IPs, user agents, and more.

When you’re exposing APIs publicly or to internal clients, **visibility is essential**. Logs help you answer critical questions like:

- Who is calling your API?
- What’s breaking in the backend?
- How long did each integration take?
- Are there abuse patterns (e.g., brute force or scraping)?
- Are your APIs compliant with expected usage or SLAs?

Whether for **debugging**, **performance tuning**, **auditing**, or **security incident response**, enabling logging gives your team a complete picture of how the API behaves under real-world traffic.

---

## Cybersecurity and Real-World Analogy

**Cybersecurity Analogy**  
Think of API Gateway Logs like a **guard post camera system**. Your API Gateway is the guard shack — it controls who comes in and where they go. But without the camera footage (logs), you have no idea:

- Who showed up?
- What badge they used?
- If the gate let them through or denied access
- How long they stayed
- What time they arrived

A security breach without API logs is like trying to investigate a robbery with no camera footage and no witnesses.

**Real-World Analogy**  
Imagine you’re running a toll booth on a highway. Every car that passes gives you money (API call), but you also want to track:

- Plate number (client identity)
- Lane used (resource path)
- Timestamps
- Toll amount (payload)
- Errors (invalid card, etc.)

API Gateway logs are like your **toll transaction logbook** — it’s your proof, audit trail, and optimization engine.

---

## Types of API Gateway Logs

There are **three major types of logs** in Amazon API Gateway:

1. **Access Logs**
   - Captures **who accessed what** and basic request/response metadata
   - Includes HTTP method, resource path, caller IP, status code, latency
   - Fully customizable format (JSON, CLF, CSV, etc.)
   - Sent to **CloudWatch Logs**

2. **Execution Logs**
   - Captures **step-by-step internal execution details**
   - Shows request/response transformation, mapping templates, integration calls, error stacks
   - Helps **debug mapping issues, integration errors**
   - Verbose and granular; can generate high volume

3. **Custom Logs (via Lambda)**
   - If your API is backed by Lambda, your function can emit custom logs to **CloudWatch Logs**
   - This is useful for **business-level context** or deeper application diagnostics

---

## What Gets Logged (Access Logs)

Here are some of the common **`$context` variables** you can log in Access Logs:

- `$context.identity.sourceIp`
- `$context.requestTime`
- `$context.httpMethod`
- `$context.resourcePath`
- `$context.status`
- `$context.integration.latency`
- `$context.error.message`
- `$context.authorizer.claims` (if using Cognito or JWT)

These logs are usually formatted in **JSON** and sent to a **CloudWatch Logs group** per stage.

---

## Integration with CloudWatch

When you enable logging, API Gateway writes to **Amazon CloudWatch Logs**:

- You can create a **log group per stage** (e.g., `/aws/apigateway/prod`)
- You can stream logs to **CloudWatch Dashboards**
- You can set up **CloudWatch Alarms** (e.g., 5xx spike)
- You can export logs to **S3** via subscription filters
- You can ingest logs into **SIEMs** via **Kinesis or Lambda**

> **Permissions Note:**  
> You must assign a **CloudWatch Logs role** to API Gateway so it can write logs. This is usually an IAM role with `logs:CreateLogGroup`, `logs:CreateLogStream`, and `logs:PutLogEvents`.

---

## Common Use Cases

| Use Case                | Description                                                   |
|-------------------------|---------------------------------------------------------------|
| **Security Monitoring** | Catch spikes in unauthorized access attempts, token abuse     |
| **Latency Troubleshooting** | Identify slow integration responses (e.g., slow Lambda)   |
| **Error Analytics**     | Track trends in 4xx or 5xx errors across APIs                 |
| **Client Behavior**     | Understand how your clients interact with different endpoints |
| **Incident Response**   | Audit logs during investigation of abuse or compromise        |
| **Rate Limiting**       | Detect scraping, rate floods, malformed payloads              |

---

## Best Practices

- Always **enable access logs** in production APIs
- Use **structured JSON format** for easier ingestion into log analysis tools
- Use **CloudWatch Metric Filters** to extract signal (e.g., alert when 5xx > 5/min)
- Set **retention policies** to manage log storage cost
- Tag logs with **stage/environment name** for clarity
- If using **WAF**, correlate WAF logs with API Gateway logs for full visibility
- In multi-tenant systems, log the **caller identity** or token claims

---


Logging itself doesn’t cost extra inside API Gateway — but you pay **CloudWatch pricing**:


| Component           | Pricing Basis                                  |
|---------------------|-------------------------------------------------|
| **Log Storage**      | Per GB per month                               |

| **PutLogEvents**     | Charged per API call to log events             |

| **Log Data Scanned** | Applies to CloudWatch Logs Insights queries    |
| **Dashboards/Alarms**| Standard CloudWatch rates                      |

---

## Real-Life Example

Snowy’s company has a public-facing API that allows vendors to submit product data via HTTPS. One day, someone reports delays and intermittent failures. Snowy enables **Access and Execution Logs** in CloudWatch.

They discover:

- High latency from the backend integration (`integration.latency`)
- Several 5xx errors caused by a misconfigured Lambda mapping template
- One specific client repeatedly submitting malformed JSON payloads

With this information, Snowy:

- Tunes the backend Lambda for better memory/performance
- Fixes the mapping template to handle optional fields
- Sets up a rate limit and custom error message for that noisy client

Later, when Security suspects credential abuse, the **API logs provide a full audit trail** — timestamps, IPs, JWT claims, error codes — for investigation and remediation.

---

## Final Thoughts

API Gateway Logs are **your single source of truth** for understanding what’s happening at your API edge. Without them, you’re blind to errors, latencies, abuse, and drift. With them, you gain:

- Deep observability  
- Forensics during incident response  
- Metrics to drive improvements  
- Compliance artifacts for audits  

If you expose any production APIs, enabling logging isn’t optional — it’s essential. It turns your API Gateway from a black box into a glass box — fully observable, traceable, and secure.

