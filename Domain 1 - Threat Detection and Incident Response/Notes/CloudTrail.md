# **AWS CloudTrail**

**AWS CloudTrail** is a **log of every single action** that happens in your AWS account.  
If **anything is done** — launching an **EC2**, deleting an **S3** bucket, changing security group rules, or even just logging in — **CloudTrail records it**.  
It’s your **source of truth** for **who** did **what**, **when**, and **from where**.

That’s critical in cybersecurity for:

- **Auditing**
- **Incident response**
- **Threat hunting**
- **Compliance**
- **Investigations**

---

## **Cybersecurity Analogy**

Amazon has named this service quite well with a self explanatory name which is nice, but I still like using my analogies to remember everything. In cybersecurity **CloudTrail is like the security camera system** for your AWS environment.

Every **API call** is like someone entering a room, turning on a switch, opening a drawer — **CloudTrail records all of it**.  
Without **CloudTrail**, it’s like running a company with **no security cameras and no visitor log**.  
If something bad happens, you’d have **no idea who did it or how**.

## **Real-World Analogy**

Let’s say you run a **library**.

- Patrons check out books  
- Staff restock shelves  
- Admins change store hours  

**CloudTrail is the logbook at the front desk:**

- When someone **checks out a book** → **it’s logged**
- When a **shelf is rearranged** → **it’s logged**
- When someone **sneaks in after hours** → **also logged**

Without it, you’d never know **who changed something or when it happened**.

---

## **What Does It Log?**

**Every API call** made in your AWS account:

- **Who** made the request (**IAM** User, Role, etc)
- **What** was requested (create, delete, modify)
- **When** it happened (timestamp)
- **From where** (IP address)
- **Response result** (success/failure)

It captures activity from:

- **Console**
- **CLI**
- **SDK/API**
- **AWS services acting on your behalf**

### **Where Are Logs Stored?**

**CloudTrail** sends the logs to:

- **S3 bucket** (default destination)
- Optionally: **CloudWatch Logs** or **EventBridge**

**You can enable:**

- **Management events** (e.g., create/delete EC2, change IAM policy)
- **Data events** (e.g., access to S3 objects, Lambda function invocations — more granular but not enabled by default)
- **Insights** (detect unusual activity patterns — optional, paid feature)

### **Types of Trails**

| **Trail Type**        | **Scope**                                                         |
|-----------------------|-------------------------------------------------------------------|
| **Single-region**     | Captures activity in one region only                              |
| **Multi-region**      | Captures global activity (**recommended**)                        |
| **Organization trail**| Logs actions across all linked AWS accounts via **AWS Organizations** |

## **CloudTrail ≠ CloudWatch**

- **CloudTrail** = **What happened?** (who/what/when/where)  
- **CloudWatch** = **How did the system perform?** (CPU, memory, logs, metrics)

Think of **CloudTrail** as **audit logs**, **CloudWatch** as **performance monitoring + system logs**.

---

## **Pricing Models**

| **Feature**            | **Pricing Notes**                               |
|------------------------|--------------------------------------------------|
| **Management Events**  | First copy is **free for 90 days** in Event History |
| **S3 Delivery (logs)** | Charged for **S3 storage** used                  |
| **Data Events**        | Charged **per event** recorded                   |
| **Insights**           | Charged **per insight** generated                |

### **Summary:**

- Basic auditing (**management events**) is **free** in **Event History (90 days)**  
- If you want to **store logs long-term**, **analyze them**, or use **insights**, you’ll pay:
  - **S3 storage**
  - Optionally **CloudWatch Logs**
  - **Data events** per request
  - **Insights** per alert

---

## **Real Life Example**

Let’s say you discover that an **S3 bucket was deleted unexpectedly**.  
You go to **CloudTrail** and search for:

- **DeleteBucket** events

You find:

- **IAM user:** `Winter`  
- **Action:** `DeleteBucket`  
- **Time:** **03:32 AM**  
- **Source IP:** **192.0.2.55** (unknown IP)  
- **User Agent:** **AWS CLI**

Turns out:

- Intern was testing scripts with **admin privileges**
- Accidentally deleted a **production** bucket

**Without CloudTrail:**

- You’d have **no idea who deleted it**
- **No timestamp**
- **No trail** to investigate

**With CloudTrail:**

- You can **prove what happened**
- You can **apply lessons** (limit IAM, add MFA, etc.)
- You can **restore** from logs/backups (if you had them)

---

## **Final Thoughts**

**AWS CloudTrail** is your **“source of truth”** for every action in AWS.  
It’s essential for tracking down **what happened, who did it, and how to prevent it next time**.  
Without it, **you’re blind**. With it, you have **accountability, visibility, and control**.
