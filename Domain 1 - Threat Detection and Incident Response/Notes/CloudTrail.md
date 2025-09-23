# **AWS CloudTrail**

## **What is the service (and why it’s important)**

**AWS CloudTrail** is a **logging and audit** service that **records every API call and console action** taken in your AWS environment.

Every time someone:

- **Launches an EC2**
- **Deletes a bucket**
- **Uses the AWS CLI**
- **Accesses a Lambda**
- **Assumes a role**

**CloudTrail records it.**

**CloudTrail answers the question:**

> “**Who did what, when, from where, and on which resource?**”

In security terms, this is called **non-repudiation** — **no one can deny** they did something because **it’s all recorded and timestamped**.  
It’s your **master evidence log** for all AWS account activity.

---

## **Cybersecurity Analogy**

**CloudTrail is like the CCTV system of your AWS account.**

Every time someone:

- **Walks into a building** (logs in)
- **Opens a door** (accesses a resource)
- **Pushes a button** (invokes a Lambda)
- **Breaks a window** (deletes a bucket)

You’ve got it **all on camera** with a **timestamp, location, and user ID**.

You don’t check CCTV 24/7 — but when something goes wrong, you **go back to the footage**.  
**Same with CloudTrail** — when a **breach, bug, or bad config** happens — you go into CloudTrail to:

- **Investigate**
- **Trace actions**
- **Identify the user/IP/resource/time**

---

## **Real-World Analogy**

Think of **AWS CloudTrail** like the **black box recorder** in an airplane.

Planes have thousands of things happening in flight:

- **Altitude changes**
- **Switch toggles**
- **Autopilot engaged/disengaged**
- **Engine thrust changes**

That’s a **LOT** of data — and you don't look at it unless something goes wrong.

In the same way, **CloudTrail:**

- **Records every change and access in AWS**
- **Saves logs for compliance, audit, and investigations**
- **Lets you reconstruct what happened**

**Example:**

A bucket got deleted.

- **Who did it?**  
- **From what IP?**  
- **Using what credentials?**  
- **At what time?**

**CloudTrail tells you all of it.**

---

## **How It Works / What It Does**

**CloudTrail logs all AWS account activity.** Here's the breakdown of how it functions:

### **Event Recording**

Records **every API call** made via:

- **AWS Console**
- **CLI / SDK**
- **AWS Services acting on your behalf**

**Examples:**

- `RunInstances`  
- `PutObject`  
- `DeleteTrail`  
- `AssumeRole`  
- `CreateUser`

**Every event includes:**

- **Who** made the request (**IAM user, role, or root**)  
- **What** action was taken  
- **When** it happened  
- **Where** (**IP address, region**)  
- **Which** resource was affected

---

### **Types of Events**

| **Type**             | **What it captures**                                 | **Notes**                                   |
|----------------------|-------------------------------------------------------|---------------------------------------------|
| **Management Events**| Control-plane actions (create/delete resources)       | Examples: `CreateBucket`, `AttachPolicy`, `CreateRole` |
| **Data Events**      | High-volume actions (read/write to S3, invoke Lambda) | **Not logged by default** (you must enable). **More expensive.** |
| **Insights Events**  | Unusual activity patterns                             | Detects spikes like **ConsoleLogin**, **StopInstances**. Only in **CloudTrail Insights**. |

---

### **Trail Creation**

You can create a **Trail** to send logs to:

- **An S3 bucket** (for storage)
- **CloudWatch Logs** (for real-time alerting)
- **EventBridge** (for automation or response)

You can create trails:

- **Per Region** (by default)  
- **Across All Regions** (**recommended for auditing**)

---

### **Integration**

**CloudTrail works best when connected to:**

- **GuardDuty** (for threat detection)
- **Security Hub** (to centralize alerts)
- **CloudWatch Logs** (for metrics and alarms)
- **Athena** (to search logs with SQL)

---

## **What’s Recorded in a CloudTrail Event?**

Each log contains **structured JSON** with fields like:

- **`eventTime`**: when the action occurred  
- **`eventName`**: action like `TerminateInstances`  
- **`userIdentity`**: who performed the action  
- **`sourceIPAddress`**: IP address of the actor  
- **`awsRegion`**: region of activity  
- **`requestParameters`**: input of the API call  
- **`responseElements`**: output/result of the call  
- **`resources`**: affected AWS resource

That gives you **everything needed to investigate incidents**.

---

## **CloudTrail Lake**

This is a **newer feature**.

**CloudTrail Lake** is like a **built-in log database**:

- **Stores events** in a specialized log table  
- **Lets you run SQL queries directly**  
- **No need** to export logs to **S3 + Athena**

Use it when you want:

- **Long-term log analysis** (up to **7 years retention**)  
- **Custom dashboards and security queries**  
- **Data from multiple accounts centrally**

You **pay per GB** **ingested and stored**, **plus per query**.

---

## **Pricing Model**

Here’s how pricing works:

| **Component**       | **Pricing**                         |
|---------------------|-------------------------------------|
| **Management Events** | **Free for 90 days** (most regions) |
| **Data Events (S3, Lambda, etc.)** | ~**$0.10 per 100,000 events** |
| **CloudTrail Lake** | **Pay per GB** ingested and stored |
| **Insights**        | ~**$0.35 per 100,000 events analyzed** |

**Storage in S3** incurs **regular S3 charges**.  
You can **store logs long-term** using **lifecycle rules**.  
**CloudTrail itself is free** for **basic event viewing in the console** — but **trails, Lake, and data events cost money**.

---

## **Real-Life Example**

You’re a **security engineer**, and one day:

- Your **production EC2** is **terminated**
- **Customers** are affected
- You need to **find out what happened**

You go into **CloudTrail**.  
You search for:

- `eventName = TerminateInstances`  
- `resource = i-0abcd123456789xyz`

**CloudTrail shows:**

- **Time:** Sep 18th, 12:03 UTC  
- **User:** `j.doe@company.com`
