# Amazon GuardDuty — Native Threat Detection in AWS

Amazon GuardDuty is AWS's native threat detection service. It continuously monitors your AWS environment to identify signs of suspicious behavior or malicious activity. Whether it’s an IAM user signing in from an unusual IP, an EC2 instance contacting a known malware domain, or unexpected API activity — GuardDuty works behind the scenes to raise alerts called **findings**.

To better internalize its purpose and functionality, I’ve found it helpful to think of GuardDuty in terms of traditional cybersecurity tools. In many ways, GuardDuty functions as a cloud-native blend of a **SIEM**, **IDS**, and **EDR** — and, when paired with automation tools, it can even mimic the behavior of a **SOAR** platform.

---

## SIEM-Like Behavior

Traditional SIEMs like Splunk, QRadar, or LogRhythm aggregate logs from multiple sources and generate alerts based on suspicious patterns. GuardDuty does this too — but it’s purpose-built for AWS. It ingests:

- **CloudTrail logs** (API activity)  
- **VPC Flow Logs** (network traffic)  
- **DNS query logs**  
- **EKS audit logs**

It applies AWS-managed threat intelligence and machine learning models to detect threats in real time.

---

## IDS-Like Capabilities

Like an IDS (Intrusion Detection System), GuardDuty watches for known attack patterns or anomalous behaviors — but without interfering with network traffic. It is passive, meaning it detects but does not block. Detection is powered by:

- Pre-integrated threat intelligence feeds from AWS and partners  
- Signature-based detection  
- Machine learning for anomaly detection

Think of it like a highly trained security guard that never sleeps — it doesn’t block the intruder, but it will immediately raise the alarm.

---

## EDR-Inspired Detection

Traditional Endpoint Detection and Response (EDR) tools like CrowdStrike or SentinelOne monitor endpoints for anomalies (e.g., lateral movement, strange processes). GuardDuty mimics this at the cloud level:

| EDR Monitors                         | GuardDuty Equivalent                         |
|-------------------------------------|----------------------------------------------|
| Endpoint devices (laptops, servers) | AWS account and resource behavior            |
| Local file/process activity         | API calls, network traffic, log patterns     |
| Lateral movement, privilege escalation | Unusual IAM usage, credential misuse    |
| Malware callbacks                   | EC2 contacting known command-and-control servers |

---

## Turning GuardDuty into a SOAR (with EventBridge and Lambda)

On its own, GuardDuty only generates alerts. But when paired with automation tools like EventBridge, Lambda, Step Functions, and Systems Manager, it becomes a reactive, automated responder — much like a SOAR (Security Orchestration, Automation, and Response) platform.

### Example Workflow:

1. GuardDuty raises a high-severity finding for an EC2 instance contacting a cryptomining domain.  
2. EventBridge matches the finding to a rule.  
3. Lambda is invoked and performs the following:
   - Isolates the EC2 instance  
   - Removes it from the Auto Scaling Group (ASG)  
   - Revokes its IAM role permissions  
   - Notifies the security team via Slack or SNS  

By combining services, GuardDuty evolves from a passive detector into an active first responder.

---

## Setup Simplicity

GuardDuty requires:

- No agents  
- Just one click to enable  

Once enabled, it immediately starts analyzing logs in the background with no impact on system performance.

### GuardDuty Watches:

- Who’s calling APIs  
- Where network traffic originates  
- Which domains are being queried  

It compares these behaviors against:

- Threat intelligence feeds (e.g., TOR exit nodes, malware IPs)  
- Known attacker patterns  
- Machine learning models  

All alerts are logged as **Findings** and can be forwarded to:

- **Security Hub** (for centralized visibility)  
- **Amazon Detective** (for investigation)  
- **EventBridge** (for triggering automation)  
- **Lambda** (for incident response)

---

## What GuardDuty Detects

### Account Compromise

- IAM user signs in from an unfamiliar country  
- Root account disables CloudTrail or deletes logs  
- Access key usage deviates from normal behavior  

### EC2 Compromise

- EC2 contacts known C2 (Command-and-Control) domains  
- Sudden outbound traffic spikes suggest cryptomining  
- DNS queries to malware-related domains  

### Reconnaissance Activity

- DNS queries that resemble data exfiltration attempts  


### IAM Misuse

- Sensitive API calls (e.g., `PutBucketAcl`, `DeleteTrail`) from unusual locations  
- Unexpected actions like `AssumeRole`, `CreateAccessKey` from new IPs  

### S3 Threats

- Anonymous access to S3 from TOR or blacklisted IPs  
- Unusual listing or downloading of sensitive data  

---

## Real-World Analogy


GuardDuty is like a smart building security system:

- It monitors the hallways (**VPC Flow Logs**)  
- Watches the doors (**CloudTrail API calls**)  
- Tracks who’s asking for directions (**DNS queries**)  
- Has a direct line to security experts (**AWS Threat Intel**)  

For example, it might alert you:  
_"Someone at Door #3 is on every global watchlist. You might want to check that out."_

---

## Personal Example

I once spun up a temporary EC2 instance for testing. A few days later, GuardDuty flagged:

**“EC2 instance `winterday2331` is communicating with IP `72.203.88.106` — part of a known cryptojacking operation.”**

I hadn't realized that SSH was open to the world — someone likely brute-forced it. GuardDuty detected outbound traffic to a known malicious IP using the VPC Flow Logs.

That IP had already been flagged by AWS threat intelligence for:

- Anonymization behavior  
- Bot activity  
- Suspicious geographic access  

### Because of GuardDuty, I was able to:

- Receive a high-severity alert  
- Shut down the EC2 instance  
- Rotate any associated credentials  
- Begin a full investigation  

Without GuardDuty, I might not have noticed anything until serious damage was done.

---

## Final Thoughts

GuardDuty isn’t just a "service that raises alerts." It's a **critical, automated lens** into the health and security of your AWS environment.

With the right integrations, it doesn’t just tell you what’s wrong — it helps you take immediate action.

