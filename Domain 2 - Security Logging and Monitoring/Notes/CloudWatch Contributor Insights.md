# Amazon CloudWatch Contributor Insights  

---

## What Is The Service

Amazon CloudWatch Contributor Insights is a specialized feature of CloudWatch that lets you identify the **top contributors** to performance problems, high error rates, or usage spikes across your infrastructure.

It’s not a logging service itself — instead, it analyzes patterns in **existing logs or metrics** and visualizes who or what is causing the most impact.

It helps answer questions like:

- “Which API users are causing the most errors?”  
- “Which IP addresses are responsible for the most failed login attempts?”  
- “Which Lambda functions are throwing the most exceptions?”

This is incredibly useful in:

- **Security incident response**  
- **Operational troubleshooting**  
- **Cost and performance optimization**  
- **DDoS detection and abuse hunting**

---

## Cybersecurity Analogy

Imagine you're running a **security checkpoint** with 10 gates. Suddenly, one gate gets overwhelmed, and alerts start firing.

You need to know:

- “Who are the top people coming through that gate?”  
- “Which country, IP range, or access card ID?”

That’s what Contributor Insights does. It turns raw log streams into **live-ranked leaderboards** — like a **Top 10 Offenders dashboard** — so you can focus defensive attention where it matters.

## Real-World Analogy

Contributor Insights is like the **Top Played Songs** chart on Spotify — but for your AWS logs.

Instead of artists, you’re tracking:

- Which API calls are getting hit the most  
- Which accounts are making those calls  
- Which resources are being overutilized  

It doesn’t just give you a spreadsheet. It shows you **live heat maps**, **charts**, and **breakdowns** — so your response teams can **act fast and with context**.

---

## How It Works

Contributor Insights uses **log patterns** you define in a rule, applied to:

- **CloudWatch Logs** (VPC Flow Logs, Lambda logs, App logs, etc.)  
- **CloudWatch Metrics** (e.g., invocation counts, error rates)

You configure a rule like:

- “Track the top N contributors by `source IP` where `action = REJECTED`”  
- “Track which `API call + userAgent` is generating the most 5xx errors”

It then:

- Scans logs/metrics **every minute**  
- Aggregates and counts based on fields you specify (IP, user, API call, etc.)  
- **Visualizes results** as time series graphs with Top-N contributors  

You can then:

- **Trigger alarms** if a contributor exceeds a threshold  
- Correlate spikes with **CloudWatch Metrics**  
- Use **Insights with Dashboards** for centralized visibility

---

## Key Concepts

| **Concept**         | **Description**                                                  |
|---------------------|------------------------------------------------------------------|
| Contributor         | The entity being analyzed (IP, account ID, API call, etc.)       |
| Rule                | A pattern definition telling Insights what to track and group by |
| Top N Ranking       | Displays top contributors by count over time                     |
| Aggregation Period  | Typically 1-minute intervals for near real-time visibility       |
| Metrics or Logs     | Can analyze structured JSON logs or CW metrics with dimensions   |
| Live Visualization  | Graphs show spikes, rank changes, and contributor behavior       |

---

## Security Use Cases

| **Use Case**              | **Example**                                                   |
|---------------------------|---------------------------------------------------------------|
| Brute-force login detection | Find top source IPs failing login requests                 |
| API key abuse             | Detect users making excessive calls to sensitive APIs        |
| Credential stuffing       | Show rapid spikes in failed logins by region or IP           |
| Data exfiltration         | Identify users downloading large volumes of S3 data          |
| DDoS or traffic analysis  | Correlate large traffic increases to specific contributors    |

---

## Example Rule: Brute Force Login Detection

```json
{
  "logGroupNames": ["/aws/lambda/AuthFunction"],
  "filter": "{ $.status = 'FAILED_LOGIN' }",
  "contribution": {
    "keys": ["$.sourceIP"],
    "value": "$.status"
  }
}
```

This rule would highlight the **top IP addresses** causing failed login attempts.

---

## Integration with CloudWatch Dashboards and Alarms

Contributor Insights rules can be:

- **Embedded in CloudWatch Dashboards** to show live contributor graphs  
- **Linked to CloudWatch Alarms** to alert when top contributors exceed thresholds  

**Example:** Alert if a single IP causes more than **1,000 failed logins in 5 minutes.**

---

## Pricing Overview

| **Component**           | **Pricing**                                                    |
|--------------------------|---------------------------------------------------------------|
| Contributor Insights     | $0.005 per 1,000 log events analyzed                          |
| Storage                  | Standard CW Logs ingestion and storage pricing applies        |
| Visualization            | Included with CloudWatch Dashboard usage                      |
| Alarms                   | Standard CW alarm pricing (first 10 alarms are free)          |

---

## Final Thoughts

**Contributor Insights** is a power tool for **real-time root cause analysis**.

While tools like **GuardDuty** or **Macie** focus on security intelligence, Contributor Insights gives you the **raw telemetry at a granular level**, allowing you to ask:

- “Who’s overloading this Lambda?”  
- “Which user is causing our API throttles?”  
- “Which CIDR block is flooding our application logs?”

It doesn’t replace **CloudTrail** or your SIEM — it **enhances** them with **Top-N operational analytics** at the “who is doing what the most right now” layer.
