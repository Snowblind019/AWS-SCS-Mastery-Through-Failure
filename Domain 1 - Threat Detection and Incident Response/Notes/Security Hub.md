# AWS Security Hub

**Security Hub is a centralized security dashboard for your entire AWS environment.**  
It gathers security alerts (called findings) from AWS services and other third-party tools, makes them easily readable so that you can view, filter, prioritize, and act on them all in one centralized place.

In the past, security data was scattered, you had to look in multiple places to see what data you wanted.  
**GuardDuty** has its own findings in its console, **Inspector** had its own vulnerability results, **Macie** had its own sensitive data alerts, and Firewalls, **SIEMs**, AV tools, etc. all had their own formats and dashboards.  
This made correlation, visibility, and prioritization difficult, and simply added **a lot** of overhead.  
To remediate this, on **June 25th, 2019**, AWS released **Security Hub**, which had the sole intention of aggregating all these into a single view making it easier to:

- Know what's happening  
- See what's most important  
- Automate responses  

---

## Cybersecurity Analogy

**Amazon has named this service quite well with a self-explanatory name which is nice, but I still like using my analogies to remember everything.**  
In a cybersecurity analogy, **Security Hub is like a SOC (Security Operations Center) Dashboard.**  
The reasoning behind saying this is that in a normal SOC, there are lots of tools running:

