# AWS Secrets Manager

## What is the service

**Secrets Manager** is AWS’s dedicated tool for keeping sensitive data like **API keys**, **database credentials**, **OAuth tokens**, and **encryption keys** locked down, rotated, and programmatically retrievable in a secure and auditable way.

Without something like this in place, secrets tend to end up where they don’t belong:

- Hardcoded directly in source code  
- Sitting unencrypted in config files  
- Passed around over Slack, email, or sticky notes  

Eventually, it’s a breach just waiting to happen.

Secrets Manager exists to eliminate that risk. It:

- **Encrypts secrets at rest using KMS**
- **Rotates secrets automatically** with Lambda
- **Restricts access with IAM controls**
- **Audits usage with CloudTrail**
- **Lets apps pull secrets at runtime**, securely

If you're ever in a situation where you need to inject a password or token into Lambda, EC2, ECS, or a container — without hardcoding anything or exposing plaintext — this is the tool to use.

---

## Cybersecurity Analogy

Think of **Secrets Manager** as a **biometric vault for credentials**.

You’ve got a critical 12-digit PIN for a financial account. You don’t trust anyone with it. You don’t write it down. So you lock it in a vault that:

- Only opens for people with a specific **fingerprint (IAM role)**
- Keeps a **log of every access**
- Rotates the PIN **automatically every month**
- Allows **temporary access**, with conditions like time-of-day or user type

That’s exactly what Secrets Manager is — except instead of locking away a PIN, you’re locking down credentials and secrets your infrastructure depends on.

## Real-World Analogy

Let’s say you're running a restaurant, and the recipe for your signature sauce is strictly confidential.

- Only the **head chef** can see it  
- The recipe changes **once a month**  
- It needs to stay secure — no copying, no leaking

Secrets Manager would be the **secure cabinet in the back**:

- Holds the recipe in a **sealed envelope (encrypted)**
- Requires a **keycard (IAM permissions)** to open  
- **Logs every access**  
- **Automatically swaps in a new envelope** on a schedule

It's the same concept — just applied to your cloud environment.

---

## How It Works

### 1) Storing Secrets

Secrets are stored as **key-value pairs**, for example:

```txt
username: admin
password: 6$Jw0!qZ9
```

They’re **encrypted with KMS**, versioned so you can roll back if needed, and organized by name (e.g., `prod/mysql-db-secret`).

You can use AWS-managed KMS keys or bring your own CMK if you want tighter control.

### 2) Accessing Secrets

Applications can retrieve secrets using:

- **AWS SDKs**
- **Boto3**
- **Secrets Manager API**

Secrets are **decrypted only at the point of access**, and **IAM permissions** control who can get them.

There’s no need to hardcode anything — your app can securely pull what it needs when it needs it.

### 3) Automatic Rotation

You can enable **automatic rotation** for any secret.

Behind the scenes, this uses a **Lambda function** that:

- Generates a **new password/token**
- **Updates the underlying system** (e.g., a database)
- **Stores the new secret** back into Secrets Manager

You never even see the new value — it just happens silently, behind the scenes.

### 4) Auditing & Monitoring

Every time a secret is accessed, it’s logged in **CloudTrail**.

You can plug these logs into **CloudWatch Alarms**, **Security Hub**, or your SIEM.

You get visibility into:

- Which identity accessed a secret
- What time it happened
- Which secret was touched

## When to Use It (vs. other tools)

| **Need**                                                      | **Use This**           |
|---------------------------------------------------------------|------------------------|
| Store passwords, tokens, DB creds w/ rotation                 | **Secrets Manager**    |
| Store simple config values like `color=blue`                  | **SSM Parameter Store**|
| Need automatic rotation + audit trail                         | **Secrets Manager**    |
| Need hierarchical configs (tree-style)                        | **SSM Parameter Store**|
| Inject secrets into Lambda/ECS/EC2 **at runtime**             | **Secrets Manager**    |
| Replacing plaintext creds in code or Git                      | **Secrets Manager**    |

---

## Pricing Model

Secrets Manager isn’t free, but its pricing is pretty straightforward:

| **Resource**                   | **Price (2025 estimate)**              |
|-------------------------------|----------------------------------------|
| Secrets stored                | ~$0.40 per secret per month            |
| Rotation (optional)           | ~$0.05 per rotation                    |
| API calls                     | First 100,000/month free, then billed  |

A single “secret” can hold multiple fields (like username + password), and you’re only billed once per secret.

So for example:

- 10 secrets → $4/month  
- Rotated weekly → ~$2/month  
- Total → **~$6/month**

Not cheap at scale, but absolutely worth it when weighed against the cost of a leaked secret.

---

## Real-Life Example

You’ve got a **Lambda function** that needs to connect to a **MySQL RDS** database.

Here’s what you do:

1. Store the DB creds in Secrets Manager:
   ```json
   {
     "username": "admin",
     "password": "P@ssw0rd123!"
   }
   ```
2. Give the Lambda IAM role permission to read the secret.
3. In your Lambda code:
   ```python
   import boto3

   client = boto3.client('secretsmanager')
   secret = client.get_secret_value(SecretId='prod/mysql-db-secret')
   creds = json.loads(secret['SecretString'])
   ```

That’s it.

- No hardcoding  
- Password rotates automatically every 30 days  
- Every access is logged  
- Secret is never exposed in plaintext

This is how you handle sensitive data **the right way**.

---

## Final Thoughts

Secrets Manager solves one of the most common — and dangerous — security gaps in modern environments: **how do we securely manage credentials and secrets** without creating a mess of hardcoded values, plaintext files, or insecure workarounds?

It lets you:

- **Centralize secret storage**
- **Enforce IAM-based access**
- **Rotate secrets automatically**
- **Audit everything**

It’s clean, mature, secure, and built for cloud-native apps that need flexibility without sacrificing protection.

If you're building secure infrastructure, Secrets Manager isn't optional — it's foundational.

**Don’t trust humans with secrets. Trust the vault.**
