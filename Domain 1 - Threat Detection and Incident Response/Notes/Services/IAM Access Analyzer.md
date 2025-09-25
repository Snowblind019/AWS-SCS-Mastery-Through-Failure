# AWS IAM Access Analyzer  

---

## What Is The Service

**IAM Access Analyzer** is a built-in tool within **AWS Identity and Access Management (IAM)** that helps detect resources in your AWS environment that are accessible **from outside your account** — whether publicly, cross-account, cross-service, or via federation.

It uses **automated reasoning** powered by the **Zelkova engine**, which performs logic-based analysis to **mathematically evaluate access paths**. This means it doesn't need logs, past activity, or user behavior — it tells you with certainty *what access is possible based solely on policy configurations.*

Why it matters: In sprawling AWS environments with complex bucket policies, KMS key permissions, and trust relationships, it's dangerously easy to create accidental exposure. IAM Access Analyzer proactively finds those before someone exploits them.

---

## Cybersecurity and Real-World Analogy

**Cybersecurity Analogy:**  
Imagine you’re managing a massive cloud data center with thousands of doors (resources), each secured by locks (policies). Some doors might be unlocked or shared without your knowledge. IAM Access Analyzer is like a drone that flies through the facility, checking every lock, and says:  
> “This door can be opened by this external party.”  

No brute-force. No breach required. It just *knows*, based on the logic of the lock.

**Real-World Analogy:**  
It’s like hiring a contract lawyer to audit every agreement (IAM, trust, bucket policies). Instead of waiting for a legal issue to arise, the lawyer reads every clause and warns you:  
> “This paragraph allows this external vendor to access your sensitive data.”  

It’s a **proactive audit**, not a reactive log search.

---

## Key Capabilities

| Capability               | Description                                                                 |
|--------------------------|-----------------------------------------------------------------------------|
| **Automated Reasoning**  | Uses math/logic (Zelkova) to evaluate IAM and resource policies             |
| **Find External Access** | Detects if a resource is exposed to another AWS account or public internet  |
| **Find Public Access**   | Flags resources accessible to everyone (`Principal: "*"`)                   |
| **Policy Validation**    | Identifies overly permissive or conflicting policy statements               |
| **Multi-Account Support**| Can scan across all AWS Organization accounts                               |
| **Custom Analyzers**     | Scopes can be region/account/resource-specific                              |

---

## How It Works

IAM Access Analyzer works by **creating analyzers** that scan your environment for **possible access paths**.

### Analyzer Scope Options:
- **Account-level** (default)
- **Organization-level** (cross-account visibility)
- **Region-specific**

Once an analyzer is created, it continuously monitors supported resources for policy changes. Any detected **external access** is reported as a **finding**.

### Types of Access It Detects:

- **Public access** (`Principal: "*"` in any policy)
- **Cross-account access** (AWS account IDs outside your own)
- **Cross-service access** (e.g., Lambda or CloudWatch from another account)
- **Federated access** (e.g., SAML or OIDC identity providers)

---

## Resources It Analyzes

| Resource Type                 | Policy Type Evaluated                    |
|------------------------------|------------------------------------------|
| **S3 Buckets**               | Bucket policies                          |
| **IAM Roles**                | Trust policies (`AssumeRole`)            |
| **KMS Keys**                 | Key policies                             |
| **Secrets Manager Secrets**  | Resource policies                        |
| **SQS Queues**               | Resource policies                        |
| **SNS Topics**               | Resource policies                        |
| **Lambda Functions**         | Resource policies                        |

---

## Findings Example

Imagine a bucket policy like this:

```json
{
  "Effect": "Allow",
  "Principal": "*",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*"
}
```

Even if you didn’t know this was in place, Access Analyzer would generate a finding like:

> **Resource** `arn:aws:s3:::my-bucket` **is accessible by anyone on the internet using** `s3:GetObject`.

You can then choose to:
- **Fix** the policy  
- **Archive** the finding (if intentional)  
- **Automate** remediation steps via EventBridge

---

## Types of Findings

| Finding Type       | Meaning                                                                 |
|--------------------|-------------------------------------------------------------------------|
| **Public Access**   | Resource is accessible to *everyone* (no auth required)                 |
| **Cross-Account**   | Another AWS account can access the resource                             |
| **Cross-Service**   | Another AWS service/account combo can interact with it                  |
| **Federated Access**| An external SAML/OIDC user can access via assumed role or direct access |

---

## Using Access Analyzer in Practice

### **1. Create an Analyzer**
- Via Console, CLI, or Terraform  
- Choose scope: **Account** or **Organization**  
- Region-specific

### **2. Review Findings**
- Findings are displayed in the console  
- Can be filtered, searched, exported  
- You decide whether to fix, archive, or automate

### **3. Take Action**
- Tighten permissions (`aws:SourceIp`, `aws:PrincipalOrgID`, etc.)  
- Remove unnecessary cross-account access  
- Validate assumptions in trust policies  
- Archive known/intentional exposures

### **4. Automate Responses**
- Send findings to **Amazon EventBridge**  
- Trigger a **Lambda** function to:
  - Notify security teams  
  - Auto-remove risky policy elements  
- Integrate with **AWS Security Hub**

---

## Example Security Use Cases

### **1. Prevent Public S3 Buckets**
- Catch `"Principal": "*"` before a customer or attacker finds it

### **2. Validate IAM Trust Relationships**
- Detect `AssumeRole` policies that allow access from unknown accounts

### **3. Detect Data Exfil Risks**
- Catch if **Secrets Manager** secrets or **KMS keys** are readable by test/dev accounts

### **4. Design Validation**
- Enforce internal-only access with conditions like:
```json
"Condition": {
  "StringEquals": {
    "aws:PrincipalOrgID": "o-abc123"
  }
}
```

### **5. Least Privilege**
- Spot policies that are *too broad* for the workload (e.g., all actions on all resources)

---

## Pricing

IAM Access Analyzer is **free** to use.

| Feature                | Cost              |
|------------------------|-------------------|
| Creating analyzers     | **Free**          |
| Running analysis       | **Free**          |
| Storing findings       | **Free**          |
| EventBridge/Lambda use | Pay per service   |

You only pay for **optional** services triggered in response to findings.

---

## Final Thoughts

**IAM Access Analyzer** is your **proactive lens** into IAM exposure — surfacing **what *could* happen**, not what already did.  

Other tools like **GuardDuty** and **CloudTrail** react to activity. Access Analyzer looks at **policy intent** and tells you:  
> “This configuration allows external access — even if no one’s used it yet.”

In a complex AWS Org with cross-account Lambdas, vendor access, dev/test environments, and evolving policies — this visibility is *invaluable*.

It’s like **X-ray vision** for your security team. You get to **see through the policies** and identify risks before they become incidents.