- IDS/**IPS** (like Snort, **Suricata**)  
- Antivirus, **EDR (CrowdStrike, SentinelOne)**  
- **SIEM (Splunk, LogRhythm, QRadar)**  
- **Vulnerability Scanners (Qualys, Nessus)**  
- **Ticketing and Incident systems (Jira, ServiceNow)**  

Each of these tools detects different types of threats ranging from malware, **misconfigurations**, failed patches, anomalies, brute-force attempts, misused privileges, etc.  
**The SOC Dashboard brings them all into one centralized place.**

---

## Real-World Analogy

**In a real-world analogy, the best is to think of a home with lots of high-tech smart devices that have many sensors:**

| **Analogy**                        | **Risk**             | **AWS Service**    |
|-----------------------------------|----------------------|--------------------|
| Door/window alarms                | Intrusion detection  | AWS GuardDuty      |
| Smoke & carbon monoxide detectors | Vulnerability        | AWS Inspector      |
| Water leak sensors                | Data exposure        | AWS Macie          |
| Camera motion sensors             | Change detection     | AWS Config         |

Now each of these sensors detect issues independently in its own way. None of them report the same issues and to add a cherry on top they all have their own apps you have to check. You can see how it'd get annoying and waste **a lot** of time having to open and check each app. Security Hub is like a wall-mounted control panel that:

| **Security Hub (Control Panel) Actions** |
|------------------------------------------|
| **Shows all alerts from all sensors in one place** |
| **Prioritizes "Fire Detected" vs "Window Opened" alerts** |
| **Trigger automated actions like turn on sprinklers, sound alarms, lock doors etc** <br> _(via integrations like EventBridge and Lambda)_ |
| **Notifies your family members or security company** _(via SNS)_ |

**Security Hub may seem like a simple service (and it is), but it has a very vital role.**

---

## How Security Hub Works

### 1) Aggregate Findings Into One Place

**It starts off by aggregating data from the various services it integrates with, and they all send their findings to Security Hub.**

#### Services That Integreate With Security Hub:

| **Service**                          | **Details**                                                                 |
|-------------------------------------|------------------------------------------------------------------------------|
| **Amazon GuardDuty**                | Threats like crypto miners, **IAM** abuse, reconnaissance                    |
| **AWS Config**                      | Sends evaluations of managed/custom rules (configuration changes, compliance checks etc) |
| **AWS Firewall Manager**            | Sends findings related to firewall rules / policies across accounts/regions  |
| **AWS Health**                      | Sends service / resource health issues that might affect security posture    |
| **IAM Access Analyzer**             | Excessive permissions or public access                                       |
| **Amazon Inspector**                | **CVEs** (vulnerabilities) in EC2, containers, **Lambda**                    |
| **AWS IoT Device Defender**         | Security issues / threat detections for **IoT** devices                      |
| **Amazon Macie**                    | Sensitive data (like **PII**) in S3 buckets                                  |
| **Route 53 Resolver DNS Firewall**  | Detects **DNS-based** threats / **misconfigurations** via DNS Firewall rules |
| **AWS SSM Patch Manager**           | Finds missing patches / OS vulnerabilities etc                               |

All the findings use **ASFF** (Amazon Security Finding Format) to provide a standardized JSON format so that all the findings follow the same formatting structure.

#### Services That Security Hub Integrates With:

**Now in addition to this, Security Hub itself integrates in other services and can send the data it receives to these services for further investigation, auditing, or action.**
| **Service**                         | **Purpose**                                                                 |
|------------------------------------|------------------------------------------------------------------------------|
| **Amazon Detective**               | To dig deeper / visualize relationships / root cause analysis of findings   |
| **AWS Audit Manager**              | Use findings for audit evidence or compliance reporting                      |
| **AWS SSM Explorer & OpsCenter**   | For aggregating operational issues, managing incidents across accounts, regions |
| **AWS Security Lake**              | Collects findings (normalized for longer-term storage / analysis / dashboards outside Security Hub) |
| **AWS Trusted Advisor**            | Can reflect overlapping findings, especially about resource usage, cost/security savings etc |


### 2) Check Against Security Standards and Best Practices

After it collects all the data, it can automatically check your AWS resources against industry standards and best practices like:

- CIS AWS Foundations Benchmark
- AWS Foundational Security Best Practices (**FSBP**)
- **PCI-DSS**
- Custom standards you create

Each standard is broken into controls like:


- `"Ensure S3 buckets are not publicly accessible"`
- `"Ensure MFA is enabled for root"`

After the checks are done, you get a pass or fail for each control you specified to run it against.


### 3) Summarize and Track Findings

- Shows summary of findings  
- Breaks down severity (High/Medium/Low)  
- Shows how many findings are:
  - Active
  - Resolved
  - Muted (manually suppressed)


### 4) Automate Responses

- **EventBridge** (to detect new findings)  
- **Lambda** (to run code when findings are created)  
- **SSM**, Step Functions, **SNS** (to notify or remediate)

An example of this would be:


- **Finding**: EC2 instance has a vulnerability (from Inspector)  
- **EventBridge** rule triggers a **Lambda** function  
- **Lambda** isolates the EC2 instance or kicks off a patching `runbook`

---

## Findings Are Alerts (Not Logs)

Something important to note is that findings aren't logs, they're alerts. They just alert you that something is wrong and needs attention.

When a finding does come in, it uses the **ASFF** format, and that looks like this:

- Title - `"EC2 instance has critical vulnerability CVE-2023-XXXX"`
- Severity - High, Medium, Low
- Product - Which service reported it (**GuardDuty**, Inspector, etc.)
- Resource - What resource is affected (EC2, S3, **IAM** etc)
- Remediation - How to fix it
- Timestamps - When it was first seen and last updated
- Compliance - Whether it violates a standard (like CIS)

---

## Real-World Example

Now for an example about how it would be in real life. The scenario: a vulnerability is found in one of my EC2 instances by Inspector.

1. **Amazon Inspector** scans my EC2 instances and finds a critical **CVE** on let's say winterday2331.  
2. Inspector sends a finding to Security Hub using **ASFF** format.  
3. Security Hub:
   - Displays the finding in the dashboard
   - Tags it with `"High Severity"`
   - Links it to the EC2 instance ID
   - Flags it as violating CIS Benchmark control `"Ensure no vulnerable packages installed"`
4. If I have **EventBridge** + **Lambda** setup to work with Security Hub it'll:
   - Isolate the EC2
   - Notify me about this
   - Start an **SSM** patching job
5. If I want to manually look into the remediation steps, I can:
   - Click into the finding to see remediation steps

---

**Security Hub may seem like a simple service (and it is), but it has a very vital role.**  
It saves a lot of time and effort by bringing everything in one place for all to see and gives it a prioritization — saving you time from trying to find what is more critical.

