# Amazon GuardDuty

Amazon GuardDuty is AWS's native threat detection service. It continuously monitors your AWS environment to identify signs of suspicious behavior or malicious activity. Whether it’s an IAM user signing in from an unusual IP, an EC2 instance contacting a known malware domain, or unexpected API activity — GuardDuty works behind the scenes to raise alerts called findings.

To better internalize its purpose and functionality, I’ve found it helpful to think of GuardDuty in terms of traditional cybersecurity tools, this is due to Amazon having a unique naming convention for their services. In many ways, GuardDuty functions as a cloud-native blend of a **SIEM**, **IDS**, and **EDR** — and, when paired with automation tools, it can even mimic the behavior of a **SOAR** platform.

---

## IDS – Intrusion Detection System

An IDS (Intrusion Detection System) simply monitors network traffic. It follows rules or intel that was input into it to detect suspicious patterns and then generates an alert. GuardDuty does this as well, but instead of monitoring networks it passively monitors:

- VPC Flow Logs  
- DNS logs  
- CloudTrail logs  
- EKS audit logs  

It has built-in threat intel (rules) that are provided and kept up to date by AWS and AWS Partners. It's like an IDS, not an IPS, in the sense that it only alerts, but doesn't block or remediate anything.

Types of threats it detects are:

- **Credential misuse** – IAM user logs in from a country it never has before  
- **Reconnaissance** (First step of Cyber Kill Chain) – attempted port scans on EC2 Instance  
- **Malware C2 traffic** (Sixth step of Cyber Kill Chain) – EC2 talking to crypto mining domain  
- **Anomalous behavior** – Root account suddenly disabling CloudTrail  
- **Known bad IPs** – Traffic from a TOR node or malicious IP known from threat intel  

---

## EDR – Endpoint Detection and Response

An EDR tool like CrowdStrike or SentinelOne runs on endpoints (i.e., laptops, servers, etc.) and monitors behavior of the device in things like file access, process creation, login attempts and, when needed, they alert and take action by isolating the device and killing the process.

GuardDuty relates to this in the following:

| EDR                     | GuardDuty                                                                 |
|------------------------|---------------------------------------------------------------------------|
| Watches behavior        | Watches AWS account and network behavior (CloudTrail, DNS, VPC Flow logs) |
| Detects threats         | Also does this by using known signatures and machine learning             |
| Endpoint = device       | In AWS, the “endpoint” is your AWS account or resources (EC2, IAM, etc.)  |

---

## What About SOAR?

A SOAR (Security Orchestration, Automation, and Response) is a SIEM that can also take action on the alerts. Normally GuardDuty can't take action on alerts, but if paired with **EventBridge** and **Lambda** it can also take action on the alerts.

EventBridge watches for findings in real time, and then Lambda executes a custom code when an alert is triggered. You can combine this with **SSM**, **SNS**, and **Step Functions** for more features.

An example flow would be as follows:

Let's say a High severity alert comes in on an EC2 instance;

1. GuardDuty raises a High severity alert for that EC2  
2. EventBridge sees the alert and matches it to a rule you made  
3. EventBridge invokes a Lambda Function that does the following:  
    - Isolates the EC2 Instance  
    - Removes it from the ASG (Auto Scaling Group)  
    - Revokes IAM (Identity Access Management) role permissions  
    - Notifies security via Slack or email using SNS (Simple Notification Service)  

---

## Analogies

These are the security analogies I use to remember the features of GuardDuty. Out of the box it performs the same as a SIEM, IDS, and EDR, but if you add other services it can become a SOAR as well. For me this is a very good way of remembering the services it offers.

In a real-world analogy, GuardDuty is like a **security camera** that watches for intruders and shady behavior by using:

- **VPC Flow Logs** – for network traffic  
- **CloudTrail Logs** – for user/API activity  
- **DNS Logs** – for domain name lookups  

It differs from CloudTrail in the fact that you don't write rules for it as you do in CloudTrail, but rather AWS does — as GuardDuty is fully managed by them. It uses **Machine Learning** and the **most up-to-date threat intel** that makes it able to give highly accurate alerts automatically.

