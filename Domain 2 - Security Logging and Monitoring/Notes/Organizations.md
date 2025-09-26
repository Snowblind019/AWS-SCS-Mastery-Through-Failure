# AWS Organizations  

---

## What Is The Service

AWS Organizations lets you **centrally manage and govern multiple AWS accounts** within a hierarchical structure, known as an *organization*. It’s a foundational service for enterprises that need:

- **Account separation** for billing, isolation, team autonomy, or blast-radius control  
- **Centralized policies** that apply across accounts (like security guardrails)  
- **Unified billing** and consolidated management  
- **Delegated administration** and cross-account operations  

It’s *not just a billing service* — it’s a **security and governance powerhouse**. Organizations is the **starting point** for any well-architected multi-account AWS environment.

---

## Cybersecurity and Real-World Analogy

**Cybersecurity Analogy:**  
Think of AWS Organizations like a **federated government system**.  
Each AWS account is like an individual state with its own governance (resources, users, applications), but the **organization root** is like the federal government. It:

- Sets the nationwide laws (guardrails via **SCPs**)  
- Manages finances (consolidated billing)  
- Appoints department leads (**delegated admins**) to oversee specific areas (like security or billing)  

You still have autonomy in individual accounts, but everything is **monitored, restricted, and reported centrally** — with fine-tuned control.

**Real-World Analogy:**  
Imagine a massive corporation with multiple departments: HR, Marketing, Engineering, Finance.  
Instead of giving all teams access to one giant building (monolithic AWS account), you give each team its **own building (account)** — but connected through a secure campus (organization).  

Now you can:

- Lock down all doors at once (**SCPs**)  
- Set a single power bill (**consolidated billing**)  
- Monitor cameras in every building (**CloudTrail org trails**)  
-  Give facilities staff (admins) access to just their own building  

And if one building burns down? The others aren’t affected. That’s the **power of account-level isolation**.

---

## Key Concepts in AWS Organizations

| **Concept**             | **Description**                                                             |
|-------------------------|-----------------------------------------------------------------------------|
| **Root**                | The top-level parent of all AWS accounts; the org’s central authority       |
| **Organizational Units**| Groupings of accounts for applying policies and managing structure          |
| **Member Accounts**     | Child accounts under the org, often for teams, workloads, or environments   |
| **Service Control Policies (SCPs)** | Guardrails applied at OU or account level to restrict permissions |
| **Delegated Admins**    | Non-root accounts assigned specific management roles (e.g., GuardDuty)      |
| **Consolidated Billing**| Single invoice and cost tracking across accounts                            |

---

## Why Use AWS Organizations

- **Security Isolation** – Each account is a security boundary. Compromise in one doesn’t affect others.  
- **Least Privilege by Design** – Avoid cramming unrelated workloads into one account.  
- **Centralized Governance** – Enforce SCPs to limit what accounts can do (even root users).  
- **Cost Control** – Centralized billing + detailed usage breakdowns per account/team/env.  
- **Scaling** – Delegate account ownership while maintaining top-level visibility and control.

---

## How It Works

1. **Create an Organization**  
   - One account becomes the **management account** (formerly "master account")  

2. **Add Accounts**  
   - Invite existing accounts or create new ones (account vending machine model)  

3. **Create OUs (Organizational Units)**  
   - Group accounts logically: Security, Prod, Dev, Sandbox  

4. **Attach SCPs (Service Control Policies)**  
   - Apply guardrails like:  
     - ✖️ No access to region `ap-south-1`  
     - ✖️ No use of EC2 Classic  

5. **Set Up Delegated Admins**  
   - Let certain accounts manage org-wide tools: GuardDuty, Security Hub, Access Analyzer  

6. **Use Org-Integrated AWS Services**  
   - CloudTrail (org-level trails)  
   - Security Hub, Macie, Inspector (findings aggregation)  
   - Tag policies and backup policies (enforce org-wide)

---

## Service Control Policies (SCPs) — Guardrails for Governance

SCPs are JSON documents that define the **maximum allowable permissions** for accounts or OUs.  

- Even if an **IAM policy allows** an action, SCPs can **override and block it**  
- They apply to **all users — including root**  
- SCPs **can’t grant permissions** — only restrict them

**SCP Example – Deny All Access to `us-east-1`:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUSEast1",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

SCPs apply to **all users, including root**, in the account.  
They **cannot grant permissions** — only restrict.

---

## Common OU Structure

| **OU**           | **Purpose**                                               |
|------------------|-----------------------------------------------------------|
| **Security**     | Centralized accounts for logging, audit, guardrails       |
| **Infrastructure**| Shared services like DNS, AD, CI/CD                      |
| **Production**   | Prod workloads (isolated accounts per app/team)           |
| **Staging**      | Pre-prod testing, similar structure to prod               |
| **Dev**          | Developer sandboxes                                       |

This structure supports:

- ✔️ Separation of concerns  
- ✔️ Tight access control  
- ✔️ Easier incident containment

---

## Pricing

AWS Organizations is **free to use** — including account creation, OUs, and SCPs.  
You **only pay** for the AWS services you use in each member account.

---

## Security Benefits

- Strong blast-radius containment  
- Root-account access can be tightly restricted via SCPs  
- Cross-account trust is **explicit and auditable**  
- Supports centralized audit and log archiving (e.g., to a **Security OU**)  
- Minimizes **privilege escalation vectors**

---

## Real-Life Example

Let’s say **Blizzard** is building out AWS for a **mid-size SaaS company**.  
They use **AWS Organizations** to:

- Spin up accounts like:  
  - `dev-blizzard-app`  
  - `prod-blizzard-app`  
  - `security-logs`  
  - `infra-shared`

- Place them in OUs like:  
  - `Dev`  
  - `Prod`  
  - `Security`

- Attach SCPs like:  
  - `"Deny S3 public buckets"`  
  - `"Disallow creation of Internet Gateways in Prod"`  
  - `"Block unused regions"`

Now each team has **autonomy within their account**, but Blizzard controls:

- ✔️ Boundaries  
- ✔️ Billing  
- ✔️ Access  
- ✔️ Logging  
- ✔️ Org-wide security services like **GuardDuty**, **Macie**, and **CloudTrail**

---

## Final Thoughts

**AWS Organizations** is one of the most important **foundation services** for long-term cloud maturity.  
It helps teams **grow safely**, **govern securely**, and **operate efficiently** — without central bottlenecks.

It’s not just about structure.  
It’s about:

- **Security architecture**  
- **Blast radius control**  
- **Cost governance**  
- **Scalable delegation**
