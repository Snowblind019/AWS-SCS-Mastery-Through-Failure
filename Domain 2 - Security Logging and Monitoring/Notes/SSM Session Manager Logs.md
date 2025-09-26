# SSM Session Manager Logs  

---

## What Is It

Session Manager, a capability within AWS Systems Manager (SSM), allows you to securely connect to your Amazon EC2 instances, on-prem servers, or edge devices **without needing to open inbound ports, manage SSH keys, or use bastion hosts.**

When enabled properly, Session Manager lets you launch secure interactive shell sessions to EC2 machines using **IAM controls, audit trails, encryption, and centralized logging**. That’s where SSM Session Logs come in.

These logs capture *everything* that happened during a session: **who connected, when, for how long, what they did**, and optionally — even the full input/output stream (command history and terminal output). This is *critical* for cloud security, compliance, incident response, and least privilege enforcement.

---

## Cybersecurity Analogy

Imagine your datacenter has a “bastion room” where engineers can go to plug directly into servers. Now imagine that:

- Every time someone enters, a guard checks their badge (**IAM**)
- A CCTV camera records everything they do (**CloudWatch Logs or S3 logs**)
- You store those tapes for 6 months in a fireproof safe (**Log Retention**)

That’s what Session Manager + logging gives you — **secure access + visibility + evidence**.

Without logs? It’s like letting people into the data center through the back door with *no cameras* and *no record* of what they touched.

## Real-World Analogy

Picture a shared laptop in a school computer lab. If multiple people log in and out, but there’s no account tracking or screen recording, you’ll never know who installed malware, deleted a file, or tampered with system settings.

**Session Manager Logs give you that accountability.** If something goes wrong (like a config change or data breach), you can go back and see *exactly who did what* — in full detail.

---

## Where Logs Can Go

You can configure Session Manager to send logs to:

| **Destination**         | **Purpose / Notes**                                               |
|-------------------------|-------------------------------------------------------------------|
| **Amazon S3**           | Long-term archival. Great for compliance, backups, forensics.     |
| **Amazon CloudWatch Logs** | Real-time visibility and querying using Logs Insights. Useful for detection pipelines. |
| **Both**                | Best practice for both short-term visibility and long-term retention. |

These logs can include:

- Session start and end time  
- Username / IAM principal  
- Instance ID  
- Commands entered  
- Full shell output (if configured)

You decide the granularity. If you want the **full transcript** (like a keylogger), that’s possible — but you must enable it explicitly.

---

## How Logging Works

You configure Session Manager preferences:

- Choose S3 bucket and/or CloudWatch Logs group  
- Assign appropriate IAM permissions to allow log delivery  

**When a session starts:**

- An SSM agent on the instance communicates with the Systems Manager backend  
- It establishes an encrypted channel between the user’s terminal and the instance  
- The agent streams logs to S3 or CloudWatch in near real-time or on session close  

You can then:

- View logs in **CloudWatch Logs Insights**
- Trigger alerts if suspicious commands are used (via **Metric Filters** or **Lambda**)
- Archive logs for X days/months/years depending on policy

---

## IAM Requirements

You must grant IAM users/roles permissions to:

- `ssm:StartSession` (start a session)  
- `ssm:TerminateSession` (terminate session)  
- `logs:*` or `s3:*` (access logs, depending on destination)

**To enforce log delivery**, use Session Policies to require that sessions *cannot* start unless logging is enabled.

**Example policy to enforce logs:**

```json
{
  "Effect": "Deny",
  "Action": "ssm:StartSession",
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {
      "ssm:SessionDocumentAccessCheck": "false"
    }
  }
}
```

---

## Security & Compliance Benefits

- No more SSH keys or open ports → **Reduced attack surface**
- Full audit trail of administrator actions → **Helps meet SOC2, PCI-DSS, ISO, FedRAMP, etc.**
- Centralized logging across all regions/accounts → **Easier detection, investigation, and correlation**
- Live analysis with **CloudWatch Logs Insights** → **Great for Security Operations Center (SOC)**

---

## Forensics Use Case

You detect **unusual traffic** from an EC2 instance. With SSM logs, you can:

- Check if anyone logged in via Session Manager  
- View timestamped commands like `curl`, `wget`, or `base64`  
- Tie it to an IAM principal (e.g., intern, contractor, automated role)  
- Download the full transcript for investigation  
- Use **CloudTrail + SSM logs** to map behavior across services  

This is **much more detailed** than CloudTrail alone.

---

## Limitations

- Logging is **not enabled by default**
- You must configure **both the destination** (S3/CloudWatch) and the **IAM roles**
- Logs only capture **Session Manager activity** — not SSH, SCP, RDP, etc.
- **Sensitive data** (e.g., passwords typed into shell) *will be logged* — be cautious
- If the instance doesn’t have the **SSM Agent** or proper **IAM role** → no session, no logs

---

## Best Practices

- Enable **both CloudWatch Logs and S3** for short-term detection and long-term storage  
- Set up **log metric filters** to flag risky commands like `chmod 777`, `curl`, `rm -rf`, `base64`, etc.  
- Create **Athena tables** on S3 logs for querying with SQL  
- Use **GuardDuty** and **Security Hub** to correlate these logs with findings  
- Rotate and prune logs based on your **data retention policy**  
- Enforce **mandatory session logging** with IAM conditions  

---

## Final Thoughts

**SSM Session Manager Logs** are one of those underrated **security goldmines**. While other services help prevent breaches, this one helps you **prove exactly what happened** when things go wrong.

Think of it as the **black box flight recorder** for your EC2 sessions. It doesn’t just tell you that someone accessed the system — it **shows you everything they did once inside.**

If you’re designing a secure cloud environment (especially under compliance regimes like **HIPAA, CJIS, or PCI-DSS**), enabling these logs is **non-negotiable**.
