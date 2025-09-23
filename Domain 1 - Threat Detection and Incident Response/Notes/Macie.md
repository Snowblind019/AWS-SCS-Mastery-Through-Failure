# **Amazon Macie**

## **What is the service (and why it’s important)**

**Amazon Macie** is a **fully managed data security and privacy service** that uses **machine learning** and **pattern matching** to **discover and protect sensitive data in Amazon S3**.

It helps you find things like:

- **PII (Personally Identifiable Information)**
- **PHI (Protected Health Information)**
- **Financial data** (like **credit cards**, **SSNs**)
- **API keys, passwords, private keys**
- **Custom sensitive data types** *(you define them)*

**Macie is not a scanner for EC2 or infrastructure** — it is **laser-focused on S3 buckets**.

It answers:

- “**Where is my sensitive data?**”
- “**Is it in the right place?**”
- “**Is it encrypted?**”
- “**Who accessed it?**”
- “**Did we accidentally make it public?**”

In the cloud, **data is everything** — **Macie helps make sure you don’t lose control over it**.

---

## **Cybersecurity Analogy**

Imagine you work for a **law firm** with **thousands of filing cabinets** full of documents.

Now imagine this:

- Every night, a **robot librarian** walks through each cabinet  
- **Opens each file**  
- **Reads every page**  
- Tries to **spot** things like **driver’s licenses**, **credit card numbers**, **social security numbers**, or **medical diagnoses**

If it finds something, it:

- **Flags it**  
- **Tells you exactly where it was**  
- **Labels it by sensitivity**  
- **Tells you if that drawer (S3 bucket) is locked or wide open**

That’s **Amazon Macie**.  
It’s your **robot librarian for S3** that’s trained to **spot sensitive documents**, **dangerous patterns**, and **open exposure** — and **report it back to you**.

---

## **Real-World Analogy**

Imagine you’re the **compliance officer** at a **bank**.

You have:

- **50 rooms** of filing cabinets (**S3 buckets**)  
- Each cabinet has **folders (objects)**

Some folders contain:

- **Social security numbers**  
- **Credit card applications**  
- **Bank routing info**

**Macie** is your **automated compliance officer** who:

- **Goes room to room**  
- **Opens every folder**  
- **Scans each document**  
- **Highlights** ones that contain **sensitive data**

Tells you:

- “**This folder has a customer’s SSN and it’s sitting in a cabinet that’s not locked.**”  
- “**This bucket has public read access and contains 200 files with credit card numbers.**”

And you — the **security team** — can now **take action immediately**.

---

## **What It Scans / What It Detects**

**Macie scans S3 buckets only** — it **does not** analyze **EC2, RDS, or other services**.

### **Sensitive Data in S3 Objects**
Built-in detection for:

- **Name, Address, Phone Number**  
- **SSN, Credit Card, Driver’s License**  
- **AWS Keys, Secrets, Passwords**  
- **Health info (PHI)**

You can define **custom data identifiers** (e.g., your **internal employee ID** format).

### **Bucket-Level Security Risks**

- **Public access** (ACLs or bucket policy)  
- **Unencrypted buckets**  
- **Bucket sharing across accounts**

### **Data Exposure Alerts**

- “**This file contains SSNs and is accessible to everyone on the internet.**”

### **Object Metadata**

- **File type**  
- **Size**  
- **Encryption status**  
- **Last modified**

> **Note:** Macie does **not** read **every object by default** — you configure **scheduled jobs** to **scan what matters**.

---

## **How It Works**

**Macie works in 3 major parts:**

### **1) Inventory Monitoring**

- **Automatically finds all your S3 buckets**  
- Checks for **public access, encryption, and logging**  
- **Flags misconfigured buckets** — even **before** scanning any data

### **2) Sensitive Data Discovery Jobs**

These are **scheduled or one-time jobs** you create. You select:

- **Which buckets or folders to scan**  
- Whether to use **full scan** or **sampling**  
- **Which sensitive data types** to look for

**Macie then:**

- **Reads the files** (object content)  
- **Detects patterns**  
- **Labels results**  
- **Stores Findings** in Macie and **sends to EventBridge, SNS, or Security Hub**

### **3) Findings and Actions**

Every detection **creates a Finding**.  
**Example:**

- “**SensitiveData:S3Object/Personal**”  
- “**S3 bucket is public and contains 74 credit card numbers**”

You can:

- **Trigger remediation** with **Lambda**  
- **Notify via SNS**  
- **Send to Security Hub**  
- **Export to S3** for **archival**

---

## **Clever Tech Underneath**

- **Macie uses ML models + regex pattern matching**  
- It **auto-classifies content**, even across file types (**CSV, TXT, DOCX, etc.**)  
- If you use **encryption with KMS**, Macie can still scan, **provided it has access via KMS grants**

---

## **Pricing Model**

**Macie pricing has two components:**

| **Component**              | **Price**                    |
|---------------------------|------------------------------|
| **S3 Bucket Monitoring**  | ~**$0.10 per bucket per month** |
| **Object Scanning (Classification Jobs)** | ~**$1.00 per GB scanned** |

That means:

- **Just turning on Macie doesn’t cost much**  
- But **running a scan can get expensive** — so **scope your jobs wisely**

You can:

- **Scan only certain prefixes**  
- **Enable sampling** *(vs full object scans)*  
- **Use job scheduling** to reduce cost

> **Also:** first **1 GB per month is free** for scanning *(subject to change).*  

---

## **Real-Life Example**

Let’s say you have an S3 bucket called: **`customer-records-prod`**

You launch a **Macie classification job** targeting that bucket.

**Here’s what happens:**

- **Macie scans** all objects with **.csv, .txt, .json** in that bucket  
- It **reads the content** and looks for:
  - **Names**
  - **SSNs**
  - **Email addresses**
  - **AWS keys**
- It finds:
  - **300 files** with **full names + SSNs**
  - **2 files** with **exposed AWS access keys**

It then **checks the bucket settings**:

- **Bucket is not encrypted**  
- **Public access block is not enabled**

**Macie generates 2 Findings:**

- **Sensitive data exposed in unencrypted bucket**  
- **Bucket is accessible to other AWS accounts**

**You:**

- **Get an alert** in **Security Hub**  
- Use **EventBridge** to **trigger a Lambda function**  
  - **Removes public access**  
  - **Encrypts bucket**  
  - **Sends notification** to **SecOps**

This is **automated compliance detection and reaction**.

---

## **When to Use Macie**

Use **Macie** when:

- You **store customer data, financial records, or regulated info**  
- You need to **comply** with:
  - **GDPR**
  - **HIPAA**
  - **PCI DSS**
- You want to know:
  - **Where is our sensitive data?**
  - **Who can access it?**
  - **Is it at risk?**

**Don’t use Macie:**

- For **infrastructure scanning** *(use **Inspector**)*  
- For **threat detection** *(use **GuardDuty**)*  
- To **search CloudTrail logs** *(use **CloudTrail Lake**)*

**Macie is about content classification and exposure detection.**

---

## **Final Thoughts**

**Amazon Macie** is your **data privacy watchdog** in the cloud.  
It **walks through your S3 storage** like a **data-loss prevention expert**, **finds sensitive material**, and **tells you** when it’s **exposed** or **stored incorrectly**.

It combines **machine learning**, **pattern recognition**, and **automated alerting** to help you avoid headlines like:

> “**Company leaks 100,000 SSNs via public S3 bucket.**”

**Macie** doesn’t just help you **be compliant** — it helps you **be secure by default**, and **accountable for your data**.
