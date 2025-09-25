# Playbooks & Runbooks Design  

---

## What Are Playbooks and Runbooks (And Why They Matter)

In incident response, every second matters. But people forget. People panic. People make mistakes.  
That’s why great teams don’t rely on memory — they rely on well-designed playbooks and runbooks.  

- A **playbook** is the high-level strategy: *What should we do when X happens?*  
- A **runbook** is the tactical execution: *Step-by-step instructions to actually do it.*

Together, they bring clarity, speed, and consistency to incident handling.  
No guessing. No reinventing. Just execution.

---

## Cybersecurity and Real-World Analogy

Think of a fire drill in a high-rise building.

- The **playbook** is the policy that says:  
  “If fire is detected, evacuate floors top-down, call 911, activate sprinklers, and notify building security.”

- The **runbook** is the checklist by the exit:  
  “Pull lever → Call fire dept at 555-5555 → Evacuate floor via Stairwell A → Close all doors → Wait at Muster Point B.”

In an actual emergency, people don’t stop to think. They execute what they’ve drilled.  
In cybersecurity, it’s no different. Incidents are loud. Fast. Chaotic.  
**Good playbooks and runbooks remove decision fatigue — so responders can focus on solving the problem.**

---

## How They Work

### Playbooks = The Strategy

A playbook defines:
- The type of incident (e.g., “Phished User Credential,” “EC2 Malware Detected,” “Data Exfiltration via S3”)
- Severity levels and escalation thresholds
- Roles & responsibilities (who owns what)
- Communication plans (who to notify, when, and how)
- Key tools/systems involved (e.g., GuardDuty, CloudTrail, IAM, SSM, etc.)
- Containment objectives (quarantine user/resource, revoke access, block traffic)
- Dependencies (legal review, PR comms, etc.)
- Trigger conditions for activating the playbook

**Playbooks should answer:**
- What triggered this?  
- What’s the goal?  
- Who is involved?  
- What systems are impacted?

> Good playbooks are reusable across environments.

### Runbooks = The Execution

Runbooks are concrete, step-by-step documents.  
They assume someone is already in front of a terminal — now what?

**Example: Runbook for leaked AWS access keys**
- Identify the IAM user from leaked key (`aws sts get-caller-identity`)
- Rotate access keys immediately via CLI
- Revoke all active STS tokens
- Check CloudTrail for activity from the leaked key
- Enable GuardDuty on the account if not already active
- Quarantine related EC2 via SSM automation document
- Create snapshot of affected instance volumes
- Export logs to S3 for analysis
- Notify SecOps lead and incident commander
- Document everything in the ticket (with timestamps)

> Good runbooks are copy-paste ready.

## Characteristics of Great Playbooks & Runbooks

| Trait       | Playbooks                | Runbooks                        |
|-------------|---------------------------|----------------------------------|
| Audience    | IR managers, analysts     | Tier 1 responders, engineers     |
| Scope       | High-level, strategic     | Low-level, tactical              |
| Format      | Flowcharts, tables, SOPs  | Shell commands, AWS CLI, screenshots |
| Reusability | High across multiple scenarios | Often specific to systems/tools |
| Flexibility | Some judgment involved    | Very procedural                  |
| Focus       | Why and What              | How                              |

## Common Security Playbooks to Build

Start with your top threats:

**Credential Theft (Phished User, Leaked Keys)**
- MFA checks  
- Key rotation  
- IAM user disablement  
- STS token revocation  
- CloudTrail review  

**EC2 Compromise / Malware**
- Isolate instance via SSM  
- Snapshot disk  
- Dump memory (if possible)  
- GuardDuty + VPC flow logs inspection  

**S3 Bucket Data Exposure**
- Remove public access  
- Audit bucket policies  
- List accessed objects via CloudTrail  
- Notify DPO/legal if regulated data  

**Insider Threat or Privilege Misuse**
- IAM policy review  
- Admin action replay from CloudTrail  
- Interview or HR escalation  

**DDoS / Availability Attack**
- Engage WAF/DDoS protections  
- Traffic redirection / autoscaling  
- Partner escalation (e.g., AWS Shield Response Team)  

## Automation Tips

Where possible, bake **automation** into your runbooks.

Use:
- **SSM documents** to isolate instances, collect memory, patch  
- **Lambda functions** triggered by EventBridge for key rotation or user disablement  
- **Slack webhooks** for automatic alerting and team comms  
- **AWS Step Functions** for orchestrating multi-step containment workflows  

> Runbooks should link directly to these automations where available.

---

## Real-Life Example

**Playbook:** “Unauthorized API Usage from Leaked Key”  
**Trigger:** GuardDuty finding — `InstanceCredentialExfiltration`

**Runbook (simplified):**
- Identify IAM user from the key  
- Rotate keys  
- Invalidate all STS tokens  
- Review CloudTrail for actions made with key  
- Create ticket with timeline of activity  
- Run automated tagging of affected resources  
- Notify security lead via Slack  

> This process is fast. Repeatable. Reliable.  
> No need to fumble around with “what do we do again?”

---

## Final Thoughts

In a live incident, you don’t rise to the occasion — you fall to the level of your preparation.  
And nothing prepares you better than solid playbooks and airtight runbooks.

They turn chaos into flow.  
They empower even junior responders to act with confidence.  
They reduce mistakes, improve MTTR, and enforce consistency across shifts.

Don’t keep these in someone’s head.  
Don’t hide them in dusty wikis.

- **Version them.**  
- **Link them to tickets.**  
- **Keep them close.**

> Your future self will thank you when everything is on fire — and you can just follow the checklist.
