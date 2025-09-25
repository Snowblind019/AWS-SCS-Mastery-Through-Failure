# AWS GuardDuty — Deep Dive (Snowy Edition)

## What Is The Service

Amazon GuardDuty is AWS’s managed threat detection engine. It continuously analyzes AWS-native telemetry—CloudTrail events, VPC Flow Logs, Route 53 DNS query logs, plus optional sources like EKS audit/runtime signals, S3 data event signals, RDS login activity, and malware scans—to surface high-fidelity security findings with severity, context, and recommended actions. No sensors to deploy for the core detections, no signature packs to babysit, and it scales with your accounts/Regions automatically.  
Why it matters: misconfigurations and software flaws are only half the story; the rest is active misuse—credential theft, lateral movement, C2 beacons, data exfil. GuardDuty turns raw signals into actionable detections so Snowy’s team can move from “something looks weird” to “Blizzard-Checkout role made impossible-travel API calls from a new ASO; block, investigate, rotate.” It’s the always-on layer that pays attention while people sleep.

---

## Cybersecurity And Real-World Analogy

**Security analogy.** Imagine a modern alarm system for the Winterday campus:

- Cameras & door logs = CloudTrail and EKS audit events (who did what).
- Hallway motion = VPC Flow (who talked to whom).
- Visitor check-ins = DNS queries (who looked up what).
- Patrol AI = GuardDuty’s detectors (anomalies, intel, heuristics).
- Siren + dispatch = findings with severities and automatic routing to responders.

**Real-world analogy.** Think of airport security: the scanners are always running; when something suspicious appears, agents don’t re-invent the tooling—they triage and act. GuardDuty is that scanner for your AWS estate.

---

## How It Works

GuardDuty ingests supported data sources within AWS (no inline network taps required for the core set), runs a blend of machine learning, statistical anomaly detection, and threat-intel matching, and emits findings into its console and APIs. Each finding includes a type, resource principal, entities (IP, user agent, geo, ASO), first/last seen, count, severity (Low/Medium/High), and recommended steps. You can archive, suppress, or forward findings to EventBridge, Security Hub, SNS, SIEMs, and ticketing.

### Major Detection Families

**IAM & account activity**
- Unusual API patterns (new geos/ASOs, credential compromise hints)
- Privilege escalation attempts, suspicious access key behavior
- Root account usage, risky PassRole patterns, policy enumeration bursts

**Network & instance/container behavior**
- Outbound to known bad IPs/domains (C2, crypto mining pools)
- Port/protocol anomalies, brute-force attempts, reflective scans
- EC2 credential exfil attempts (IMDS misuse patterns)
- Malware protection (on-demand scans of EBS volumes tied to suspicious activity)

**DNS**
- DNS lookups tied to phishing/C2/malicious domains
- Newly observed domains with risky characteristics

**S3 data access**
- Suspicious data events (unusual gets, large volumes, new principals/locations)
- Anonymous/public access anomalies, cross-account risks

**EKS**
- Audit-log detections: abnormal k8s API verbs, high-risk object changes
- Runtime-oriented detections (when enabled): suspicious process/network/file signals from workloads

**RDS logins**
- Abnormal DB authentication sources, brute force, geo anomalies

_(Exact coverage evolves; you generally opt-in for S3 data events, EKS, RDS, and malware scans to expand beyond the core CloudTrail/VPC/DNS set.)_

## Findings: Anatomy, Severity, And Lifecycle

Every finding has:

- **Type** (taxonomy like `UnauthorizedAccess:EC2/SSHBruteForce`, `Recon:IAMUser/AnomalousBehavior`, `Exfiltration:S3/ObjectRead`)  
- **Severity** (Low/Medium/High) tied to potential impact & confidence  
- **Resource context** (ARNs, instance/role IDs, bucket/object paths)  
- **Evidence** (IP, geo, ASO, user agent, counts, timelines)  
- **Recommended steps** (block, isolate, rotate, investigate)

You can archive (acknowledge), suppress (mute by pattern/time window), or route to downstream systems. Findings remain searchable for post-incident review.

## Org-Wide Enablement & Scale

- Choose a **GuardDuty administrator** (e.g., Snowy-Security).  
- Auto-enroll member accounts via AWS Organizations; keep Regions aligned with where you actually operate.  
- Centrally view **coverage, data source status, and findings** across the Winterday organization.  
- Use delegated admin in the security account; keep per-account least-privilege roles for automation.

## Where GuardDuty Fits In The Bigger Picture

