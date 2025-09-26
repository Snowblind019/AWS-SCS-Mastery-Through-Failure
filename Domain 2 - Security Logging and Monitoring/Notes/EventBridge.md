# Amazon EventBridge  

## What Is It

Amazon EventBridge is a serverless event bus that connects applications using events — signals that something has occurred. It's built to respond in near real time to changes across AWS, SaaS platforms, and even custom apps.  
But this isn't just for automation. In security, EventBridge is how you react to threats fast, trigger containment, notify teams, and orchestrate response workflows with precision.

Think of EventBridge as the central nervous system for your cloud. It doesn’t do the work — but it detects signals from all parts of your body (AWS) and fires off instructions to the right responders (Lambda, SNS, SQS, Step Functions, etc).

You can use it to:

- Trigger auto-remediation (isolate an EC2, revoke credentials)  
- Route security alerts to responders  
- Detect unusual behavior (ex: root login, policy changes)  
- Build audit and detection pipelines  
- Integrate third-party alerts from things like Datadog, PagerDuty, or Okta  

---

## Cybersecurity Analogy

Picture a modern SOC (Security Operations Center).  
The analysts aren’t manually watching every server. They’ve got alerts wired into an orchestration system.  

When a login happens from two countries in 10 minutes, or a user elevates their own privileges — the system reacts. Fast.

**EventBridge is like the SOAR system in a cloud-native SOC:**

- Something happens? It knows.  
- It evaluates the pattern.  
- It notifies the right person, takes action, or logs the evidence.  

And unlike CloudWatch Events (which it evolved from), EventBridge can consume events from thousands of sources, not just AWS services.

## Real-World Analogy

Imagine a fire alarm system in a smart building.

- The smoke detector (source) sends a signal  
- The system routes it to the fire department, unlocks exit doors, shuts off gas  
- Each action is independent, but event-triggered  
- The system is decentralized, but reacts immediately  

That’s EventBridge. It’s not the one fighting the fire — but it’s what starts the response.

---

## How It Works

EventBridge has 4 key components:

| Component    | Purpose                                                                 |
|--------------|-------------------------------------------------------------------------|
| Event Bus    | A pipeline that receives and routes events (default, partner, or custom)|
| Events       | JSON-formatted messages describing what happened                        |
| Rules        | Match events using pattern matching (NOT SQL; uses JSON match logic)    |
| Targets      | Where the matched event goes (Lambda, SNS, SQS, Step Functions, etc.)   |

---

### Event Flow Example

- A new IAM user is created via the console  
- AWS IAM sends an event to the default EventBridge bus  
- A rule matches: `"detail.eventName": ["CreateUser"]`  
- The rule routes the event to:  
  - Lambda function that sends a Slack alert  
  - SNS topic that emails the SecOps team  
  - Step Function that tags and logs the new user  

You didn’t write any polling loops. No cron jobs. No delay.  
Just pure event-driven response.

---

## Common Event Sources

### AWS Services

| Service       | Event Types (Examples)                              |
|---------------|-----------------------------------------------------|
| IAM           | CreateUser, AttachUserPolicy, UpdateAccessKey       |
| EC2           | StartInstances, StopInstances, ModifySecurityGroup  |
| S3            | PutObject, DeleteObject, LifecycleExpiration        |
| CloudTrail    | All logged actions                                  |
| GuardDuty     | New finding generated                               |
| Security Hub  | Finding imported, status changed                    |
| AWS Config    | Compliance state change                             |
| AWS Health    | Service outages, planned maintenance                |
| CodePipeline  | Pipeline started, failed, completed                 |

And also SaaS integrations:

- Datadog  
- Okta  
- Zendesk  
- PagerDuty  
- GitHub  
- Segment  

And your own **custom applications** can send events via `PutEvents` API.

---

## Security Use Cases

| Use Case                   | EventBridge Role                                                                |
| Auto-remediation           | Trigger Lambda to quarantine EC2 or revoke access                               |

| Real-time alerting         | Forward critical findings to Slack, email, or PagerDuty                         |
| Tagging non-compliant resources | Apply tags automatically when AWS Config marks a resource non-compliant   |

| Insider threat detection   | Alert if root account logs in, or if MFA is disabled                            |

| Policy change tracking     | Detect if someone edits Security Group, KMS policy, or CloudTrail settings      |

| Workflow automation        | Trigger a ticket in Jira or ServiceNow on critical GuardDuty finding            |

| Evidence collection        | Snapshot EBS, dump metadata, and store in S3 when a threat is detected          |


---

### Example Rule JSON (Match Event)

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{ "numeric": [">=", 7.0] }],
    "type": [{ "prefix": "UnauthorizedAccess" }]
  }
}
```

### This Rule Matches:

- All GuardDuty findings  
- With severity >= 7  
- That start with type `"UnauthorizedAccess"`  

**You can route this to:**

- Lambda for auto-tagging  
- SNS for SecOps notification  
- Step Function for IR workflow  

---

## Event Archive & Replay

One of the most underrated features of EventBridge is **event archiving**.

- You can **store past events** for later analysis or compliance  
- You can **replay** them into the bus to test new rules, workflows, or simulate past attacks  

This is huge for:

- IR playbook testing  
- Post-mortem analysis  
- Debugging automation pipelines  

**Example:**  
Replay all IAM changes from last 24 hours into a test bus with new remediation rules.

---

## Schema Registry

EventBridge can auto-detect the structure of incoming events, store them in a **schema registry**, and let you generate **code bindings** in Java, Python, or TypeScript.

This makes building event-driven apps more maintainable:

- You know what fields are in each event  
- You can validate structure and types  
- You can enforce versioning and contracts  

---

## Pricing Model

| Action                    | Cost                         |
|---------------------------|------------------------------|
| Ingesting custom events   | $1.00 per 1 million events   |
| Events from AWS services  | Free                         |
| Archive storage           | $0.10 per GB per month       |
| Replays                   | $0.10 per GB of replayed data|

> Most of your costs come from custom app events, replay usage, or high-volume custom buses.

---

## Real-Life Example

**Snowy’s IR team** wanted to reduce **MTTR** on GuardDuty findings.

They built an EventBridge rule:

- When a finding with `"type": "CryptoCurrency:EC2/BitcoinTool.B"` and `"severity" >= 7` was generated:  
  - Route to Lambda → isolate EC2 via security group update  
  - Tag instance `compromised=true`  
  - Forward finding to Security Hub  
  - Post alert in Slack  

Then, during **post-mortem**, they used **event replay** to simulate the attack again and test their updated workflow — no new data needed.

> This turned a **30-minute manual response** into a **10-second auto-containment**.

---

## Final Thoughts

**EventBridge is your cloud security reflex arc.**

It doesn’t analyze.  
It doesn’t decide.  
It doesn’t store logs.  

But it **moves**, and it moves fast — when you design it well.

**Security isn’t just about detection.**  
It’s about **response at the speed of the threat**.

If you're still manually watching logs and responding to emails, you're already behind.

Let EventBridge **listen, match, and fire.**  
You just tell it what to look for.  
**It’ll do the rest.**
