# AWS IAM

## What Is The Service

AWS Identity and Access Management (**IAM**) is the control plane for **who** (principal) can do **what** (actions) on which resources, **under which conditions**, and for **how long**. It issues and evaluates permissions, mints **short-lived credentials** via **STS**, and enforces **least privilege** across every AWS API call. If **CloudWatch** tells you *what happened*, and **Config** tells you *what changed*, **IAM** is the thing that decides *whether an action is even allowed* in the first place.

Why it matters: breaches often start with **credential misuse** or **over-permissioned identities**. IAM shrinks blast radius with **roles** instead of long-lived keys, **conditions** instead of blanket access, and **explicit denials** that stop bad paths cold. Done right, IAM turns *“I hope no one can delete the production KMS key”* into *“that path is impossible by design.”*

---

## Cybersecurity And Real-World Analogy

**Security Analogy.** Think of a secured campus:

- **Badges** = identities (users, roles, federated sessions)
- **Doors & rooms** = AWS resources
- **Door schedules & guards** = policies with **conditions** (time, network, device posture)
- **Temporary visitor passes** = **STS** sessions (scoped, short-lived)
- **Master rules posted at the gate** = **explicit denies / SCP** guardrails

**Real-World Analogy.** Hotels with **keycards**:

- You don’t copy the master key for every guest; you issue a **short-lived key** for their room and dates *(assume role)*.
- The key stops working at checkout *(expiration)*.
- Some doors never open with guest keys *(explicit deny / SCP)*.
- Staff get keys that work **on their wing only** *(least privilege + ABAC/RBAC)*.

---

## How It Works

### Principals And Identities

- **Principal**: the caller of an API (user, role session, service principal, federated identity).
- **Identities** (things you attach identity-based policies to):
  - ○ **Users**: long-term identities (avoid access keys where possible).
  - ○ **Roles**: assumable identities that issue **temporary credentials via STS** (preferred).
  - ○ **Groups**: collections of users for shared permissions (**RBAC-style**).

## Credentials

| Type                 | Lifespan        | Use case                         | Notes                                                      |
|----------------------|------------------|----------------------------------|------------------------------------------------------------|
| Password (console)   | Until rotated    | Human console access             | Enforce MFA; avoid root for daily ops.                     |
| Access keys (AKIA…)  | Long-lived (bad) | Legacy programmatic access       | Replace with roles; if used, rotate + scope tightly.       |
| STS temp creds       | Minutes–hours    | Apps, CI/CD, cross-account       | Best practice; auto-expire; scoped.                        |
| Web identity/OIDC    | Minutes–hours    | EKS IRSA, GitHub Actions, Cognito| No static secrets; issuer + audience bound.                |
| SAML federation      | Session          | Enterprise SSO (IdP → AWS)       | Centralized identity; least privilege per role.            |
| Roles Anywhere       | Cert-based       | Servers/on-prem devices          | X.509 trust to assume roles without keys.                  |

## Policy Types

- **Identity-based policies** (attached to user/group/role).
- **Resource-based policies** (attached to the resource: S3 bucket policy, KMS key policy, SQS queue policy, Lambda permission). Enable **cross-account** access without touching the caller.
- **Permissions boundaries** (upper limit for an identity’s effective permissions).
- **Session policies** (additional inline limits attached to an STS session).
- **Service-control policies (SCPs)** (Organizations guardrails—not IAM, but evaluated before IAM; they set the outer fence for accounts).

**Policy doc shape**:  
`Version`, array of `Statement` each with `Sid?`, `Effect` (**Allow** | **Deny**), `Action` / `NotAction`, `Resource` / `NotResource`, and optional `Condition`.

## The Evaluation Logic

1. **SCPs / boundaries / service context** set the outer *“possible”* space.  
2. Collect all relevant policies (identity-based, resource-based, session policies).  
3. If any statement produces an **explicit Deny**, the request fails. *(Deny wins.)*  
4. Else, if at least one statement **Allows** (and conditions pass), the request **succeeds**.  
5. Otherwise, **implicit deny**.

**Short version**: Deny beats Allow; no Allow = Deny.

## Conditions And ABAC (Attribute-Based Access Control)

