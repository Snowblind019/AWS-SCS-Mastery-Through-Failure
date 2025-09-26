# AWS Service Quotas 

---

## What Is The Service

**AWS Service Quotas** is a **centralized dashboard and API** that lets you view and manage **resource limits** (also called *quotas*) for AWS services in your account.

Every AWS service — EC2, Lambda, IAM, etc. — has built-in thresholds that prevent abuse, overuse, or system strain. These quotas govern:

- Max EC2 instances per region  
- Max IAM roles  
- Max VPCs, subnets, EIPs, and more  

Most quotas are **soft** (can be increased), while others are **hard** (fixed).

### Why It Matters

If you **don’t track your quotas**, you risk:

- Failed deployments  
- Blocked scaling events  
- Outages due to “silent throttling”  

For **security and governance**, quotas help:

- Limit blast radius  
- Enforce privilege boundaries  
- Prevent runaway costs  

You can also create **Custom Quotas** and **CloudWatch alarms** to monitor usage proactively.

---

## Cybersecurity Analogy

Think of Service Quotas like a **governor on a high-speed engine**.

Without it? The system could burn out or crash. In cloud security, quotas are like **circuit breakers**:

- Prevent **resource exhaustion attacks**  
- Limit damage from **misconfigured automation**  
- Provide **containment during incident response**

You're not just limiting usage for billing — you're **limiting exposure**.

## Real-World Analogy

Imagine you're managing a **corporate office building**:

- “Only 10 visitors per room”  
- “Max 50 people on Floor 3”  
- “5 printers per department max”

These aren’t technical limits — they’re **administrative safeguards** to prevent chaos, fire hazards, or resource hogging.

**Service Quotas = AWS’s version of those rules**:

- “No one can spin up a 96-core EC2 without approval.”  
- “No dev team can create 1,000 IAM roles from a Terraform bug.”  

---

## Types of Quotas

| **Quota Type**         | **Description**                                           | **Examples**                             |
|-------------------------|-----------------------------------------------------------|-------------------------------------------|
| Service-Level Quotas    | Per-service resource limits                               | 20 EC2s per Region, 200 IAM Roles         |
| Account Quotas          | Limits at the AWS Account level                           | 5,000 API Gateway RPS                     |
| Hard Quotas             | Non-changeable limits (safety/infra reasons)              | 5 CloudTrail trails per Region            |
| Soft Quotas             | Default limits — can be increased via request             | 75 S3 Buckets, 10K Lambda concurrency     |
| Custom Quotas           | Internal rules + CloudWatch monitoring                    | 5 EC2s per team, 2 VPCs per project       |

---

## How It Works

With **Service Quotas**, you can:

- View **current limits** per region & service  
- Track **current usage** (where supported)  
- Request **increases** via Console/API  
- Set **CloudWatch alarms** when nearing limits  

### Example Flow

1. You check and see 5 EIPs used out of 5 allowed  
2. You request a quota increase to 10  
3. AWS approves (automatic/manual depending on risk)  
4. You get headroom, and optionally alarm at 8 EIPs used  

You can also enable **Trusted Advisor alerts** for quota risk notifications.

---

## Security & Operations Use Cases

| **Scenario**                        | **Why Quotas Help**                                                               |
|-------------------------------------|------------------------------------------------------------------------------------|
| Prevent DoS from automation loops   | Stop infinite resource creation from buggy IaC                                     |
| Limit blast radius per environment  | Separate quota ceilings for Dev/Test/Prod accounts                                 |
| Audit privilege boundaries          | Understand what’s *possible*, not just what’s *allowed*                            |
| Enforce cost ceilings               | Cap high-cost resources (e.g., EBS, EC2 instance families)                         |
| Abuse prevention during a breach    | Stops attackers from deploying massive infra if access is gained                   |
| Operational health tracking         | Use CloudWatch to detect quota nearing failure thresholds before outages           |

---

## CloudWatch Integration

You can create alarms based on **quota usage percentage**, like:

- **80% EIP usage** → Notify security  
- **IAM Role count near limit** → Alert ops  
- **Block EC2 launches** when **custom quota exceeded**  

This gives **predictive awareness** *before* you get hit with “LimitExceeded” errors.

---

## Limit Increase Requests

| **Step**                  | **Details**                                                                 |
|---------------------------|------------------------------------------------------------------------------|
| Manual via Console/API    | Use **"Request quota increase"** or CLI: `aws service-quotas`               |
| Some auto-approved        | Others need **manual AWS review** (esp. high-risk changes)                  |
| Per-region scope          | Quotas are often regional — increases apply **per region**, not global      |
| Billing consideration     | Higher quotas = more usage potential → use as **cost guardrails**           |

### Example CLI
```bash
aws service-quotas request-service-quota-increase \
  --service-code ec2 \
  --quota-code L-0263D0A3 \
  --desired-value 20
```

---

## Key Features Table

| **Feature**             | **Description**                                                              |
|--------------------------|-------------------------------------------------------------------------------|
| Dashboard View           | See all quota values across AWS services in one place                        |
| Programmatic Access      | API + CLI support for automation, alerting, and visibility                   |
| CloudWatch Integration   | Set alarms for usage thresholds                                               |
| Custom Quotas            | Define internal rules (e.g., limit EC2 per team/project)                     |
| Quota Increase Requests  | Request raises as team usage scales                                          |
| Cross-Account Scope      | Use **delegated admin** in orgs to manage quotas centrally                   |

---

## Final Thoughts

**AWS Service Quotas** is more than a billing or DevOps tool — it’s a **security architecture pillar**.

It:

- Ensures **predictable growth**  
- **Reduces surprise failures**  
- Sets **clear governance boundaries**  
- Helps **limit misconfigurations** and **cost overruns**