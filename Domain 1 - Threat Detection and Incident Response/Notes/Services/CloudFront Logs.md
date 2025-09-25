# Amazon CloudFront Logs

## What Is It

Amazon CloudFront generates logs that allow you to see *every single request* made to your distribution. These logs capture:

- **Who** is accessing your content  
- **What** they’re trying to access  
- **How often**  
- Whether they’re being **blocked or allowed**  
- What **performance outcomes** occurred  

This telemetry is crucial for:

- Threat detection (e.g., scraping, bots, malformed requests)  
- Usage analytics  
- Troubleshooting edge performance  
- Monitoring billing/delivery metrics  
- Investigating abuse or anomalies  

CloudFront logs are **ground truth at the edge** — your first and closest visibility into what's really hitting your application, globally.

---

## Cybersecurity Analogy

Think of CloudFront logs like **entry logs at every outpost gate** in your global perimeter. Instead of just watching internal systems (like your EC2 or S3), you get footage of *every attempted entry point* at the edge.

These logs help you:

- Detect reconnaissance attempts  
- Identify IPs hammering your API  
- Trace stolen signed URL usage  
- Investigate suspicious download behavior  

It’s like **CCTV footage**, but across your entire digital fence — showing all attempted access, from every region, in near real time or archived batch format.

---

## Two Main Types of CloudFront Logs

### **1. Standard Access Logs (to S3)**

Batch logs that include all HTTP/S requests CloudFront processes.

- Configured per distribution  
- Delivered to an S3 bucket  
- Format: plain text with dozens of fields  
- Ideal for historical analytics or compliance

**Sample Fields:**

| Field                  | Description                                           |
|------------------------|-------------------------------------------------------|
| `date`, `time`         | Timestamp of the request                              |
| `c-ip`                 | Client IP address                                     |
| `cs-method`, `cs-uri`  | HTTP method and path                                  |
| `sc-status`            | HTTP response code (200, 403, 503, etc.)             |
| `x-edge-location`      | Which edge location handled the request               |
| `cs(User-Agent)`       | Browser or client used                                |
| `cs(Referer)`          | Referrer header value                                 |
| `x-edge-result-type`   | Cache result (Hit, Miss, Error, etc.)                 |
| `x-edge-response-type` | Final outcome after error handling                    |
| `ssl-protocol`         | TLS version used                                      |
| `x-forwarded-for`      | Client IP behind proxies                              |

---


Streamed log records within **seconds** of the request.


- Define which fields to capture (up to 24 from 40+ options)
- Set sampling rate (e.g. 100%, 10%)
- Stream to **Kinesis Data Streams**
- Ideal for SIEM, alerting, threat detection, and automated response

**Real-time logs are more expensive**, but give you second-level visibility, which is essential for live investigations.

---

## When to Use Each Log Type

| Use Case                        | Standard Logs | Real-Time Logs |
|---------------------------------|:-------------:|:--------------:|
| Retrospective forensics         | ✅            | ✅ (if stored)  |
| Compliance auditing             | ✅            | ✅              |
| Bot detection                   | ✅            | ✅              |
| Real-time anomaly detection     | ❌            | ✅              |
| Billing optimization            | ✅            | ❌              |
| Threat hunting (IP pivoting)    | ✅            | ✅              |
| Alerting & automation           | ❌            | ✅              |

> **Pro Tip:** Use **both** — archive standard logs for compliance, and process real-time logs for security intelligence.

---

## Integration with Other AWS Services

| Service         | Integration Detail                                                              |
|-----------------|----------------------------------------------------------------------------------|
| **WAF**         | Logs show which requests triggered which rules (via `x-edge-result-type`)       |
| **Shield**      | Correlate logs with mitigation metrics                                           |
| **Athena**      | Query archived logs in S3 directly                                               |
| **S3 Glacier**  | Archive logs at low cost                                                         |
| **Macie**       | Scan for sensitive data in logs                                                  |
| **CloudWatch**  | Trigger alarms based on log processing pipelines                                 |
| **Lambda@Edge** | Add headers, tag requests, or enrich logs inline                                 |

---

## Security-Specific Use Cases

### **1. Detect Credential Stuffing**
- POST requests to `/login`  
- Repeated 403s from one IP  
- Pattern: failed logins in bursts  
> Trigger WAF rule to block IP and alert SOC

### **2. Investigate Stolen Signed URL Usage**
- URI: `/downloads/private/video.mp4`  
- User-agent: `curl/7.68.0` from unknown region  
- Signature expiry mismatch → rotate signing keys  
> Logs highlight policy gap + malicious access pattern

### **3. Investigate API Abuse / Spikes**
- Sudden 10x traffic increase  
- All requests from single ASN (e.g. known scraping bot)  
- 90% of hits to `/api/health` or `/status`  
> Add CAPTCHA, header validation, geo-blocking

---


## Retention & Storage Strategy


- **S3 Archiving:** Use Object Lock or versioning + lifecycle for compliance  
- **SIEM Pipelines:** Send via Kinesis Firehose → OpenSearch or S3  
- **Threat Detection:** Kinesis + Lambda for real-time alerting  
- **Encryption:** Always encrypt logs at rest using S3 SSE or KMS


---

## Pricing Considerations

| Log Type          | Notes                                                                 |
|-------------------|------------------------------------------------------------------------|
| Standard Logs     | Free to enable; S3 PUT and storage costs apply                        |
| Real-Time Logs    | Charged per log record (based on volume + fields captured)           |
| Athena Queries    | Priced per TB scanned — use Parquet or partitions for cost savings   |
| Kinesis Stream    | Costs based on throughput + downstream processing (Lambda, etc.)     |

**Cost Optimization Tips:**
- Compress and archive standard logs
- Use **sampling** and **field selection** in real-time logs
- Only log what's actionable — avoid capturing unnecessary fields

---

## Final Thoughts

CloudFront logs are your **edge perimeter telemetry** — your earliest signal of abuse, misbehavior, or malfunction.

- **Standard logs** give you historical forensic depth  
- **Real-time logs** give you live detection capability  

Together, they give you a full-stack view into who’s knocking at your door — and what they’re trying to do before they even touch your origin.

If you’re a cloud security architect, **treat CloudFront logs like Tier 1 signals**, right up there with:

- VPC Flow Logs  
- CloudTrail  
- GuardDuty findings  

Because when it comes to public-facing workloads, **the edge is your new perimeter** — and logs are your eyes.