- **Conditions** let you express *when* an Allow applies: IP ranges, VPC endpoints, MFA present, time of day, encryption in transit, tags on resources and principals, etc.
- **ABAC**: design policies around **tags** (e.g., `Service=Blizzard-Checkout`, `Env=prod`).  
  One policy grants “operate on resources where `ResourceTag:Service == PrincipalTag:Service`.”  
  Scale permissions by attributes, not ever-growing lists.

## Trust Policies vs Permissions Policies

- **Trust policy** (on a role): *who is allowed to assume this role?* (e.g., `sts:AssumeRole`, `sts:AssumeRoleWithWebIdentity`, `sts:TagSession`).  
- **Permissions policy** (on the role): *what can the role do once assumed?*

You need **both**: someone must be trusted to assume, and the assumed session must be allowed to act.

## Cross-Account Access

- **Target account** attaches a resource policy or trusts a **caller role** in the source account (`sts:AssumeRole`).
- **Caller** assumes `arn:aws:iam::<target>:role/Winterday-ReadOnly` → receives temp **creds** → uses them on target resources.
- Optionally add **session policies** to scope the session further.

## Federation (SAML / OIDC)

- **Enterprise IdP** (Okta, Azure AD, etc.) issues assertions to AWS for **SAML** roles; users land in console or get temp **creds**.
- **OIDC** for workloads: **EKS IRSA** maps pod service accounts to roles; GitHub Actions **OIDC** maps **repo** workflows to roles — *no secrets*.

---

## Handy Comparisons

### Identity vs Resource vs Boundary vs Session vs SCP

| Type                 | Attached to             | Grants or Limits? | Cross-account?     | Typical use                                                 |
|----------------------|--------------------------|--------------------|---------------------|--------------------------------------------------------------|
| Identity policy       | User/Group/Role          | Grants             | Indirect (caller’s account) | Day-to-day permissions for a principal              |
| Resource policy       | Resource (S3/ KMS/SQS…)  | Grants             | Yes                 | Share resource; VPC-endpoint-only access; allow services     |
| Permissions boundary  | User/Role                | Limits             | N/A                 | Cap what builders can give themselves                        |
| Session policy        | STS session              | Limits             | N/A                 | Further restrict a powerful role per use                    |
| SCP                   | Account/OU (Orgs)        | Limits             | Org-wide            | Global guardrails (“no `kms:ScheduleKeyDeletion` in prod”)  |

## Common Condition Keys

| Condition            | Key                                       | Example                  |
|----------------------|--------------------------------------------|--------------------------|
| MFA required         | aws:MultiFactorAuthPresent                 | true                     |
| IP range             | aws:SourceIp                               | 203.0.113.0/24           |
| VPC endpoint only    | aws:sourceVpce                             | vpce-0123…               |
| TLS required         | aws:SecureTransport                        | true                     |
| Tag match (ABAC)     | aws:PrincipalTag/Service vs aws:ResourceTag/Service | Equal            |
| Time window          | aws:CurrentTime                            | 2025-09-24T00:00:00Z…    |
| Encryption enforced  | s3:x-amz-server-side-encryption            | aws:kms                  |

---

## Pricing Model

IAM itself is **free**.  
You pay nothing to create users/roles/policies. Costs show up **indirectly**: **STS calls**, **Access Analyzer findings storage**, **federation infra (IdP)**, and whatever the principals do.  

The big “cost” is **operational risk** — **over-permissioning** is expensive when something goes wrong.

---

## Operational & Security Best Practices

1. Prefer **roles** over long-lived keys. Humans federate (**SAML/Identity Center**); workloads assume roles (**IRSA/OIDC/STS**).
2. **Short sessions, scoped permissions.** Keep session duration reasonable; add session policies when handing powerful roles to automation.
3. **MFA everywhere that matters.** Console, break-glass roles, production change roles.
4. **ABAC for scale, RBAC for clarity.** Start with groups (**RBAC**), evolve to **tags** for horizontal scale (**ABAC**).
5. **Explicit denies for sharp edges.** Deny `kms:ScheduleKeyDeletion`, `iam:PassRole` except approved roles, dangerous EC2 actions in prod, etc.
6. **Permission boundaries for builder roles.** Let engineers create roles, capped by boundaries you define.
7. **Guardrail with SCPs.** Block forbidden regions, root usage, wildcard administrator policies, and data-destructive actions in prod.
8. **Access Analyzer + policy validation.** Detect unintended public/ cross-account access; lint policies for over-broad `Action`/`Resource`.
9. **Credential hygiene.** Prohibit access keys unless justified; rotate; monitor with **Credential Report** and **Access Advisor/Last Accessed**.
10. **Network conditions.** Enforce **VPC endpoints**, **TLS**, and **source IPs** where applicable.
11. **Log and alert.** **CloudTrail** + **CloudWatch Alarms** on sensitive IAM events (`CreateUser`, `PutUserPolicy`, `AttachRolePolicy`, `PassRole`, `UpdateAssumeRolePolicy`).
12. **Break-glass done right.** One tightly-scoped emergency role with short session, MFA, strong monitoring, and a loud paper trail.

