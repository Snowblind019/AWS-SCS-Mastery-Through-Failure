# AWS Detective

## What Is The Service

Amazon Detective is the place you go **after an alert** to answer the questions that actually matter: *what really happened, how far did it spread, who touched what, from where, and what should we do next?* It automatically ingests **CloudTrail**, **VPC Flow Logs**, **EKS audit logs**, and **GuardDuty** findings, then builds a **behavior graph**—a linked dataset of principals, resources, **IPs**, user agents, **geos**, and timelines you can pivot through fast.  
The result: you click into an entity profile (an **IAM** role, an EC2 instance, an IP, a user), see context over time, and either verify a threat or disprove it with evidence.  
That’s the difference between *“we think it’s bad”* and *“it was bad, here’s the blast radius and the exact API calls.”*

---

## Cybersecurity And Real-World Analogy

### Security analogy

Picture **Snowy** walking onto the SOC floor after **Blizzard-OnCall** gets paged. **GuardDuty** already raised a finding, but the team needs the story, not just the title.  
**Detective** is the investigator’s board: strings between pins, timestamped photos, phone records, door swipes.  
You click the principal, and the board re-arranges: **new geolocations**, **new user agents**, **new ASOs**, **related findings**, and **activity sequences**.  
You can tell if this is a noisy spike or a real intrusion creeping across roles and **subnets**.

### Real-world analogy

Think of an aircraft incident review.

- **CloudWatch** is the cockpit gauges during the flight.  
- **Config** is the maintenance log.  
- **Detective** is the **post-flight investigation kit**—the flight data recorder and a replay tool that shows **who did what, when, and against which systems**, with the ability to pause on any second and zoom into a specific control input.

---

## How It Works

### The Behavior Graph

Detective **collects and links** data from your accounts into a **graph**: nodes (identities, **IPs**, instances, pods, sessions) and edges (API calls, network connections, relationships).  
It applies **ML**, stats, and graph analysis so you can pivot:

- from a finding  
- → to a role  
- → to its API calls  
- → to a new IP and **geo**  
- → to an EC2 instance it touched  
- → to other findings that share those elements.  

This is **investigation at click speed** rather than grep speed.

### Data Sources

- **AWS CloudTrail** – management & data events (who called what).  
- **Amazon VPC Flow Logs** – network conversations (who talked to whom).  
- **Amazon EKS audit logs** – Kubernetes control-plane events (who did what in the cluster).  
- **Amazon GuardDuty** – findings feed that kicks off many investigations.  

Detective keeps **up to a year** of aggregated investigation data (not your raw logs) for fast pivots and entity profiles.

## Entity Profiles, Timelines, And Finding Groups

- **Entity profiles.** One page per thing you care about (**IAM** user/role, EC2, IP, account).  
  You see **activity volume charts**, **unusual geographies**, **new user agents**, and **related findings**—handholds for hypothesis testing (*“was this role ever used from that country before?”*).

- **Timelines.** **Scrollable, filterable** windows around the suspicious period to line up API calls, network flows, and cluster events as a single sequence.

- **Finding groups.** Detective **clusters related findings** that likely describe one incident (one campaign, many symptoms).  
  This removes the “whack-a-finding” problem and gives you a bundled case to review and close.

## Detective Investigation

**Detective Investigation** adds **prebuilt analyses** for common threat patterns so you can ask richer questions with fewer clicks:

- impossible travel  
- suspicious **IAM** activity  
- flagged **IP** addresses  
- finding-group triage  

It runs **ML + threat intel** over your behavior graph to highlight **IOCs** and anomalies tied to a principal or finding.

> Practically, this means **Snowy** can open an **IAM** role and immediately see  
> *“this role started from a new geo + new ASO + burst in API categories”*  
> with links to all evidence.

## Admin Model And Org Scale

- **Administrator & members.** You pick a **Detective administrator** account; other accounts become **members** whose data feeds the same graph.  
  This gives you **one pane** to investigate across the **Winterday** organization.

- **Usage & cost view.** The admin can view **ingested volume by source** and **by member account**, and see a **30-day cost projection** for the whole graph.  
  Useful for **chargeback** and early cost control.

- **GuardDuty integration.** Turn on **Detective + GuardDuty** and they **integrate automatically**;  
  **GuardDuty findings** start flowing into the graph immediately.

---

## Pricing Model

You pay **per GB ingested** into the behavior graph, **per account/Region**, with tiered pricing.  
There’s **no extra charge** for the investigation UI or for storing the aggregated investigation data (**Detective retains up to 12 months**).

> Current tiers (per GB) start at $2.00 for the first 1,000 GB/account/Region/month, then drop across higher tiers.

**Data sources counted:** CloudTrail, VPC Flow Logs, EKS audit logs, GuardDuty findings (and certain findings ingested via Security Hub).

| Cost driver     | What counts                                    | Tips                                                                 |
|------------------|------------------------------------------------|----------------------------------------------------------------------|
| Ingested GB      | CloudTrail, VPC Flow, EKS audit, GuardDuty findings | Control upstream volume; disable noisy sources you don’t need in non-prod. |
| Accounts/Regions | Charged per account/Region                    | Centralize Regions; limit Regions in use for prod.                  |
| Retention        | 12 months (aggregated) included               | You don’t pay extra for this in Detective; raw logs retention lives in their native services. |

