# AWS Route 53 Resolver Logs — Deep Dive (DNS Logging in AWS)

## What Is the Service (and Why It’s Important)

When an application or service in your **VPC** wants to resolve a domain name (like `api.vendor.com`), it uses DNS resolution. In AWS, this is typically handled by the **Amazon Route 53 Resolver**, AWS’s built-in DNS server (accessible at `169.254.169.253` in every VPC).

**Route 53 Resolver Query Logging** is an *opt-in* feature that records:
- Who is making DNS queries
- What domains are being requested
- Where those queries are going

This information is critical for:

- **Security visibility**: Spot malware beacons, C2 domains, data exfiltration, or typo-squatting attempts.
- **Threat hunting**: Search for newly discovered malicious domains.
- **Troubleshooting**: Debug DNS failures and resolution path issues.
- **Compliance**: Prove DNS activity is monitored for integrity and egress control.

---

## Cybersecurity Analogy

DNS logging is like monitoring *who’s asking questions* in your office.  
You don’t know the full answer or the outcome — but you *do* know:

- **Who** asked the question  
- **What** they asked  
- **When** they asked it  
- **Where** the answer came from  

Example:  
If your developer’s laptop starts asking:

- `kittens.botnet.ru`
- `exfil.maliciousdomain.xyz`

That’s *not* normal. Even if the malware encrypts its payload, it still needs to resolve domain names. DNS becomes the last weak link — and if you’re logging it, you *catch the signal*.

## Real-World Analogy

Think of DNS logs like the **phone directory request logs** at an office helpdesk:

> Every time someone asks, “What’s the number for Alice in accounting?”, the request is logged.

You might not know what was said during the call — but if someone asks for Alice’s number 12 times at **2 AM**, something’s off. Same logic applies to cloud workloads.

Questions DNS logs help answer:

- Why is an EC2 querying a domain **500 times an hour**?
- Why is a **Lambda** resolving domains it shouldn't?
- Who resolved a **phishing domain** that just popped up?

---

## How It Works / What It Logs

You can enable **Route 53 Resolver Query Logging** per VPC and export logs to:

- **Amazon CloudWatch Logs** – for alerts, dashboards, and real-time analysis
- **Amazon S3** – for long-term archival and Athena-based threat hunting
- **Kinesis Data Firehose** – for streaming into SIEMs

Each log record includes the following fields:

| **Field**            | **Meaning**                                               |
|----------------------|-----------------------------------------------------------|
| `queryTimestamp`     | When the DNS request was made                             |
| `srcIP`              | Source IP (typically ENI address from the instance)       |
| `srcPort`            | Source port of the DNS request                            |
| `queryName`          | Domain being resolved (e.g., `api.example.com`)           |
| `queryType`          | Type of DNS record (A, AAAA, MX, etc.)                    |
| `queryClass`         | DNS class (typically `IN`)                                |
| `rcode`              | DNS response code (e.g., `NOERROR`, `NXDOMAIN`)          |
| `resolverEndpointID`| If custom endpoints used, shows which one handled query   |
| `vpcID`              | The VPC where the query originated                        |

---

## Security Use Cases

| **Use Case**                | **What DNS Logs Help You Detect**                                               |
|-----------------------------|----------------------------------------------------------------------------------|
| **Beaconing / Malware C2**  | Frequent or periodic queries to weird domains (e.g., `.xyz`, `.ru`, low TTLs)   |
| **DNS Data Exfiltration**   | DNS queries with embedded data in subdomains                                   |
| **Phishing Kit Install**    | Resolution of staging domains like `clone-paypal-login.shop`                   |
| **Command and Control Infra** | Lookups to **newly registered domains** or suspicious DGA patterns           |
| **Suspicious Developer Tools** | Connections to `ngrok`, `burp`, `requestbin` domains                         |
| **Malicious Lambda Calls**  | If a Lambda resolves domains, it may be making unapproved external calls       |

---

## Example Log Entry

```json
{
  "queryTimestamp": "2025-09-22T04:33:21Z",
  "srcIP": "10.0.42.103",
  "queryName": "auth-check-update.spotify.updates.fake.ru",
  "queryType": "A",
  "queryClass": "IN",
  "rcode": "NOERROR",
  "vpcID": "vpc-0abc123456789def0"
}
```

### The Security Team:

- Used **Athena** to query S3 DNS logs for that FQDN
- Found **4 DNS queries** from a dev EC2 over the past 7 days
- Cross-referenced with **VPC Flow Logs** — confirmed outbound HTTPS immediately after resolution

### What They Uncovered:

- The EC2 instance had a **misconfigured GitHub token**
- A malicious actor uploaded a poisoned `install.sh` script
- The script resolved the phishing domain and executed a payload

**Without DNS Logs?**  
This would’ve gone undetected. The domain was short-lived, never hit antivirus signatures, and didn’t trigger API-level CloudTrail alerts.

---

## Final Thoughts

DNS logs are **criminally underrated** in cloud security.

They don’t show full payloads.  
They don’t show who clicked what.  
But they do show **intent**.

They’re the first breadcrumb in many attacks — and the last unencrypted hint that something's going wrong.

### Route 53 Resolver Logs help you:

- See what domains your workloads are curious about  
- Catch **weird behavior early**  
- Investigate stealthy malware or exfil attempts  
- Hunt IOCs retroactively with **confidence**

> In a world of encrypted payloads, backdoors, poisoned scripts, and fleeting infrastructure —
> **DNS remains your clearest signal.**

**If you’re serious about threat detection in AWS — start with DNS.**
