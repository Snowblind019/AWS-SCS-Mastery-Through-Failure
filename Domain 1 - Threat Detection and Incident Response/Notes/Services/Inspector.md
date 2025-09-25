# Amazon Inspector

## **What Is the Service**

**Amazon Inspector** is AWS’s *managed vulnerability scanner for your compute*: it continuously evaluates **EC2 instances**, **container images in Amazon ECR**, and **AWS Lambda** functions/layers for **known software vulnerabilities (CVEs)** and related risks. It tracks what packages you’re running, correlates them with vulnerability intelligence, and prioritizes findings so you can fix the right things first.  
No cron jobs, no fleet daemons to hand-tune—turn it on, scope what to cover, and Inspector keeps watching as code, images, and functions change.

**Why It Matters:** real incidents often begin with **unpatched software**. A forgotten library in `Blizzard-Checkout`, a base image in `Winterday-Orders` that quietly went stale, a Lambda layer with a vulnerable parser—these are the doors attackers try first. Inspector shortens the **“known vulnerable → exploitable in your estate”** window and **pushes the right fixes to the right owners** with evidence you can track.

---

## **Cybersecurity and Real-World Analogy**

### **Security Analogy**

Picture the **Winterday** campus with maintenance tags on equipment. **Inspector** is the **automated safety inspector** who walks the floors every day, scans serial numbers, checks them against a global “bad parts” list, and posts **priority work orders**:

> “Replace this valve now (*actively exploited*), that one next sprint (*low likelihood*).”

It doesn’t replace alarms (**GuardDuty**) or door logs (**CloudTrail**); it keeps **known weaknesses** from sitting around.

### **Real-World Analogy**

Car recall notices. The manufacturer discovers a defect in a specific batch of parts (**CVE**). Inspector checks your “garage” and says:

> _“Snowy, your 2019 model with build XYZ needs a fix; here’s where it is and how urgent.”_

---

## **How It Works**

### **Coverage at a Glance**

| **Surface**     | **What Inspector Scans**                  | **How It Collects Data**                                     | **Typical Triggers**                                                             |
|-----------------|-------------------------------------------|---------------------------------------------------------------|----------------------------------------------------------------------------------|
| **EC2**         | OS packages, language runtimes, common libs | **SSM Agent** inventory + service-side analysis (no separate agent) | On instance start/stop, package changes, scheduled refresh                        |
| **ECR (Containers)** | Image layers (OS + language deps)        | Event-driven at push + periodic re-scan of existing images     | Push to ECR, new **CVE** published, **repo** policy change                        |
| **Lambda**      | Function code and layers (packaged libs)   | Service-side analysis of artifacts                             | On publish/update, new **CVE**, layer change                                     |

### **Finding Anatomy**

_All targets_:  
- **CVE ID**  
- Affected package & version  
- Path/layer  
- **Severity**, **exploit context**, **remediation guidance** (fixed version or mitigation)  
- Affected resources

## **Prioritization (Why Some Findings Matter More)**

Inspector doesn’t just say *“you have CVE-XXXX.”*  
It scores findings to reflect **likelihood of exploit** + **exposure**, so **Snowy** works on the right issues first.

**Inputs commonly include:**
- ✅ **Exploit intel signals** (is there known active exploitation or a weaponized **PoC**?)
- 🌐 **Network exposure context** for EC2 (publicly reachable vs private)
- ⚙️ **Runtime relevance** (package present vs actually used, where available)
- 📌 **Asset criticality** (you can feed tags/metadata to reflect importance)

> **Result**: a **sorted queue** where *High/now* often means *actively exploitable + internet-exposed*.

## **Inspector Findings Lifecycle**

