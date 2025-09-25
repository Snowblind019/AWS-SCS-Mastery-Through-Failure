# AWS WAF (Web Application Firewall)

---

## What Is The Service

**AWS WAF** is a web application firewall that lets you monitor, filter, and control HTTP(S) traffic to your web applications based on customizable rules. It helps protect against common web exploits and vulnerabilities such as SQL injection, cross-site scripting (XSS), bad bots, and malicious IPs.

AWS WAF sits in front of your applications — not your EC2 instance directly, but services like:
- Amazon CloudFront (CDN)
- AWS Application Load Balancer (ALB)
- Amazon API Gateway
- AWS App Runner

It’s not a traditional network firewall — it operates at **Layer 7 (Application Layer)**, examining HTTP headers, URIs, query strings, cookies, and body contents.

**Why it matters:**  
Cloud-native apps and APIs are public-facing, often accessible from anywhere on the internet. That makes them targets. With WAF, you can block bad requests before they even reach your backend systems. It’s especially useful in zero-trust architectures and defense-in-depth strategies.

> WAF isn't meant to replace traditional controls like IAM or encryption — it's a first line of defense for application-level threats.

---

## Cybersecurity Analogy

Imagine your application is a nightclub.  
You don’t want just anyone walking in — especially not someone with a knife, fake ID, or a reputation for causing trouble.

**AWS WAF is your bouncer.** It:
- Checks every guest’s credentials (IP address, headers, geolocation)
- Blocks anyone on the blacklist (IP reputation list)
- Denies entrance if someone tries sneaking in a suspicious bag (SQL injection string in query)
- Enforces guest behavior rules (rate-limiting, geo-blocking, user-agent filtering)

You still have:
- Cameras inside (**CloudTrail**)
- A safe room (**VPC private subnets**)
- An emergency plan (**Security Hub**)  
But WAF is your **front-line filtering mechanism**.

## Real-World Analogy

Think of WAF like a **spam filter for your web app**.  
Before the email lands in your inbox, it gets scanned for:
- Known bad domains
- Suspicious patterns
- Rule violations

Same with AWS WAF — it filters malicious or unwanted web traffic before it reaches your **ALB**, **API Gateway**, or **CloudFront distribution**.

---

## How AWS WAF Works

AWS WAF is made up of **Web ACLs (Access Control Lists)** and rules.  
You associate a Web ACL with a resource (e.g., CloudFront distro or ALB), and it processes every incoming request through the rules in the ACL.

### Web ACL

A container for rules. It defines what happens when a rule matches (`ALLOW`, `BLOCK`, or `COUNT`).

Web ACLs can include:
- **Managed Rules** (prebuilt by AWS or AWS partners)
- **Custom Rules** (your own logic)
- **Rule Groups** (collections of rules)

### Rule Types

| Rule Type       | Purpose                                                       |
|------------------|---------------------------------------------------------------|
| IP Set Match     | Allow/block specific IPs or CIDR ranges                      |
| Geo Match        | Allow/block traffic by country                               |
| Regex Match      | Match against URI, headers, or body using regular expressions|
| Size Constraint  | Match on length of request component (e.g., URI, body)       |
| Rate-Based Rules | Automatically block IPs that exceed a set request threshold  |
| SQL Injection/XSS| Detect and block common exploit attempts                     |
| Label Match      | Use labels (tags) to build conditional logic across rules    |

**Rule Priority:**  
WAF evaluates rules in order. **First match determines the action** (unless using `COUNT` mode).

### Actions

- `ALLOW` — lets the request through  
- `BLOCK` — rejects the request  
- `COUNT` — tracks matching requests without blocking

### Visibility
- Matches can be sent to **CloudWatch Logs**
- Trends and dashboards can be viewed using **AWS WAF metrics**

---

## Managed Rule Groups

AWS and Marketplace vendors offer predefined rule groups you can plug in:

- `AWSManagedRulesCommonRuleSet`  
- `AWSManagedRulesSQLiRuleSet`  
- `AWSManagedRulesKnownBadInputsRuleSet`  
- `AWSManagedRulesAmazonIpReputationList`  

These provide out-of-the-box protection for common threats, with **automatic updates** maintained by AWS.

> You can mix **managed rules** with your **custom rules**.

---

## Integration Points

| Service          | Role with AWS WAF                                     |
|------------------|--------------------------------------------------------|
| CloudFront       | Most common entry point. Protect global CDN traffic   |
| ALB              | Protect apps hosted behind an Application Load Balancer |
| API Gateway      | Control which clients can hit your APIs               |
| App Runner       | Serverless app hosting — can be protected with WAF    |
| CloudWatch       | Logs and metrics for request counts and blocked requests |
| AWS Firewall Manager | Centralized WAF policy management across accounts |
| AWS Shield Advanced | Integrated with WAF for DDoS protection            |

---

## Pricing Models

AWS WAF is priced in **three parts**:

| Component | Cost Model                              |
|-----------|------------------------------------------|
| Web ACL   | Flat fee per ACL (monthly)               |
| Rules     | Fee per rule or rule group per ACL       |
| Requests  | Fee per million requests processed       |

> If you're using **Managed Rules**, there may be **additional fees** per rule group, depending on the vendor.

> **Use WAF intelligently** to avoid ballooning costs — keep rules concise and use `COUNT` mode while testing.

---

## Use Cases

- Block traffic from high-risk countries  
- Limit abuse from scraping bots  
- Protect APIs from injection attacks  
- Rate-limit sign-up attempts or login attempts  
- Filter payloads based on regex (e.g., suspicious headers)  
- Meet compliance for web-facing applications (PCI DSS, HIPAA, etc.)

---

## Best Practices

- Use `COUNT` mode to monitor a rule before enforcing it  
- Pair with **CloudWatch Dashboards** for visibility  
- Enable **AWS WAF Logging** (via Kinesis Firehose to S3) for long-term retention  
- Apply **Managed Rules** for baseline protection, then layer on **custom rules**  
- Tune rules based on real traffic — avoid over-blocking  
- Use **Labels and conditional logic** for complex scenarios  
- Integrate with **Firewall Manager** for org-wide policy management

---

## Final Thoughts

WAF isn’t about stopping everything — it’s about **raising the bar for attackers** and **filtering out garbage traffic** before it costs you performance, money, or security.

As a Cloud Security Architect, **AWS WAF** gives you one of the most direct levers to control the behavior of internet traffic before it hits your infrastructure.

But like any tool — its power is in how well you **design your rules**, **test them**, and **integrate them** into a larger detection and response workflow.

> It’s not enough to just turn on WAF — you need to know what you're defending, what you're filtering, and how it fits into your threat model.

**Done right, WAF becomes your scalable, automated, always-on application gatekeeper.**

