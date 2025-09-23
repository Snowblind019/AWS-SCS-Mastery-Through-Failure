# **AWS Config**

## **What is the service (and why it’s important)**

**AWS Config** is a **resource configuration recorder and compliance checker**.  
It **tracks how your AWS resources change over time**, **records the exact configuration at every point**, and **tells you if those configurations violate rules or best practices**.

Think of **AWS Config** as your **always-on surveillance system** for your AWS environment. It answers questions like:

- “**Who changed the S3 bucket to public?**”
- “**When did this security group get opened to 0.0.0.0/0?**”
- “**Is this IAM role compliant with our policy?**”
- “**Was this EC2 instance tagged properly at launch?**”
- “**What did the resource look like last week?**”

In security and auditing, **knowing the state of everything, at all times, is priceless**.

---

## **Cybersecurity Analogy**

Imagine you’re a **security auditor** walking through a **datacenter** with a **clipboard and camera**:

- You **write down every firewall rule, server setting, port configuration**  
- You **revisit that room every few minutes**  
- If **anything changed**, you **record the difference**  
- You **check your clipboard against compliance policies**: “Are firewalls configured correctly?”, “Are password policies in place?”

That’s what **AWS Config** does in your cloud. It:

- **Monitors** the state of your cloud  
- **Captures snapshots** of your resources  
- **Detects drift**  
- **Compares** against **compliance rules**  
- **Stores a timeline** of everything for **audit/review**

---

## **Real-World Analogy**

Imagine **managing a massive warehouse** with **hundreds of machines**.  
You install **CCTV cameras** on every aisle and set **motion sensors**.

Every time someone:

- **Changes a machine’s setting**
- **Unplugs something**
- **Moves a box from Shelf A to Shelf B**

… your system **logs the change**, **records a before/after snapshot**, and **checks** if that change **broke any safety rule** (e.g., “Don’t block emergency exits”).

That’s **AWS Config** for your **cloud resources**.

---

## **How It Works / What It Does**

**AWS Config** does **three major things**:

### **1) Resource Recording**
Tracks configurations for **supported AWS resources**.  
**Examples:** **S3, EC2, IAM, Security Groups, VPCs, Lambda**, etc.

Each config includes metadata like:

- **Settings**
- **Tags**
- **Permissions**
- **Relationships** *(e.g., this EC2 is in this VPC)*

### **2) Timeline and History**
It stores a **versioned timeline** for each resource. You can:

- **View history** of changes  
- **Compare past versions**  
- **See who/what made the change** *(via CloudTrail)*

### **3) Compliance and Rules**
You create **Config Rules**: checks that say **what’s allowed**.

- **Example:** “**S3 buckets must not be public**”  
- **Example:** “**EC2 instances must be in a VPC**”

These can be:

- **AWS Managed Rules** *(prebuilt)*  
- **Custom Lambda Rules** *(you write logic)*

**AWS Config** evaluates your resources against rules, and shows:

- **Compliant**
- **Non-compliant**
- **Last checked timestamp**
- **Why it’s non-compliant**

**Bonus:** You can **remediate non-compliant resources automatically** via **SSM Automation Docs**.

---

## **Deep Integration With**

- **CloudTrail:** Shows **who changed what and when**  
- **SNS/EventBridge:** **Send alerts** on rule violations  
- **SSM:** **Run automation** on non-compliant findings  
- **Security Hub:** **Send compliance posture**

---

## **Pricing Model**

**AWS Config** pricing has **two components** (plus Conformance Packs):

| **Component**                   | **Cost**                          |
|---------------------------------|-----------------------------------|
| **Configuration Items Recorded**| ~**$0.003 per item**              |
| **Config Rules Evaluations**    | ~**$0.001 per evaluation**        |
| **Conformance Packs**           | **Bundle of rules — billed by # of evaluations** |

**Configuration Item (CI)** = **1 resource snapshot/version recorded**.  
Every time a resource **changes**, a **new CI** is stored. **Can become expensive in large environments.**

**To reduce cost:**

- **Scope** which **regions/services** are recorded  
- **Exclude noisy or ephemeral resources**  
- Use **Aggregators** in **multi-account** setups

---

## **Real-Life Example**

Let’s say you work at a company where **S3 buckets must never be public**.

You:

1. **Enable AWS Config** in **us-west-2**  
2. Turn on the managed rule: **`s3-bucket-public-read-prohibited`**  
3. **AWS Config** begins **scanning all buckets** in that region  

One day, a dev makes **`my-bucket-prod`** **public**.  
**Config** detects this on next evaluation:

- **Marks bucket as Non-Compliant**  
- **Sends alert** to **SNS**  
- **Triggers a remediation action** using **SSM**:
  - **Automatically changes** the bucket policy **back to private**

You go to the **Config dashboard** and:

- **See the resource**  
- **See the rule violation**  
- **View the history** of configuration changes  
- **See timestamp + CloudTrail entry** of **who made the change**

This is **automated detection, auditability, and correction** — **all native to AWS**.

---

## **Conformance Packs**

Think of **Conformance Packs** as **bundles of rules** that align with compliance standards like:

- **CIS AWS Foundations Benchmark**
- **NIST**
- **HIPAA**
- **PCI DSS**

They **group multiple Config Rules into one pack** and show **pass/fail posture** in bulk.

It’s like AWS saying:

> “**Here are 60+ best practices. Let us scan your account and tell you which you pass or fail.**”

---

## **When to Use Config (and When Not To)**

**Use AWS Config when:**

- You need **compliance auditing**  
- You care about **who changed what, when, and how**  
- You want to **enforce guardrails** on how **AWS is used**  
- You want a **record of all resource states over time**

**Don’t use it as:**

- A **real-time security tool** *(that’s **GuardDuty**)*  
- A **network monitor** *(that’s **VPC Flow Logs**)*  
- A **vulnerability scanner** *(that’s **Inspector**)*

Instead, use **Config** for **visibility and governance**.

---

## **Final Thoughts**

**AWS Config** is like your **auditor-in-the-cloud**.  
It **watches your entire AWS environment**, **takes detailed notes** every time something changes, and **flags anything** that breaks your rules.

It’s **not** about “live attack detection” — it’s about **ensuring that your cloud stays within guardrails**.  
For **compliance-heavy industries** like **finance, healthcare, or government** — it’s **essential**.  
For **DevOps teams**, it’s a **powerful way** to enforce **good practices** and **automate corrections**.

If you care about **who did what, when**, and **whether your environment is following best practices** — **AWS Config is your best ally**.
