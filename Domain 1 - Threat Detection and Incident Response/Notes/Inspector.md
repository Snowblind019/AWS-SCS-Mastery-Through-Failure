# **AWS Inspector**

**Inspector** is a vulnerability scanner for your cloud environment. It automatically scans your **Amazon EC2** instances, containers (in **ECR**), and **Lambda** functions for known security vulnerabilities. These vulnerabilities consist of things like **outdated packages**, **unpatched software**, or **misconfigurations** that could be exploited.

**Inspector's value shines** when you're searching for holes in your infrastructure that attackers can exploit. It's most useful when you have **EC2 instances**, **containers**, and **Lambda** code that attackers are constantly looking for vulnerabilities in—stuff like unpatched packages and vulnerable libraries—so they can exploit it. **AWS Inspector finds** those holes in your security and shows you exactly where they are and how severe they are, so you can fix them before an attacker finds them.

---

## **Cybersecurity Analogy**

Amazon has named this service quite well with a self explanatory name which is nice, but I still like using my analogies to remember everything. In a cybersecurity analogy, **Inspector is like a security guard with a CVE checklist**. The guard wants into every system (**EC2**, containers, **Lambda**), looks at what's installed, checks the versions of every software, and then pulls out a giant book of known vulnerabilities (**CVE's**) and says:

> “Hmm... **OpenSSL** version **1.1.1g**? That's vulnerable to **Heartbleed**. That's a critical problem.”

**After which** it hands you a report saying:

- **What it found**
- **How bad it is (severity)**
- **Where it is (which EC2/container)**
- **How to fix it (remediation)**

## **Real-World Analogy**

Imagine you're managing a fleet of rental cars.

Every car you have has:

- Its own engine parts
- Brakes, software systems, etc.

Now a global automotive authority announces that:

> “Brake System 2.0 has a major defect that could cause accidents.”

**AWS Inspector is like your automated mechanic**, who:

- Checks every car in your fleet **every day**
- Sees if Brake System 2.0 is in any of your cars
- Flags it if it's present
- Rates how urgent it is to fix
- Tells you exactly how to replace or patch it

---

## **What It Scans**

Inspector scans the following:

### **1. EC2 Instances**
- It reads your installed OS packages, software, and libraries.
- It doesn't interrupt or slow down your EC2, it **reads** the **SSM (Systems Manager)** inventory data.

### **2. Amazon ECR (Elastic Container Registry)**
- It scans container images that are uploaded or already in the registry.
- It checks for vulnerable libraries inside images.

### **3. Lambda Functions**
- Scans Lambda function code and any layers (external packages).
- Checks for vulnerable packages, like outdated **NumPy**, **Boto3** etc.

---

## **How It Works**

### **1. Inventory Collection (via SSM)**
For EC2 instances, Inspector uses **AWS SSM (Systems Manager)** to gather information about installed software.

### **2. Constant Scanning (Event-Based & Daily)**
- New EC2 instance? **Scanned automatically.**
- New container image pushed? **Scanned automatically.**
- New Lambda published? **Scanned automatically.**
- It also **rescans EC2 and Lambda resources daily** for new **CVE's**.

### **3. CVE Matching**
Inspector matches the inventory data against the **National Vulnerability Database (NVD)**, a database of all known **Common Vulnerabilities and Exposures (CVEs)**.

### **4. Finding Generation**
When a match is found:
- Inspector generates a **Finding**
- Assigns a **CVSS-based** Severity Score
- **Suggests** Remediation Steps
- Sends the Finding to **Security Hub** (if integrated)

If needed here is an explanation on **CVE's** and **CVSS**:

| **CVE** | **CVSS** |
|---|---|
| **CVE** stands for *Common Vulnerabilities and Exposures*, it's like a “known vulnerability encyclopedia.” | **CVSS** stands for *Common Vulnerability Scoring System*, it scores vulnerabilities to put simply. |
| Each **CVE** has an ID (e.g., **CVE-2025-12345**) and a severity score. | Score from **0.0 to 10.0**, the higher the number the worse the **CVE** is. <br> • **9.0 – 10.0 = Critical** <br> • **7.0 – 8.9 = High** <br> • **4.0 – 6.9 = Medium** <br> • **0.1 – 3.9 = Low** |

---

## **Pricing Models**

Inspector uses a **pay-as-you-go** model, there is no upfront cost. You're billed per resource scanned, every month.

As of 2025, their pricing model is:

| **Resource Type** | **Pricing Model** |
|---|---|
| **EC2 Instances** | Charged **per instance scanned per month** |
| **ECR Container Images** | Charged **per image scan** |
| **Lambda Functions** | Charged **per function version scanned** |

> You can reduce costs by excluding resources you don't need to scan.

---

## **Real-Life Example**

Imagine you launch a new EC2 instance called **"web-prod-us-east-1"** with **Ubuntu 22.04**.

This instance you launched has **OpenSSH**, **Apache2**, and a few other packages installed.

1. Inspector **detects** the new EC2 via **Systems Manager**.  
2. **Scans** the software list.  
3. Notices **Apache** version **2.4.49**.  
   - **Title:** “Apache HTTP Server Path Traversal Vulnerability”
   - **Severity:** Critical

   - **Affected Resources:** EC2 instance ID
   - **Remediation:** “Upgrade to Apache version 2.4.51 or later”

You can then use **EventBridge** to automatically trigger:
- A **Lambda** that isolates the EC2.
- An **SSM Patch Manager** job that installs the update.
- An **SNS** notification to alert your team.


---

## **Final Thoughts**

**Inspector** is like your always-on vulnerability watchdog.  
It does the repetitive, but vital work of checking every system, every day, against the world's list of known weaknesses.  
In security, **speed matters**, and Inspector gives you the head start you need to stay ahead.

