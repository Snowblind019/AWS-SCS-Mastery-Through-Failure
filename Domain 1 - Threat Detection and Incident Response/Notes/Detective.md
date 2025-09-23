# **AWS Detective**

## **What is the service (and why it’s important)**

**AWS Detective** is a **cloud-native investigation tool** that helps you analyze and understand **security events** across your AWS environment. It **doesn’t alert you** (like **GuardDuty**), and it **doesn’t block or patch** anything (like **WAF** or **Inspector**). Instead, it helps you **investigate** — piecing together clues from logs, graphs, and timelines to answer questions like:

- **What happened to this compromised IAM user?**
- **How did this EC2 instance get accessed?**
- **Where did this API call come from?**
- **Which resources were involved in this incident?**

**Detective** builds **visual timelines** and **graphs of activity** so you can trace relationships across **accounts, users, IPs, VPCs, and workloads**.  
It’s especially valuable for **security analysts** and **incident responders**, who need **context and correlation** — not just raw alerts.

---

## **Cybersecurity and Real World Analogy**

### **Cybersecurity Analogy**
Imagine you get a **GuardDuty** alert: “**IAM user has anomalous API activity.**”  
With **Detective**, you click the user and instantly see:

- **When they logged in**
- **What API calls they made**
- **What resources they touched**
- **What roles they assumed**
- **What IP addresses they used**
- **Whether they launched EC2, touched S3, or changed VPCs**

This **saves hours of log digging**.

### **Real World Analogy**
Imagine you’re a **detective** investigating a **break-in** at a house.  
You don’t just want the security camera footage that said “someone broke in.” You want:

- **Timeline** of when each door was opened  
- **Which rooms** were entered  
- **Who else** entered the house that day  
- **Phone calls** or **alarms** made during the incident  
- **Who the suspect interacted with**

**AWS Detective** gives you that **timeline, relationship map, and evidence trail** — but for your **AWS account**.

---

## **How it works / what it does**

### **Ingests and analyzes logs automatically**
**Detective pulls in:**

- **VPC Flow Logs** (network traffic)  
- **AWS CloudTrail** (API calls)  
- **Amazon GuardDuty findings**

You **don’t need to manually enable or parse** these logs — **Detective consumes and processes** them automatically once enabled.

### **Creates a behavioral graph database**
All ingested data is **organized into a graph**:

- **Nodes:** resources (**IAM users, IP addresses, EC2 instances**, etc.)  
- **Edges:** relationships (**who accessed what, when, from where**)  

This graph is **continuously updated** and **stored for 365 days**.

### **Correlates events and builds profiles**
**Detective** automatically builds **profiles** for every entity (user, IP, instance, etc.):

- **What was normal** for this entity (usual logins, API calls)  
- **What changed recently** (new regions, higher access patterns)  
- **What other entities** it interacted with

### **Interactive investigation dashboards**
For any AWS resource (e.g., a **suspicious EC2** or **IAM user**), you can:

- **View timeline** of **API activity**  
- **See network traffic** in/out of the instance  
- **See which users or roles were assumed**  
- **Compare recent behavior vs normal**  
- **Drill down** to other **related entities** (pivoting)

It’s **click-through visual evidence** for **root cause analysis**.

---

## **Pricing Models**

**AWS Detective pricing** is based on the **amount of data it ingests per GB**.

| **Source Data Type**                                           | **Approximate Price**          |
|----------------------------------------------------------------|--------------------------------|
| **Ingested log data** (CloudTrail, VPC Flow Logs, GuardDuty)   | ~**$2.00 per GB per month**    |

There are **no upfront costs** and **no charges** for **queries, dashboards, or retention**.  
**Detective stores and visualizes 12 months of data** without extra cost.

**Cost-saving tips:**

- **Use in regions/accounts** where investigations matter  
- **GuardDuty and CloudTrail** generate lots of data — **enable selectively** if needed  
- **Estimate costs** via **AWS Pricing Calculator** before full rollout

---

## **Other Explanations (if needed)**

### **Detective is not a SIEM or alerting system**
It’s important to know that:

- It **does not send alerts**  
- It **does not detect threats** on its own  
- It **doesn’t have custom queries or rules** like a traditional SIEM

Instead, it’s **purpose-built for incident response**:

- You **start with an alert** (ex: from **GuardDuty**)  
- You **investigate with Detective**  
- You **uncover relationships** and answer the **“what happened?”** question

### **Tightly integrated with other AWS services**
**Detective** works especially well with:

- **GuardDuty** → alerts **send you into Detective**  
- **Security Hub** → **finding cards link directly** into Detective  
- **IAM and EC2 Console** → **pivot** from specific users/instances to investigate

---

## **Real Life Example**

Let’s say you’re on the **security team** at a **fintech startup**.  
One morning, **Security Hub** flags a **high-severity GuardDuty finding**:

> “**IAM User JohnDoe has credentials exfiltrated and is using API in unusual region.**”

You jump into **Detective**:

1. Click **“Investigate in Detective.”**  
2. See **JohnDoe’s timeline**: he normally operates from **US West**, but this time from **Russia**.  
3. View **API calls**: he **created new EC2 instances**, **downloaded S3 objects**.  
4. See **relationships**: instances launched in **new region**, traffic from **new IPs**.  
5. Realize he also **assumed a role** used by your **billing team**.

With all that context:

- You immediately **disable the user**  
- **Rotate credentials**  
- Start **forensic investigation**  
- **Audit** the **S3 bucket** downloads  
- **Notify impacted customers**

**Without Detective**, you’d be **sifting through CloudTrail logs line-by-line**. It would take **hours or days**.  
**With Detective**, you **visualize the entire attack timeline** in **minutes**.

---

## **Final Thoughts**

**AWS Detective isn’t about prevention or detection** — it’s about **answering questions after something suspicious happens**.  
It gives you the **forensic tooling** to **investigate, correlate, and understand** security incidents **at scale** — **without** building complex **log pipelines** or **dashboards** yourself.

It’s especially powerful for **incident responders**, **security engineers**, and **SOC analysts** who need **evidence-based, time-based, and relationship-based visibility** — **fast**.  
It helps you go from **“an alert happened”** to **“here’s what happened, when, how, and what else was affected.”**

If you're in an environment with **GuardDuty**, **Security Hub**, or **high compliance needs**, **Detective is a force multiplier**.
