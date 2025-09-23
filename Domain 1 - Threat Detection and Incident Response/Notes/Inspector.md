# **AWS Inspector**

## **What is the service (and why it’s important)**

**AWS Inspector** is an **automated vulnerability scanner** for your cloud infrastructure. It continuously assesses the security of your:

- **Amazon EC2** instances  
- **Container images in Amazon ECR**  
- **AWS Lambda** functions  

Its job is to find known vulnerabilities, such as:

- **Unpatched software**  
- **Outdated libraries**  
- **Dangerous misconfigurations**

It helps you **find these issues before attackers do**.

**Why it matters:**  
Modern attackers don’t always brute-force. They **scan for soft targets** — services with known **CVEs (Common Vulnerabilities and Exposures)**. **Inspector** constantly checks your environment against public vulnerability databases and gives you a **report you can act on**.

---

## **Cybersecurity and Real World Analogy**

### **Cybersecurity Analogy**

Imagine you hire a full-time **security guard** who:

- **Walks through your servers, containers, and Lambdas**  
- **Opens every package, script, and library**  
- **Cross-references each one** against a global book of known vulnerabilities (**CVE** database)

He finds **Apache 2.4.49** running on an EC2 instance and says:

> “Uh-oh, that version has a **path traversal** bug. That’s a **critical vulnerability**.”

Then hands you a report with:

- **What was found**  
- **Where it was found**  
- **How bad it is (Severity Score)**  
- **How to fix it**

### **Real World Analogy**

Imagine managing a **fleet of rental cars**.  
Each car has different:

- **Engine parts**  
- **Software** (navigation, auto-braking)  
- **Brake systems, airbags**, etc.

Now imagine an **auto-safety authority** says:

> “**Brake System v2.0** has a **critical defect** — urgent recall.”

**AWS Inspector** is like your **personal mechanic** that:

- **Checks every car daily**  
- **Flags** ones with **Brake System v2.0**  
- **Labels severity** (critical, high, medium)  
- **Tells you exactly** what to replace and **how soon**

---

## **What It Scans**

**Inspector covers the following resources:**

### **Amazon EC2 instances**
- Uses **Systems Manager (SSM)** to read installed software  
- Checks **OS packages, libraries, and app dependencies**  
- **No performance impact**, since it uses **inventory metadata**

### **Amazon ECR (Elastic Container Registry)**
- **Scans container images** when they’re pushed or already stored  
- Digs into the **image layers** to find **libraries with vulnerabilities**

### **AWS Lambda functions**
- Scans **Lambda function code** and **external layers**  
- Flags issues like **outdated boto3, numpy**, etc.

---

## **How It Works**

### **1) Inventory Collection (via SSM)**
- **For EC2:** Inspector uses **SSM Agent** to gather a **software inventory**.  
- **For Lambda:** it reads the **code and dependencies**.  
- **For ECR:** scans the **image layers**.

### **2) CVE Matching**
Inspector matches what it finds to the **National Vulnerability Database (NVD)**.  
It checks:

- **CVE ID** (e.g., **CVE-2025-12345**)  
- **CVSS Score** (severity: **Critical, High, Medium, Low**)  
- **Description** of the exploit  
- **Fix availability**

### **3) Continuous Scanning (event + scheduled)**
- When a **new resource is created** (EC2, Lambda, image upload) — Inspector **scans immediately**.  
- Then **re-scans daily** to catch **new CVEs**.

### **4) Finding Generation**
When a match is found:

- It **creates a Finding**  
- **Assigns a severity** based on **CVSS Score**  
- **Suggests remediation steps**  
- Optionally **sends alerts** to:
  - **AWS Security Hub**  
  - **EventBridge**  
  - **SNS, Lambda**, etc.

---

## **CVE and CVSS Refresher**

| **Term** | **What it is** |
|---|---|
| **CVE (Common Vulnerabilities and Exposures)** | Every known software flaw gets a **CVE ID** (e.g., **CVE-2023-1234**). Think of it like a **bug ID** in a massive global **security bug tracker**. |
| **CVSS (Common Vulnerability Scoring System)** | Scores the CVE from **0.0 to 10.0**. Inspector uses this score to **classify severity**. <br> **9.0–10.0 = Critical** <br> **7.0–8.9 = High** <br> **4.0–6.9 = Medium** <br> **0.1–3.9 = Low** |

---

## **Pricing Models**

**AWS Inspector is usage-based** — you pay for:

- **Number of EC2 instances scanned monthly**  
- **Number of Lambda versions scanned**  
- **Number of container images scanned**

**Here’s an idea of the pricing breakdown:**

| **Resource Type** | **Pricing Basis**        |
|---|---|
| **EC2**           | **Per instance per month** |
| **Lambda**        | **Per function version scanned** |
| **ECR**           | **Per image scanned** |

**Example:**  
If you have:

- **10 EC2 instances**  
- **5 Lambdas** (each with **2 versions**)  
- **50 ECR images**

You’re billed for:

- **10 EC2 scans/month**  
- **10 Lambda versions/month**  
- **50 image scans**

> **Cost saving tip:** You can **exclude non-production environments** or **specific resources** using **filters** to reduce scanning cost.

---

## **Real Life Example**

You spin up an EC2 instance named **prod-web-01** with:

- **Ubuntu 22.04**  
- **Apache HTTP Server v2.4.49**  
- **OpenSSL**  
- **PHP**

You also **upload a container to ECR** and **update your Lambda function**.

**Here’s what happens:**

1. **Inspector detects** `prod-web-01` via **SSM**  
2. **Scans** its software inventory  
3. Finds **Apache 2.4.49** → links it to **CVE-2021-41773** *(real CVE: RCE via path traversal)*  
4. **Generates a finding:**
   - **Title:** *Apache Path Traversal RCE*  
   - **Resource:** EC2 Instance **prod-web-01**  
   - **Severity:** **Critical (CVSS 9.8)**  
   - **Recommendation:** *Upgrade to Apache **2.4.51** or later*

**Inspector sends this finding to:**

- **Security Hub** *(if enabled)*  
- **EventBridge** → **triggers Lambda** to **isolate the EC2**  
- **SNS** → sends **SMS alert** to **SecOps**

All of this is **automated** — **no manual scans**, **no third-party tools**.

---

## **Final Thoughts**

**AWS Inspector** is your **cloud-native vulnerability watchdog**.  
It saves you from:

- **Manual scanning**  
- **Outdated tools**  
- **Delayed patching**

Security today is about **speed** and **visibility** — **Inspector** gives you **both**.

You can pair it with:

- **GuardDuty** to **detect threats**  
- **Macie** to **protect sensitive data**  
- **AWS Config** to **enforce patch compliance**  
- **Systems Manager Patch Manager** to **automate the fix**

If you want **real security maturity** in AWS, **Inspector** is a **foundational** part of that journey. It’s **not just a box checker** — it’s your **early warning system** against the most common cloud attack vector: **know**