1. **Discovery** — Inspector detects packages/images/functions and maps them to known **CVEs**.  
2. **Correlation & Scoring** — Applies **exploitability/context** signals; assigns severity & a ranked score.  
3. **Delivery** — Findings surface in **Inspector**, **Security Hub (ASFF)**, and optionally **EventBridge**.  
4. **Triage** — Filter by `Environment=prod`, `Service=Blizzard-Checkout`, severity, or tag.  
5. **Remediation** — Patch OS, rebuild container base, bump Lambda layer; verify fixed version.  
6. **Closure** — Inspector auto-resolves when the vulnerable version disappears from the asset.

## **Integrations That Matter**

- **Security Hub** — Central queue, standards scoring, custom actions to run playbooks.  
- **EventBridge** — Route **High** findings to **SSM Automation** or ticketing; batch **Medium/Low**.  
- **Systems Manager (SSM)** — Patch Manager/Automation docs to apply fixes for EC2 fleets.  
- **CodeBuild/CodePipeline** — Break builds or block deploys on **High vulns** (containers/Lambda).  
- **EKS/ECR Policies** — Prevent image deploy if **repo** has unresolved **Highs** (policy + admission control, if wired).

---

## **Pricing Model**

Inspector charges are tied to **what you scan** and **how much**:

- 🖥️ **EC2 Coverage** — Based on the population of covered instances over time.  
- 🐳 **ECR Images** — Per image scan/re-scan volume.  
- 🔁 **Lambda** — Per function/layer coverage over time.  

> It’s tiered by volume and Region. Tune cost by:  
> - Scoping coverage (environments/Regions)  
> - Retiring unused images  
> - Focusing **repos/functions that ship to prod**

Check your Region’s pricing before committing.

---

## **Operational & Security Best Practices**

1. **Turn It On Where It Matters First.**  
   Prod accounts & Regions → then expand to staging.

2. **Tag for Ownership.**  
   On EC2, **ECR repos**, Lambda:  
   `Service`, `Team`, `Environment`, `DataOwner`.

3. **Shift Left.**  
   Scan container images at **build** and **block** deploys with High/Critical findings.

4. **Thin Your Images.**  
   Smaller base → fewer packages → smaller attack surface + faster remediation.

5. **Pin + Update.**  
   Pin versions in `Dockerfiles`/`requirements.txt`; adopt **base-image refreshes**.

6. **Automate EC2 Patching.**  
   Use **SSM Patch Manager**; validate with Inspector that vulns closed.

7. **Lambda Hygiene.**  
   Centralize layers, version them, roll forward aggressively. Keep functions small.

8. **Don’t Cargo-Cult Severity.**  
   Use **exploitability + exposure**. Score with Inspector, then layer business value.

9. **Wire Response.**  
   High → `EventBridge → SSM`/Lambda **runbook**  
   Medium/Low → ticket with **SLA**.

10. **Kill the Long Tail.**  
    Remove **unused ECR images** and dead **Lambda layers**.

# Comparisons You’ll Actually Use

