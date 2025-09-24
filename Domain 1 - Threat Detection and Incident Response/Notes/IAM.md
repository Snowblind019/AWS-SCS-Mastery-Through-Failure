# AWS Identity and Access Management (IAM)

## What Is the Service

**IAM** is the **gatekeeper** of AWS. It’s how you control **who can do what, where, and under what conditions**.

Without IAM, your cloud environment would be wide open — anyone could spin up EC2s, delete S3 buckets, or modify security groups. IAM makes sure that **only the right identities get access, and only to the things they’re supposed to touch**.

IAM doesn’t run in a specific region — it’s global. And it sits at the very core of every AWS account, whether you're running a single dev box or managing a hundred-account org.

**Security starts here.**

---

## Cybersecurity Analogy

Imagine your AWS account is a secure facility.

**IAM is the security checkpoint at the front gate**:

- Verifies your **identity** (who are you?)
- Checks your **badge** (what are you allowed to do?)
- Confirms **time-based conditions** (are you allowed in now?)
- Enforces **extra rules** (do you need gloves, MFA, a chaperone?)

IAM policies are like color-coded access wristbands:

- **Red = full access** to every room in the building (admin).
- **Blue = read-only access** to the billing room.
- **Green = access to Room 302 only to press a single button (`StartInstances`)**.

And every time you touch a door (API), IAM checks your wristband. No exceptions.

## Real-World Analogy

Think of a hospital.

- **Doctors** can access sensitive patient records.
- **Nurses** can update charts but not see full records.
- **Janitors** can get into supply rooms but not the OR.
- **Visitors** have to wear badges and stay in public areas.
- **Paramedics** get **temporary emergency access** under strict conditions.

IAM is the system that enforces all of this — programmatically. It’s not personal. It’s **policy-based control**.

---

## How It Works

IAM boils down to:

- **Identities** (who's doing it)
- **Policies** (what they can do)
- **Resources** (what they’re doing it to)

### IAM Identities (Who)

- **Users** — actual humans with long-term credentials (avoid if possible)
- **Groups** — collections of users with shared policies
- **Roles** — short-lived, assumable identities (best practice for apps, automation, and services)
- **Federated Identities** — external users from SSO, Google Workspace, AD, etc.

### Resources (What)

Anything in AWS:
- EC2 instances
- S3 buckets
- Lambda functions
- RDS databases
- Anything with an ARN

### Policies (How and Under What Conditions)

Policies are **JSON documents** with rules like:

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::mybucket/*",
  "Condition": {
    "Bool": {
      "aws:MultiFactorAuthPresent": "true"
    }
  }
}
```

You define:
- **Effect** — Allow or Deny
- **Action** — What’s permitted (e.g., `ec2:StartInstances`)
- **Resource** — What it applies to
- **Condition** — When/how it's allowed (e.g., MFA required)

---

## Types of Policies

| Policy Type             | Purpose                                                                 |
|--------------------------|------------------------------------------------------------------------|
| **Identity-based**       | Attached to users, groups, or roles                                     |
| **Resource-based**       | Attached to S3, Lambda, KMS, etc.                                       |
| **Permissions boundaries** | Cap the **maximum** permissions a role or user can have                |
| **Service Control Policies (SCPs)** | Org-level guardrails across accounts                          |
| **Session policies**     | Applied temporarily via `sts:AssumeRole` sessions                       |

Each plays a specific role. Know when to use which.

---

## Built-In Security Features

IAM isn't just about access — it's also about hardening and hygiene.

- **MFA enforcement** for critical users and roles
- **Password policies** (length, rotation, complexity)
- **Access Advisor** — tells you what services a user actually used
- **Credential Reports** — audit access key usage and last logins
- **Least Privilege Enforcement** — only grant what’s needed, nothing more
- **Key rotation** — automatically or manually rotate secrets and access keys

It’s on you to implement these — but IAM gives you the tools.

## Temporary Credentials (The Right Way to Access AWS)

Avoid long-term IAM user credentials when possible.

Use **roles with temporary credentials** via:

- **Federated logins** (SSO or identity providers)
- **EC2 instance profiles**
- **Lambda execution roles**
- **`sts:AssumeRole` across accounts or automation pipelines**

Short-lived credentials mean **less exposure** and **better security posture**.

## IAM vs SSO vs STS

| Service       | Role                                                                  |
|---------------|------------------------------------------------------------------------|
| **IAM**       | Core service to define **who can access what**                        |
| **STS**       | Issues **temporary tokens** for assumed roles                         |
| **AWS SSO**   | Connects external identity providers and manages **centralized login** |

They work together — not in isolation.

## Pricing

**IAM is free.**  
You’re only charged for the services your users/roles interact with (e.g., EC2, S3).

---

## Real-Life Example

You’re securing access for a small team.

- **Dev Team** needs EC2 control (start/stop only)
- **Finance** needs to download billing reports
- **Security** needs full read-only across the account
- **Terraform** needs infra deployment access — nothing more

You:
1. Create IAM groups: `Developers`, `Finance`, `SecurityAuditors`
2. Write a policy for EC2 access:

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:StartInstances",
    "ec2:StopInstances"
  ],
  "Resource": "*"
}
```

3. Attach it to the `Developers` group
4. Add MFA requirement
5. Create a **role for Terraform** with least-privilege permissions
6. Enable **Access Analyzer** to audit public access

Now access is managed, scoped, and controlled — and you can prove it.

---

## Final Thoughts

**IAM is the backbone of AWS security.**  
It controls everything else — from your Lambda execution context to who can log in and who can delete prod resources.

It’s free. It’s powerful. And it’s easy to get wrong.

Master IAM and you master the **who, what, where, and how** of everything happening inside AWS.

IAM doesn’t just protect your cloud — it defines the **boundary between trust and risk**.