| Layer        | Purpose                                           | GuardDuty’s role                                                |
|--------------|---------------------------------------------------|------------------------------------------------------------------|
| Prevention   | Least privilege, segmentation, encryption, patching | Not prevention; informs where controls failed or are being probed |
| Detection    | Spot active misuse fast                           | **Primary signal generator** for active threats in AWS           |
| Investigation| Reconstruct story & scope                         | Hand findings to **Amazon Detective** (pivot across entities/timelines) |
| Response     | Contain/eradicate/recover                         | Wire to **EventBridge → SSM Automation / Lambda / SOAR**         |
| Posture      | Standards, config drift, hygiene                  | Pair with **Security Hub and Config** to keep posture tight      |

---

## Pricing Model

You pay for GuardDuty **data analysis** per enabled data source (e.g., per million CloudTrail events analyzed, per GB of VPC Flow/DNS/EKS/RDS data processed, per malware scan), with **tiering** as volumes grow. There’s no charge for “deploying agents” for the core sources because **GuardDuty taps AWS-native logs**.  
**Cost control** is about:

- Enabling only the data sources that matter to each account/Region,  
- Reducing upstream noise (e.g., chatty dev VPC Flow in sandboxes),  
- Scoping S3 data event coverage to critical buckets/prefixes,  
- Running malware scans when findings warrant (not as a blanket).  

_(Exact $ vary by Region and evolve; treat the above as design guidance rather than fixed numbers.)_

---

## Operational And Security Best Practices

1. **Turn it on everywhere that matters.** Start with prod + security-critical dev; align Regions with where workloads actually run.  
2. **Enable the right sources.** Core (CloudTrail, VPC Flow, DNS) is a must; add S3 data events, EKS, RDS logins, malware protection where applicable.  
3. **Route findings immediately.** EventBridge → Incident Manager, Pager, SSM Automation. High severity should page Blizzard-OnCall; low/medium can open tickets with SLA.  
4. **Suppress responsibly.** Use suppression rules for known benign patterns, but time-box them and review monthly.  
5. **Tag for ownership.** Surface Service, Team, Environment on implicated resources so findings auto-route to the right people.  
6. **Pre-bake runbooks.** For top finding types, have one-click containment: quarantine SG, disable access keys, detach suspicious policy, snapshot & isolate an instance, rotate secrets.  
7. **Close the loop.** After containment, jump into Detective for blast-radius confirmation; update Config/SCP/IAM so the same path is harder next time.  
8. **Watch the signals.** Dashboards for findings by severity over time, MTTD/MTTR, top reoccurring types, and coverage drift by account/Region.  
9. **Cost hygiene.** Prune dev VPC Flow in toy accounts, scope S3 data events to sensitive prefixes, and keep Regions to a minimum.  
10. **Game days.** Rehearse: staged “impossible travel,” fake exfil, noisy brute force—measure time from alert → containment → evidence collection.

## Common Finding → Response Map

| Finding family           | Example types                                      | Fast containment                                                        | Next steps                                                                 |
|--------------------------|----------------------------------------------------|-------------------------------------------------------------------------|----------------------------------------------------------------------------|
| Credential compromise    | Recon:IAMUser/AnomalousBehavior, UnauthorizedAccess:IAMUser/ConsoleLogin | Disable access keys, require password reset/MFA, block source IPs       | Detective pivot (new geo/ASO), rotate related Secrets Manager creds, review trust/PassRole |
| EC2 abuse / C2           | Backdoor:EC2/Spambot, CryptoCurrency:EC2/BitcoinTool.B!DNS, Behavior:EC2/NetworkPortUnusual | Quarantine SG (egress restrict), snapshot volumes, tag for forensics     | Malware scan on volumes, patch AMI/bake, analyze user-data/IMDS use        |
| S3 exfil/suspicious access | Exfiltration:S3/ObjectRead, Policy:S3/BucketPublic (from posture) | Block public access, tighten bucket policy/KMS conditions, suspend temporary access | Validate requester, run Macie scan, review CloudTrail for new sharing      |
| EKS risky activity       | Kubernetes:MaliciousIPCaller, Kubernetes:AnonymousAccess | Isolate workload namespace/SG, cordon+drain node if needed              | Detective/DynamoDB for pivot; audit RBAC, rotate service account tokens     |
| RDS brute/abnormal logins | RDS:SuspiciousLogin                               | Network ACL/SG tighten, require IAM DB auth/MFA where possible           | Review application creds & rotation cadence                                |

## EKS & Container Nuance