| **Tool**               | **Best At**                              | **How Inspector Fits**                                                                 |
|------------------------|------------------------------------------|----------------------------------------------------------------------------------------|
| **GuardDuty**          | Active threat **detections** (anomaly/intel) | Inspector addresses **vulnerability presence** before/alongside threats                |
| **Macie**              | Sensitive data in S3                     | Complementary: Inspector lowers software risk; Macie lowers **data exposure** risk     |
| **Config**             | Resource posture/compliance             | Use Config to enforce “must run current AMI/must use approved base image”; Inspector verifies **CVE status** |
| **Security Hub**       | Posture hub & workflow                  | Central triage + automation for Inspector findings                                     |
| **Third-Party Scanners** | Language/**SCA/DAST/SAST** depth       | Pair with Inspector for **AWS-native continuous coverage** and org scale               |

---

# Hands-On: Patterns That Work

## 1) Containers (**ECR**) — Build-Time + Registry Guard

### **Flow**

- CI builds `blizzard/web` → runs an image scan → **fails build** if High/Critical.  
- Successful images are pushed to **ECR** (Inspector scans again on push, then **re-scans** on new **CVEs**).  
- **Admission** (if using **EKS**): only allow images with `ImageTag=approved` **and** no High findings.

### **Why It Works**

- Catches issues before they hit prod, and **keeps catching** as new **CVEs** appear.

## 2) EC2 Fleet — Patch-at-Scale With **SSM**

### **Flow**

- Inspector flags `CVE-XXXX` on `Snowy-API` instances.  
- **EventBridge** rule (High) triggers **SSM Automation** to:  
  - snapshot (optional),  
  - apply OS patch set,  
  - reboot during maintenance window,  
  - verify package versions.  
- **Security Hub** updates the finding workflow to `NOTIFIED → RESOLVED` when Inspector closes it.

### **Why It Works**

- Turns findings into **repeatable runbooks** with auditability.

## 3) Lambda — Layer Version Discipline

### **Flow**

- All functions use a small set of **blessed layers** (e.g., `blizzard-python-layer@v42`).  
- Inspector flags High in that layer; pipeline bumps to `v43` with fixed libs.  
- **One rollout** updates all functions referencing the layer; Inspector findings drop.

### **Why It Works**

- **Single knob** per language stack instead of 200 function-specific patches.

---

# Tuning & Hygiene Tables

## Coverage & Routing Matrix

| **Surface** | **Environment** | **Severity**     | **Action**                                                                 |
|-------------|------------------|------------------|--------------------------------------------------------------------------|
| ECR         | prod             | High/Critical    | Block deploy; open Pager + ticket; rebuild base image                    |
| ECR         | staging/dev      | High/Critical    | Block merge; ticket to team; justify & time-box exceptions               |
| EC2         | prod             | High             | Patch via **SSM** this window; escalate if not closed in 48h             |
| Lambda      | prod             | High             | Bump layer; redeploy; verify via Inspector; notify **Blizzard-OnCall**   |
| Any         | any              | Medium           | Ticket with 7–14d **SLA**; batch per service                             |
| Any         | any              | Low              | Weekly digest; fix during posture sprint                                 |

## Dockerfile Quick Wins

| **Problem**          | **Fix**                                                                |
|----------------------|------------------------------------------------------------------------|
| Bloated base image   | Switch to `distro-minimal/base-alpine` (when compatible)              |
| Unpinned packages    | Pin exact versions; record **SBOM/manifests**                         |
| Layer sprawl         | Combine RUN steps; purge caches (`apk/apt clean`)                     |
| Old base             | Rebuild weekly from latest base, even if app code unchanged           |

---

# Real-Life Example

### **Scenario**

A new `Winterday-Orders` release is ready. CI succeeds, but Inspector (**ECR**) flags a **High** in  
`blizzard/orders-api:2025.09.24 – OpenSSL` in the base image has a fresh **CVE**.

1. **Gate Hits**: The build step fails because policy forbids deploying Highs to prod.  
2. **Fix Path**: Platform team updates the **base image** to a patched digest. App rebuilds with **no code changes**; image size shrinks a bit thanks to a slimmer base.  
3. **Verification**: Inspector re-scans on push; **no Highs** remain. Security Hub rule clears the block.  
4. **Deploy**: **EKS** admission allows the new tag; rollout proceeds.  
5. **Cleanup**: A **repo** policy job deletes the vulnerable image tag after a grace period; findings auto-close as the artifact disappears.  
6. **Evidence**: Hub shows finding closed with remediation notes; monthly report captures time-to-fix and affected services (just **Blizzard-Orders**, thanks to shared base images).

**Outcome**: A widely exploited **CVE** never reached prod; you got the fix in hours with minimal human choreography.

---

# Final Thoughts

**Amazon Inspector** is your **always-on CVE radar** for EC2, containers, and Lambda.  
Use it to **rank risk sanely** (exploitability + exposure), **push fixes automatically** (**SSM**, rebuilds, layer bumps), and **prove closure** via Security Hub.
