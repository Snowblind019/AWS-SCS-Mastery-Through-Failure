# AWS Secrets Manager

## What Is the Service

**AWS Secrets Manager** is the managed place to **store, rotate, retrieve, and audit secrets**—database passwords, API keys, OAuth tokens, SSH keys, webhooks, third-party creds—**without hardcoding** them into code, images, or templates. It encrypts secrets at rest with **AWS KMS**, returns them to callers over **TLS**, supports automatic rotation, and emits a full audit trail to **CloudTrail**.

**Why it matters**: most breaches have a boring root cause—**exposed or long-lived credentials**. Secrets Manager shrinks that blast radius by moving secrets out of code, shortening their lifetime (rotation), tightening access with **IAM** and resource policies, and giving you **provable evidence** of who accessed what, when. For **Snowy’s** teams, that’s the difference between *“we think the key was safe”* and *“here is the access log, rotation history, and the KMS key policy that enforced it.”*

---

## Cybersecurity and Real-World Analogy

### Security Analogy. Think of a Bank Vault:

- **Vault** = Secrets Manager (encrypted storage with policies).
- **Keys** = KMS CMK that encrypts everything inside the vault.
- **Teller window** = API that only hands out the secret to callers with the right identity + permissions.
- **Shift routine** = rotation that swaps combinations on a schedule (and makes the old code useless).
- **Cameras & logs** = CloudTrail entries for create/read/rotate/delete.

### Real-World Analogy:

You keep a password in a sealed envelope, locked in a safe. The safe has a logbook for every time someone checked the envelope, and you change the password and reseal a fresh envelope every few weeks. If the envelope ever goes missing, you know exactly when, and the leaked password is already stale.

---

## How It Works

### Core Model

| Concept         | What It Is                                                  | Why It Matters                                                                 |
|----------------|-------------------------------------------------------------|--------------------------------------------------------------------------------|
| **Secret**      | Named container holding a string or binary payload (JSON)   | `{"username":"winterday","password":"...","host":"..."}` is common.           |
| **KMS CMK**     | Key used to encrypt the secret at rest                      | BYOK/CMK policy gates decryption; separation of duties.                        |
| **Versions & Stages** | Every change creates a version. Labels like `AWSCURRENT`, `AWSPREVIOUS`, `AWSPENDING` point to versions. | Rotation swaps labels so apps don’t have to change code. |
| **Rotation**    | Lambda-based or managed function that updates the secret   | Automates least-lifetime; supports check/rollback.                            |
| **Resource policy** | JSON policy on the secret itself                        | Enables cross-account access, VPC endpoints, limits blast radius              |
| **Replication** | Multi-region replicas of a primary secret                  | Read near apps in other Regions; DR                                            |
| **Caching client** | SDK cache/sidecar to avoid hot-looping                   | Lower cost/latency; jittered refresh                                          |

## Create → Store → Get → Rotate (The Lifecycle)

### 1. Create the Secret

- Choose or create the **KMS CMK**
- Put the value (string JSON or binary)
- Tag the secret (`Service=Snowy-Checkout`, `Env=prod`), optional resource policy

### 2. Use the Secret

- App calls `GetSecretValue` (or the SDK helper / caching client)
- Secrets Manager decrypts with **KMS** and returns the value over **TLS**
- Prefer JSON payloads so you can add fields without schema pain

### 3. Rotate the Secret

- Enable rotation: schedule + Lambda or managed rotation
- Secrets Manager creates new version under `AWSPENDING`, calls your rotation function to:
  - create new creds
  - set them active
  - test them
  - label new version `AWSCURRENT` (old becomes `AWSPREVIOUS`)
- If tests fail: it rolls back labels, your app keeps working

### 4. Observe & Audit

- **CloudTrail** logs for: `GetSecretValue`, `PutSecretValue`, `RotateSecret`
- **CloudWatch** metrics/alarms on throttles, failures, unusual reads

## Rotation Patterns

| Target                                  | Rotation Option    | Notes                                                      |
|----------------------------------------|--------------------|------------------------------------------------------------|
| RDS/Aurora (MySQL, PostgreSQL, SQL)     | Managed            | One-click, best starting point. Handles create/test/set/finish |
| Redshift / DocDB                        | Managed (when supported) | Similar flow.                                       |
| Anything else (DynamoDB, API tokens...) | Custom Lambda       | Implement 4-step contract; store metadata in secret        |

## Design Tips

- Keep rotation functions **idempotent**, **re-entrant**, and observable (structured logs)
- Use **dual users** (old/new) for DBs to reduce cutover
- Refresh connection pools or app will use stale creds
- Time rotations to **quiet hours**, and add **post-rotate health checks**

## Access Control

- **IAM** policies control **who** can call `GetSecretValue`, `PutSecretValue`, `UpdateSecret`, `RotateSecret`
- **KMS** key policy controls **who can decrypt** (second lock)
- **Resource policy** can allow **cross-account reads**
- **VPC endpoints** keep traffic inside AWS
- **Condition keys**: `aws:PrincipalTag`, `secretsmanager:ResourceTag`, `kms:ViaService`

## Replication & Multi-Region

- Mark secrets for **replication**
- Primary writes & rotates; replicas are **read-only**
- Use for **DR** or **multi-region latency** reduction

## Formats & Retrieval