---

## Real-Life Example

**Scenario.** Blizzard-CI in Account A needs to deploy to production in Account B without storing long-lived keys.

### 1. Target role (Account B).
- Create **Blizzard-Prod-Deployer** role with permissions to update ECS services, read from ECR, and read specific SSM params.
- Trust policy allows `sts:AssumeRole` from Account A’s CI role and requires a session policy that denies destructive actions.
- Add condition: `aws:PrincipalTag/Service = "Blizzard-CI"` and `aws:SourceIdentity = "main"` (branch binding).

### 2. CI role (Account A).
- Role **Snowy-Blizzard-CI** with no standing prod perms. Pipeline step calls:

```python
sts:AssumeRole(
  RoleArn=arn:aws:iam::<AccountB>:role/Blizzard-Prod-Deployer,
  DurationSeconds=1800,
  Policy=<session-policy-denying-dangerous-actions>,
  Tags={Service=Blizzard-CI, Team=Winterday}
)
```

### 3. Guardrails.
- **SCP** in Account B denies `*Delete*` and `kms:ScheduleKeyDeletion` in prod.
- **Permissions boundary** on any human-created role in B caps privileges to a safe envelope.
- **CloudTrail + alarms** on `PassRole`, `UpdateAssumeRolePolicy`, `AttachRolePolicy`.

### 4. Outcome.
- Deploys run with **short-lived, scoped credentials**.
- If the CI role is compromised, the **session policy**, **SCP**, and **explicit denies** prevent data-destructive calls.
- Full audit trail ties each deploy to a branch/tag and a human approver.

## Reference Snippets

### 1) Minimal ABAC Policy (Operate only on matching Service tag)
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "ABACServiceMatch",
    "Effect": "Allow",
    "Action": ["ec2:*","logs:*","s3:*"],
    "Resource": "*",
    "Condition": {
      "StringEquals": {
        "aws:ResourceTag/Service": "${aws:PrincipalTag/Service}"
      }
    }
  }]
}
```

### 2) Require MFA For Sensitive Actions
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "RequireMFAForSensitive",
    "Effect": "Deny",
    "Action": ["iam:*","kms:*","ec2:TerminateInstances"],
    "Resource": "*",
    "Condition": {"BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}}
  }]
}
```

### 3) VPC-Endpoint-Only S3 Access
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "S3ViaVPCEOnly",
    "Effect": "Deny",
    "Action": "s3:*",
    "Resource": "*",
    "Condition": {"StringNotEquals": {"aws:sourceVpce": "vpce-0123456789abcdef0"}}
  }]
}
```

### 4) Permission Boundary
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BuilderEnvelope",
      "Effect": "Allow",
      "Action": ["ec2:*","logs:*","ssm:*","s3:*"],
      "Resource": "*",
      "Condition": {"StringEquals": {"aws:RequestedRegion": ["us-west-2"]}}
    },
    {
      "Sid": "NoDangerous",
      "Effect": "Deny",
      "Action": ["kms:ScheduleKeyDeletion","iam:*","ec2:Delete*","s3:DeleteBucket","s3:PutBucketPolicy"],
      "Resource": "*"
    }
  ]
}
```

---

## Final Thoughts

**IAM** is where you make **security a property of design** instead of an afterthought.  
Prefer **assumable roles** and **short-lived sessions**; shape access with **tags and conditions**; place **explicit denies** and **SCP guardrails** on the sharp edges; and validate continuously with **Access Analyzer** and logs.