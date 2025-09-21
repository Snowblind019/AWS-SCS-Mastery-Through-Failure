# **AWS Config**

**Config** is a record-keeping compliance monitoring service that **tracks and records the state of your AWS resources over time**. For example, some of the resources it tracks and records are:

- **What settings are turned on/off**
- **What policies are applied**
- **What tags exist**

It also checks whether your resources follow specific rules like **"All S3 buckets must be encrypted".**

**AWS Config is a vital service** due to AWS being full of switches, knobs, and permissions.

Sometimes someone makes a mistake like **disabling encryption**, **deleting a tag**, or **opens a port by accident**. In cases like these **Config helps you**:

- **Know what changed**
- **When it changed**
- **Who changed it**
- **Whether that change broke compliance**

This is vital for **auditing, security, incident response, and governance.**

The things that Config monitors are called **Configuration Items**, which simply put are a snapshot of a resource’s settings.  
For example:

- **An EC2 instance:** ID, type, tags, attached security groups, **IAM** role, etc.
- **An S3 bucket:** encryption, **ACLs**, public access, logging config, etc.

Each snapshot counts as one **“configuration item.”**

---

## **Cybersecurity Analogy**

Amazon has named this service quite well with a self explanatory name which is nice, but I still like using my analogies to remember everything. In cybersecruity **AWS Config** is like a **security camera + compliance officer**:

- The **security camera** part records **every configuration change** to your AWS resources (like a DVR for your cloud).
- The **compliance officer** part looks at these changes and says:
  - “**This EC2 instance was launched without encryption. That’s a violation.**”
  - “**This IAM role was modified. Here's what changed.**”

## **Real-World Analogy**

Imagine running a **factory** with **100 machines**.

Now, each machine:

- Has its own **settings**
- Can be **tuned differently**
- May be **compliant or non-compliant** with safety policies

You install:

1. **CCTV cameras** to watch who changes what
2. **A daily inspector** who goes to each machine and checks:
   - “Is the emergency shutoff enabled?”
   - “Is this machine properly labeled?”
   - “Is the temperature setting within the safe range?”

**AWS Config = both the CCTV and the inspector.**

---

## **What AWS Config Does**

1. **Records the state of your AWS resources**
   - Tracks configuration **snapshots over time**
   - Example: **EC2** instance type, **IAM** role permissions, **S3** bucket versioning, **VPC** rules, etc.

2. **Detects and records changes**
   - If a setting changes (e.g., encryption disabled), it logs:
     - **The new config**
     - **The old config**
     - **The timestamp**
     - **The user** or service that made the change

3. **Evaluates compliance using rules**
   - You can set **Config Rules** like:
     - “**All S3 buckets must have encryption enabled**”
     - “**EC2 instances must be in approved VPCs**”
     - “**IAM policies must not be overly permissive**”
   - Config constantly **checks your environment** against these rules

4. **Notifies you if something is non-compliant**
   - You can plug in **SNS** or **EventBridge** to take action
   - You can **auto-remediate** with **SSM** or **Lambda**

**Config monitors the following:**

- **EC2, EBS, IAM, S3, RDS, VPC, Lambda, Route53,** and **dozens more**
- Even supports **custom rules** with your own **Lambda** logic

---

## **Pricing Models**

| **Feature**            | **How It's Billed**                                                                              |
|------------------------|---------------------------------------------------------------------------------------------------|
| **Configuration Items**| Billed **per resource recorded per region per month** (e.g., one EC2 instance = one config item) |
| **Rule Evaluations**   | Billed **per evaluation** (every time a resource is checked against a rule)                      |
| **Conformance Packs**  | Billed **per pack per evaluation** (group of rules bundled together)                             |

> It can get expensive if you record thousands of resources or use lots of rules across many accounts.

---

## **Real Life Example**

Let’s say you have this rule:

> “**All S3 buckets must have encryption enabled.**”

You create that rule inside **AWS Config**.

**Here’s what happens:**

1. Config **scans all S3 buckets**
2. Sees that:
   - **Bucket A** has encryption
   - **Bucket B** has **no** encryption
3. It marks **Bucket A** as **compliant** and **Bucket B** as **non-compliant**
4. You set up **SNS notifications** for non-compliant resources
5. **AWS Config sends a message:**
   > “**Bucket B is not encrypted. Rule: s3-bucket-encryption. Region: us-east-1.**”
6. You can go into the dashboard to:
   - **View the exact setting** that triggered it
   - **See who created/edited** the bucket
   - **Review change history** over time
   - **Trigger a Lambda or SSM action** to fix it automatically

---

## **Final Thoughts**

**AWS Config** is your **eyes and memory** in the cloud.  
It **records everything**, **alerts you** when policies are broken, and lets you **prove compliance** to auditors and your security team.  
When things go wrong, **Config tells you what changed, who changed it, and when** — which is often half the battle in security.