- JSON is best for most secrets
- Binary: certs, SSH keys, etc.
- Retrieve with **SDK**, **CLI**, or cached sidecar
- Use **exponential backoff**, jitter, and short **TTL**

---

## Pricing Model

| Cost Area        | Driver                                 | Guidance                                                                 |
|------------------|----------------------------------------|--------------------------------------------------------------------------|
| Secret storage   | Per secret per month (primary + replicas) | Consolidate fields into one JSON; use replicas only where needed        |
| API calls        | `GetSecretValue`, `PutSecretValue`      | Use caching client, batch if possible                                   |
| Rotation         | Lambda + target system ops             | Rotate on risk; keep code efficient                                     |
| KMS              | Usually included                        | Keep **CMK policy tight**; one per sensitivity tier                      |

> **Rule of thumb**: Cache secrets in memory (short TTL), rotate practically (e.g., 30d for DB creds)

---

## Parameter Store vs Secrets Manager

| Aspect            | Secrets Manager                         | SSM Parameter Store (`SecureString`)             |
|------------------|------------------------------------------|--------------------------------------------------|
| Primary use       | High-value secrets w/ rotation + audit  | Config, flags, low-risk secrets                  |
| Rotation          | Native (managed + custom)               | No native rotation                               |
| Replication       | Multi-region                            | No native multi-region                           |
| Resource policy   | Yes (cross-account)                     | Limited (via IAM)                                |
| Pricing           | Per secret + API                        | Cheaper per param                                |
| Best for          | DB creds, API tokens, 3P keys           | Non-rotated config or low-sensitivity secrets    |

## Operational & Security Best Practices

1. **Never hardcode** credentials. Cache with short TTL, retry on error.
2. **One secret = one unit** (prefer JSON blob per app/db)
3. **Tight IAM + KMS**. Separate readers from writers, use `kms:ViaService`
4. Use **VPC endpoints**
5. Rotate **on risk**, not superstition
6. **Stagger rotations** to avoid sync'd DB hits
7. Alert on anomalies (`CloudWatch`, `GetSecretValue` spikes)
8. **Tag + scope** by `Service`, `Team`, `Env`, `Sensitivity`
9. **DR**: replicate to failover region; test often
10. Pull secrets at **deploy** only if needed; prefer runtime
11. **Audit** access via `CloudTrail`; validate least privilege
12. **Hygiene**: delete old versions, avoid large plaintext blobs

---

## Real-Life Example (End-to-End, Winter Names)

**Scenario**: Blizzard-Checkout in `us-west-2` stores DB creds in `prod/blizzard-checkout/db`. Rotation every 30d with managed rotation.

### Setup

- JSON secret:  
  `{"username":"snowy_app","password":"...","host":"winterday-db.cluster-...","port":5432}`
- Encrypted with: `alias/prod-app-secrets`
- Resource policy: only Blizzard-Checkout task role can read
- **VPC endpoints** enabled

### Runtime

- App caches secret (TTL 5m)
- Cold start or refresh → `GetSecretValue('prod/blizzard-checkout/db')`

### Rotation Day

- Manager triggers rotation:
  - Create new cred (`AWSPENDING`)
  - Test
  - Promote to `AWSCURRENT`, demote old to `AWSPREVIOUS`
- App refreshes password automatically

### Audit & Guardrails

- `CloudTrail`: shows only `Snowy-OnCall` read outside deployment
- `CloudWatch`: alarms for:
  - Rotation failures
  - `GetSecretValue` spikes

### Multi-Region Read

- Replicates to `us-east-1` for read-only reporting job
- Reporting job uses replica ARN

**Outcome**: No creds in code. Rotation hands-off. Auditable per role/person.

---

### Quick IAM Sketch (Least-Privilege)

| Role                        | Allowed Actions                                       | Scope                                                      |
|-----------------------------|-------------------------------------------------------|------------------------------------------------------------|
| Blizzard-Checkout-TaskRole | `secretsmanager:GetSecretValue`, `DescribeSecret`     | `arn:aws:secretsmanager:...:secret:prod/blizzard-checkout/db*` |
| Winterday-Secrets-Admin    | `CreateSecret`, `PutSecretValue`, `RotateSecret`, `TagResource`, `UpdateSecret` | Secrets tagged `Team=Winterday`                      |
| Snowy-Rotation-Role        | `secretsmanager:*` (on this secret), `rds:*`, `kms:Decrypt` | Scoped to DB + KMS CMK                                     |

## Rotation Cadence (Examples)

| Secret Type                       | Suggested Cadence | Notes                                         |
|----------------------------------|-------------------|-----------------------------------------------|
| Public-internet API token        | 7–14 days         | Short-lived; consider provider-side scopes    |
| DB app user (prod)               | 30 days           | Staggered across services; dual-user         |
| Internal webhook/shared key      | 30–60 days        | Prefer HMAC + scoped perms                   |
| Machine-to-machine OAuth secret  | 30 days           | Consider **OIDC** + short-lived tokens       |

---

## Final Thoughts

Secrets Manager is the practical path to **credential hygiene**: encrypted storage with tight policies, automated rotation that doesn’t break apps, multi-region replicas, and full auditability.

Use:

- Short-lived IAM roles  
- Cached TTL-based patterns  
- `CloudWatch` / `CloudTrail` + alerts  
- Guardrails that page **Snowy-OnCall**

**Secrets become boring, reliable, and loud when broken.**