---

## Ease of Setup

Setting up GuardDuty is also easy. All it takes is 1 click and then it automatically starts analyzing logs behind the scenes.

It keeps its eyes open for stuff such as:

- Who's calling APIs?  
- Where is traffic coming from?  
- What domains are being queried?  

It then matches the results against:

- Known attack behaviors  
- Threat intel feeds like bad IP lists, Tor, etc.  
- Machine learning that understands patterns  

After which it sends alerts (also called findings):

- EC2 instance is communicating with a known cryptocurrency mining domain  
- IAM user did something unusual at 02:00 from Russia  
- Root user used with no MFA and tried to disable logging  

These alarms come in the GuardDuty console, but you can also integrate it with other AWS services:

- **Security Hub** – to centralize all findings from different services all into one centralized place  
- **Detective** – to investigate alerts further  
- **Organizations** – for multi-account setups  
- **CloudWatch Events** – to trigger actions (like isolating an instance)  
- **Lambda** – to automatically respond to findings  

---

## More Detailed Reports on What GuardDuty Protects Against

### Detecting Account Compromise – Stolen keys, root usage from unusual IPs

- An IAM user or root user is used from a country/IP that it has never logged in from before  
- Someone logs in with an access key, and suddenly behaves very differently from usual  
- The root account does dangerous actions (e.g., disabling CloudTrail, deleting logs, etc.)  

### Detecting EC2 Compromise – Malware communication, crypto miners

- EC2 instance making connections to command-and-control servers (C2)  
- EC2 sending lots of outbound traffic = possible crypto mining  
- EC2 making DNS requests for known malware domains  

### Detecting Recon Activity – Port Scans, DNS exfiltration

- VPC Flow Logs showing many ports being scanned on your EC2  
- DNS queries that look like data exfiltration, e.g., weird subdomains like `secretdata.xyzcompany.com.attacker.com`  

### IAM Abuse – Unusual API calls from unknown locations

- An IAM user is calling sensitive APIs they never used before (e.g., `DeleteTrail`, `PutBucketAcl`)  
- Requests are made from unfamiliar IPs or regions  
- Spike in usage of actions like `AssumeRole`, `CreateAccessKey`  

### S3 Threats – Public buckets accessed by Tor IPs

Anonymous access to your S3 bucket from:

- TOR exit nodes  
- Blacklisted IPs  
- Known scanning services  

Files being read or listed from S3 buckets that were never meant to be public.

---

## Real-World Example

Now for an example that could happen in real life. Let’s say that I spin up an EC2 instance to test something. Some days later, GuardDuty generates a finding:

> "EC2 instance winterday2331 has established outbound communication with IP 72.203.88.106 — known for exhibiting high-risk behavior."

I didn’t even know that the instance was compromised in this short time of creating it. Maybe I left SSH open to the world and someone brute forced it, or some other reason. It’s good I had GuardDuty monitoring, otherwise I wouldn’t have known about this.

What GuardDuty was monitoring was the winterday2331's outbound traffic in the VPC Flow Logs. The IP address it found matched a known threat feed. 72.203.88.106 is actually a real malicious IP that is flagged as a High-Risk IP by the **minFraud** network. It shows the following signs:

- Use of anonymizing services  
- Automated traffic (bots or scripts)  
- Unusual geographic or behavioral patterns  

Because GuardDuty is being kept up to date with threat intel by AWS, it knows that this is a malicious IP and thus generated a **High severity alert** for it.

In this instance I have to:

- Shut down the instance to prevent the malware from running rampant any longer  
- Rotate the keys (I will talk more about this in the relevant domain) so that the attacker can't access the EC2 instance anymore  
- From here I can do more forensic investigating (that I won’t get into right now as it’s tied to another AWS service)  

---

## Final Thoughts

GuardDuty isn’t just a "service that raises alerts." It's a critical, automated lens into the health and security of your AWS environment.

With the right integrations, it doesn’t just tell you what’s wrong — it helps you take immediate action.