- Audit-only gives control-plane visibility (kubectl/API anomalies).  
- With runtime signals enabled, detections include suspicious processes/network/file activity from cluster workloads.  
- Combine with EKS IRSA (per-pod IAM roles) to shrink blast radius; GuardDuty findings will then point to the exact service account/role.

## S3 & Data Exfil Nuance

- Turn on S3 protection for sensitive buckets (customer data, secrets, reports).  
- Pair with Macie for classification so “suspicious read” is weighted by data sensitivity.  
- Enforce KMS with conditions (`s3:x-amz-server-side-encryption`) and VPC endpoint-only access for internal readers.

## Wiring It To Automation

**EventBridge** rule on High severity → **SSM Automation**:

- If EC2 implicated → attach quarantine SG, snapshot EBS, tag `QuarantinedBy=GuardDuty`.  
- If IAM user implicated → disable keys, require password reset, notify owner.  
- If S3 implicated → apply Block Public Access and restrictive bucket policy, notify DataOwner.

- Medium severity → ticket with 24-hour SLA; Low → weekly triage queue.  
- Notify: Pager for High, chat webhook for Medium/Low summaries.

---

## Real-Life Example

**Scenario.** At 02:11 UTC, High severity finding: “Credential compromise suspected for role Blizzard-Deploy@prod.”

1. **Alert lands** via EventBridge → Incident Manager; Blizzard-OnCall is paged with finding JSON and a deep link.  
2. **Immediate containment runbook:**
   - Update the role’s session policy to block sensitive actions,  
   - Detach recently attached inline policies,  
   - Add a Service Control Policy exception block for destructive calls until triage completes.  
3. **Pivot to context:**
   - CloudWatch alarm also shows unusual S3 GetObject spikes from the same role.  
   - Open Detective: new geolocation + ASO for the role, burst of `List*` and `Describe*` across multiple services, DNS queries to suspicious domains from an EC2 in the same account.  
4. **Widen the net:**
   - Quarantine the EC2 instance SG (egress deny), trigger malware scan of its EBS volume, snapshot for forensics.  
   - Rotate Secrets Manager tokens tied to pipelines the role can reach.  
5. **Resolution:**
   - Finding group shows related `Exfiltration:S3/ObjectRead` on Snowy-Data-Share; Macie scan confirms no regulated PII in the accessed prefix.  
   - Root cause: leaked CI token in a third-party tool; trust tightened with OIDC + session tags and reduced duration.  
6. **Close & harden:**
   - Add trust-policy conditions (`aws:SourceIdentity`, repository OIDC claims), implement MFA for human paths, and keep suppression rules tight (only for noisy, benign patterns).  

**Outcome:** Compromise verified and contained in minutes; scope precisely documented; controls updated so the same path is narrower tomorrow.

---

### Enablement Checklist By Account/Region

| Item                                         | Status | Notes                                         |
|----------------------------------------------|--------|-----------------------------------------------|
| Delegated admin set (Snowy-Security)         | ⬜     | Organization-wide                             |
| Core sources (CloudTrail, VPC Flow, DNS)     | ⬜     | All prod Regions                              |
| S3 protection (sensitive buckets)            | ⬜     | Winterday-PII, Blizzard-Exports               |
| EKS protection (clusters)                    | ⬜     | Audit + runtime where required                |
| RDS login protection                         | ⬜     | Prod DBs only                                 |
| Malware protection                           | ⬜     | Triggered by suspicious instance findings     |
| EventBridge routing                          | ⬜     | High → page; Med → ticket; Low → digest       |
| Detective integration                        | ⬜     | “Investigate in Detective” tested             |
| Runbooks bound                               | ⬜     | Quarantine, rotate, snapshot, block patterns  |

## Tuning Knobs You’ll Actually Use

| Area     | Knob                     | Why                                               |
|----------|--------------------------|----------------------------------------------------|
| Noise    | Suppression rules        | Mute benign repeaters; time-box and review         |
| Cost     | S3 protection scope      | Focus on sensitive prefixes/buckets                |
| Coverage | Region/account map       | Fewer Regions = fewer surprises & $                |
| Response | Severity routing         | High pages humans; Med/L can batch                 |
| Evidence | Session tagging (sts:TagSession) | Tie findings to owners/changes fast     |

---

## Final Thoughts

GuardDuty is the always-awake detector for AWS. Turn it on broadly, feed it the right sources, wire the top finding families to one-click containment, and make Detective your right-after step for context. Pair it with Config/Security Hub for posture, and keep Snowy’s runbooks sharp so High severity doesn’t become a committee meeting at 3 a.m. Do that, and when the wind picks up, you’ll get fewer surprises—and faster, cleaner responses when one slips through.
