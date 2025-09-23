# **AWS GuardDuty**

## **What is the service (and why it’s important)**

**AWS GuardDuty** is a **threat detection** service. It continuously monitors your **AWS accounts, workloads, and data** for **malicious activity** and **unauthorized behavior**.

Think of it as a **security camera system** for your AWS environment:

- **It watches your logs 24/7.**
- **It identifies suspicious activity** using **threat intelligence, machine learning, and anomaly detection**.
- **It doesn’t block the attack itself** — it **alerts you** so you can act fast.

**Why it matters:**

Cloud environments are complex and hard to monitor manually. **GuardDuty** gives you **automated threat detection** without needing to build your own **SIEM (Security Information and Event Management)** system.

---

## **Cybersecurity and Real World Analogy**

### **Cybersecurity Analogy**
Think of **GuardDuty** as a **SOC (Security Operations Center) Analyst** that:

- **Reviews all your logs**
- **Knows the latest hacker tricks and malware patterns**
- **Flags anything suspicious:** credential exfiltration, unusual API calls, port scanning, Bitcoin mining, etc.

It’s like **hiring an AI security analyst** who **works 24/7**, never sleeps, and **understands AWS-specific risks**.

### **Real World Analogy**
Imagine your **smart home** has **security cameras, motion sensors, and door alarms**. You connect them to a smart hub (like Ring or Nest) that:

- **Knows when you’re usually home**
- **Alerts you** if **someone opens a window at 3am**
- **Recognizes** the difference between **your dog** and **a burglar**

**GuardDuty is that smart hub — but for your cloud.**

---

## **How it works / what it does**

### **Ingests AWS logs automatically (no agents to install):**

- **VPC Flow Logs** → captures network activity  
- **CloudTrail Logs** → captures AWS API activity  
- **DNS logs** → captures domain name lookups  
- **EKS audit logs (optional)** → captures Kubernetes activity  
- **S3 data access logs (optional)** → captures access to S3 buckets  

### **Analyzes using built-in intelligence:**

- Uses **machine learning** to detect anomalies (ex: login from a strange country)  
- Uses **threat intel feeds** from AWS and partners (ex: known malware IPs)  
- Uses **pattern analysis** (ex: sudden spikes in API calls or unexpected EC2 traffic)  

### **Generates “Findings”:**

Each finding is a **detailed alert** showing:

- **What happened**
- **What AWS resources were involved**
- **Severity** (Low, Medium, High)
- **Recommended next steps**

### **Integrates with other services:**

- **Security Hub:** Centralizes your findings across AWS  
- **EventBridge:** Trigger automated remediation actions (e.g., quarantine EC2)  
- **AWS Detective:** Investigate findings further with **timeline + relationships**  

---

## **Pricing Models**

**GuardDuty is pay-as-you-go.** You’re billed for the **amount of data analyzed**, not the number of alerts or users.

| **Component**       | **Approximate Pricing**                 |
|---------------------|-----------------------------------------|
| **VPC Flow Logs**   | ~**$1.00 per million events**           |
| **CloudTrail Events** | ~**$4.00 per million events**         |
| **DNS Logs**        | ~**$1.00 per million queries**          |
| **S3 Protection**   | ~**$1.00 per million objects monitored**|
| **EKS Protection**  | **Based on audit log volume**           |

There’s a **30-day free trial** for **each region per account**.

**Tips to control costs:**

- **Enable only in regions you care about**
- **Exclude high-volume test accounts**
- **Use filtering** (include/exclude log sources or resources)

---

## **Other Explanations (if needed)**

- **GuardDuty is NOT a firewall, WAF, or antivirus**  
  **It doesn’t block anything.** It only **detects and alerts**.

- **You don’t write rules or signatures**  
  Unlike IDS/IPS or WAFs where you define patterns to detect, **GuardDuty** uses **AWS-managed intelligence and ML models**.

- **No agent required**  
  Works by **analyzing logs** — no need to install software on **EC2** or **containers**.

- **Complementary to other AWS services:**
  - Use **AWS Config** to check if resources are compliant
  - Use **Inspector** to scan for vulnerable packages
  - Use **Macie** to detect sensitive data exposure
  - Use **Detective** to investigate GuardDuty findings

---

## **Real Life Example**

You work for a **small fintech** company using AWS.  
One day, **GuardDuty** raises a **High Severity Finding**:

> **"AnomalousBehavior:User/AnomalousBehavior.UnusualConsoleLogin"**

You open the alert and see:

- **IAM User:** `billing-admin`  
- **Country:** **Russia** (you normally operate in the **US**)  
- **API call:** Logged in via Console, **attempted to access S3**  
- This user **never logged in outside business hours** before

**What you do next:**

- **Disable the user** or **revoke credentials**
- **Notify your security team**
- **Investigate further** using **AWS Detective**
- **Check** if any **S3 data** was downloaded
- **Rotate sensitive credentials**

All of this **started from a single, automatic finding** by **GuardDuty**.

---

## **Final Thoughts**

**GuardDuty** gives you **smart threat detection** without the overhead of **building your own alerting system**.  
It’s **AWS-native**, **low-effort**, and incredibly powerful for:

- **Monitoring for compromised IAM credentials**
- **Detecting malicious EC2 behavior**
- **Catching lateral movement** and **command-and-control traffic**
- **Surfacing risky behavior** before it becomes a breach

If you already have **CloudTrail** and **VPC Flow Logs**, **enabling GuardDuty is a no-brainer** for **early warning detection** and **defense in depth**.