> **Rule of thumb:** keep Detective on for prod and sensitive **dev**, and shape cost by managing **upstream log volumes** and **Regions**.

## Operational And Investigation Workflow

1. **Start in the finding.**  
   Open the **GuardDuty** finding; hit *“Investigate in Detective.”* The context page loads with the principal, IP, or instance already centered.

2. **Scan the indicators.**  
   Look for **new geos, new ASOs, new user agents, and activity spikes** in the timeline around the alert. These are quick reality checks for compromise.

3. **Pivot deliberately.**  
   From the role → its recent API calls → the **IPs** and **geos** → to the EC2 instances touched → to related findings in the finding group.  
   This draws a clean blast radius.

4. **Confirm or kill the hypothesis.**  
   If the signals line up (e.g., *impossible travel + new ASO + unusual API categories*), escalate;  
   if not, document why it’s benign and close with evidence.

5. **Feed remediation.**  
   For confirmed incidents, hand off to **EventBridge/SSM runbooks** (disable keys, detach policies, block **IPs**, quarantine instances).

6. **Export + lessons.**  
   Screenshot key visuals, capture the sequence, and update **Blizzard-Runbooks** or **Winterday-Detections** with the new pattern.

## Comparisons You’ll Actually Use

| Tool                    | What it’s best at                                | How Detective fits                                                                 |
|-------------------------|--------------------------------------------------|-------------------------------------------------------------------------------------|
| **GuardDuty**           | Managed **detections** (threat intel + ML over logs). | **Detective** is the **investigation layer** you jump into after a **GuardDuty** alert to verify and scope. |
| **CloudWatch**          | **Signals** (metrics/logs/alarms) and real-time ops.  | Use it to see performance impact; **Detective** to reconstruct actor/resource behavior. |
| **Config**              | **What changed** (resource state & compliance).      | Pair with **Detective** to explain *who used the misconfig and how it was exploited*. |
| **CloudTrail Lake/Log** | **Raw log search** & long retention.                 | Use for deep-dive queries/retention; **Detective** is faster for **entity-centric pivots**. |

## Best Practices

- **Enable Detective org-wide** (at least prod + security-critical **dev**).  
  You can still scope which **sources** to include by account/Region to manage cost.

- **Keep GuardDuty on.**  
  Many investigations start there; integration is automatic.

- **Name owners for roles** (tags) so entity profiles immediately map to a team  
  (e.g., `Service=Blizzard-Checkout`, `Owner=Winterday`).

- **Standardize session tagging** (`aws:SourceIdentity`, `sts:TagSession`)  
  so **Detective’s** role views align with human change tickets.

- **Reduce upstream noise:** prune **VPC Flow** scope in sandbox accounts;  
  keep **EKS** audit logging focused (avoid ultra-verbose categories in **dev** clusters).

- **Practice game days.**  
  Stage a benign *“impossible travel”* or a noisy token **exfil** lab;  
  time the path from finding → **Detective** → blast radius → remediation **runbook**.

- **Track usage & cost.**  
  The **Usage** page shows ingested GB by source/account and 30-day projections—use it for **showback** and tuning.

---

## Real-Life Example

**Scenario.** A **GuardDuty** alert fires:  
*“Anomalous IAM activity: possible credential compromise for role Blizzard-Deploy@prod.”*

1. **Open in Detective.**  
   The role’s **entity profile** shows a **sudden spike in API categories** around 02:11 UTC, plus new **geolocation** (country we’ve never seen for this role) and a **new user agent**.  
   **Finding group** lists three related alerts (unusual S3 reads, `iam:List*` sprawl, and EC2 Describe bursts).

2. **Pivot by IP.**  
   The profile shows source IP `203.0.113.77`, new **ASO** for the account.  
   We pivot into that IP and see connections to two EC2 instances in a **public subnet** during the same window.

3. **Scope the blast radius.**  
   Timeline view shows the role touching `Winterday-Orders` and `Snowy-Data-Share` buckets with list/get attempts,  
   then failing to assume a higher-privileged role (blocked by boundary).

4. **Decide and act.**  
   Multiple “new” indicators + breadth of access attempts → escalate.  
   **EventBridge** triggers **SSM Automation**: revoke the role’s session permissions, rotate related **Secrets Manager** keys, and quarantine the two EC2 instances with a restrictive **SG**.

5. **Close with evidence.**  
   We capture the **Detective visuals** (new **geo**, new **ASO**, activity burst, related findings) and paste into the incident doc.  
   Postmortem: add a **condition on the trust policy** requiring `aws:SourceIdentity` and enforced **MFA** for human-federated paths;  
   add a **GuardDuty suppression rule** to carve known noise; keep **Detective** as the standard triage entry.

> **Outcome:** verified compromise attempt, bounded blast radius, precise evidence trail for leadership and auditors, and clear follow-up controls.

## Pricing Snapshot

**Current public pricing (example Region):**

- **First 1,000 GB/account/Region/month** → $2.00/GB  
- **Next 4,000 GB** → $1.00/GB  
- **Next 5,000 GB** → $0.50/GB  
- **Over 10,000 GB** → $0.25/GB

> Detector stores **12 months** of investigation data; you don’t pay extra for the UI or those aggregates.  
> Plan spend by shaping **upstream logs** and **which accounts/Regions** feed the graph.

---

## Final Thoughts

**Amazon Detective** is the **post-alert truth machine**:  
it turns raw log **firehoses** into an **investigable graph** you can walk like a map.
