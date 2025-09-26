# AWS CloudFormation Drift Detection

---

## What Is The Service

**CloudFormation Drift Detection** is a built-in feature of AWS CloudFormation that identifies mismatches between what your template *says* your infrastructure should be and what it actually *is* in your AWS environment.

In simple terms: when you define your infrastructure as code (IaC) via a CloudFormation template, you’re saying, *“This is exactly how I want my resources to look and behave.”* But real-world changes happen:

- A developer edits a security group rule in the console
- A script modifies an IAM policy
- Someone disables versioning on a bucket manually

**Drift Detection** tells you when your stack's real-world state has “drifted” from its intended state — including:
- Configuration changes
- Deleted or missing resources
- Property mismatches

It’s essential for **cloud security**, **compliance**, **governance**, and **IaC hygiene**.

---

## Cybersecurity and Real-World Analogy

### **Cybersecurity Analogy**

Imagine you’re a **CISO**. You’ve defined that *port 22 (SSH)* should never be open.

But your firewall logs show someone manually opened it.

There’s now a **security drift** between your documented intent and actual implementation.

Drift Detection is like having an **automated compliance auditor** — one that constantly checks production against policy and flags unauthorized changes.

### **Real-World Analogy**

You're building a house using **blueprints**. The blueprint says, “3 windows on the north wall.”

But during construction, someone installs 4.

That’s *drift* — what got built doesn’t match the design.

**CloudFormation Drift Detection** is your **final walkthrough** before approving the structure. It ensures that what was declared is what was delivered.

---

## How It Works

Drift Detection is available at **two levels**:

- **Stack-level**: Scans all supported resources in the stack
- **Resource-level**: Scans a single named resource in a stack

It compares:
- **Your declared desired state** (the CloudFormation template)
- **The actual deployed state** in your AWS account

If any differences are found, the resource is flagged.

---

## Drift Status Types

| **Status**     | **Meaning**                                                                      |
|----------------|----------------------------------------------------------------------------------|
| `IN_SYNC`      | No differences — resource matches the template                                   |
| `MODIFIED`     | Resource exists but property values differ from the template                     |
| `DELETED`      | Resource is missing in the environment but still exists in the template          |
| `NOT_CHECKED`  | Drift detection hasn’t been run on this resource yet                             |

---

## Which Resources Are Supported?

Not everything can be drift-detected — but **many core services are supported**, including:

- EC2 Instances
- Security Groups
- IAM Roles and Policies
- S3 Buckets
- RDS Instances
- VPCs & Subnets
- Route Tables
- CloudWatch Alarms
- Lambda Functions
- ELBs (Elastic Load Balancers)

🟣 Some complex services, nested stacks, or resources with dynamic properties may be **unsupported**.

> **Rule of thumb**: If the resource has a clearly defined, static configuration — it’s likely supported.

---

## Example Use Case

You deploy a CloudFormation stack with:

- ✔️ An EC2 instance with a specific tag
- ✔️ An S3 bucket with versioning enabled
- ✔️ A Security Group allowing only port 80

Three weeks later:

- ✖️ A developer manually opens port 22
- ✖️ Versioning is turned off on the S3 bucket

You run **Drift Detection**, and it reports:

- `MODIFIED`: Security Group (port rules changed)
- `MODIFIED`: S3 bucket (versioning disabled)

You now have proof of drift — and can **take corrective action**.

---

## CLI Commands

### Detect drift on the full stack:
```bash
aws cloudformation detect-stack-drift --stack-name myAppStack
```

### Check the status of drift detection:
```bash
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id abc123...
```

### Detect drift on a single resource:
```bash
aws cloudformation detect-stack-resource-drift \
  --stack-name myAppStack \
  --logical-resource-id MySecurityGroup
```

---

## Security Relevance

- **Enforces IaC discipline** — discourages unauthorized manual edits
- **Detects security misconfigurations** — open CIDR ranges, disabled logs, altered IAM permissions
- **Compliance proof** — auditors can verify that deployed infra matches defined policy
- **Early warning system** — identify drift before it results in a misconfiguration or breach

---

## Limitations

| **Limitation**                           | **Explanation**                                                                 |
|------------------------------------------|----------------------------------------------------------------------------------|
| Not automatic                            | You must run drift detection manually or automate it with scripts                |
| No historical state tracking             | It shows that something drifted, but not *what the previous value was*           |
| Doesn’t detect resources created manually | If you create a resource outside of the stack, CloudFormation won’t detect it    |
| Not all resources supported              | Some services and dynamic resource types aren’t drift-aware yet                  |

---

## What You Can Do About Drift

Once drift is detected, you have options:

| **Action**                              | **Purpose**                                                                 |
|-----------------------------------------|------------------------------------------------------------------------------|
| Investigate and Fix                     | Manually compare current vs template and correct the mismatch               |
| Update Template                         | Update the template to reflect the new (drifted) state — *if intended*      |
| Redeploy the Stack                      | Recreates/reconfigures resources to match the template                      |
| Use Stack Policies                      | Prevent specific resources from being changed outside CloudFormation        |

---

## Best Practices

- ✔️ Automate drift checks using **Lambda + EventBridge** (cron)
- ✔️ Integrate drift detection into your **CI/CD pipelines**
- ✔️ Use **AWS Config** for tracking historical config changes
- ✔️ Enable **Stack Termination Protection**
- ✔️ Store and version your templates in **Git** for full change history

---

## Final Thoughts

**Drift Detection** may not be flashy — but it’s *foundational*.

If you're serious about:

- Infrastructure as Code
- Zero-trust infrastructure
- Regulatory compliance
- Secure cloud governance

...then you need to know *when reality starts to deviate from the blueprint*.

It’s like a **routine security scan** for your infrastructure. You might not notice symptoms, but underneath, something could be misaligned.

**Drift Detection** gives you that awareness — and helps you catch problems **before they become breaches.**
