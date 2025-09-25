# Root Cause Analysis — Not Just Fixing the Symptom  

---

## The Problem

Most teams stop too early in incident response.  
Something breaks.  
Someone patches it.  
Services come back up.  
Everyone moves on.

But if you don’t ask *why* it broke in the first place — you’re not solving the real problem.  
You’re just applying a band-aid to something that’s going to bleed again later.  

**Root Cause Analysis (RCA)** is about digging deep.  
Not just *what* happened, but *why* it happened and what you’ll do to prevent it next time.

---

## Why Root Cause Analysis Matters

If you skip RCA, here’s what tends to happen:
- Outages and security events repeat themselves  
- The same vulnerabilities go unpatched  
- You waste engineering hours fixing the same issues over and over  
- Stakeholder confidence drops (execs, customers, compliance)  
- Detection doesn’t improve, because you’re not learning  

**RCA turns pain into progress.**  
It turns incidents into fuel for hardening your systems, your processes, and your team’s mindset.

## Symptom vs Root Cause

Here’s what that difference looks like:

| Symptom                        | Root Cause                                              |
|-------------------------------|----------------------------------------------------------|
| EC2 instance unreachable       | Security group misconfiguration                         |
| S3 bucket publicly accessible  | Lack of automated checks for bucket policies            |
| IAM key used by attacker       | Developer committed .env file to GitHub                 |
| Lambda throwing API error      | Upstream service deployed breaking change, no versioning |
| Unauthorized login via API     | No MFA, key reused from prior breach                    |

> You can’t stop at the surface.  
> You need to ask:  
> *Why was this possible?*  
> *What would have prevented it?*

---

## The 5 Whys Technique

A simple (but powerful) RCA method: keep asking **why** until you hit the real cause.

**Example:**
- Why was the S3 bucket public?  
  → Because someone removed the deny policy.  
- Why did they do that?  
  → The app was failing in test and they wanted to debug quickly.  
- Why didn’t they use a safer debug method?  
  → They didn’t know how and were under pressure.  
- Why wasn’t there a default protection or SOP?  
  → Because the org never published S3 hardening guidelines.  
- Why not?  
  → Because no one owns cloud policy education internally.  

So the root cause is **not** “someone made a bucket public.”  
It’s **“lack of internal guidance and ownership around secure S3 defaults.”**

> Now you can fix the real issue.

---

## RCA Report Template (What to Capture)

You don’t need to write a novel — but you do need a consistent format.

**Incident Summary**
- What happened  
- When and how it was detected  

**Impact**
- Who or what was affected  
- Duration and severity  

**Timeline of Events**
- Chronological facts  
- No emotion, no blame  

**Root Cause**
- Go beyond symptoms  
- Document the underlying flaw  

**Contributing Factors**
- Weak alerts, noisy logs, poor architecture  
- Anything that made it worse  

**Resolution**
- What fixed it (temporary or long-term)  

**Lessons Learned**
- What caught you off guard?  
- What should change going forward?  

**Action Items**
- Code/process/tooling fixes  
- Due dates + owners  
- Timeline for completion  

---

## Blameless ≠ Toothless

“Blameless” doesn’t mean “soft.”  
It means we don’t single out people — but we *do* call out process failures.

It’s okay to say:
- “No required code reviews on IAM changes”  
- “Rollback procedures were never tested”  
- “No alert on critical logins from new geos”  

But the moment you start pointing fingers at individuals, people shut down.  
**Blame leads to silence.**  
**Silence kills learning.**

---

## Final Thoughts

Incidents are your system’s way of saying:  
**“Something’s wrong — please look deeper.”**

Fix the fire, yes. But then go figure out **who built the flammable walls**.

**Root Cause Analysis** is your best tool for long-term maturity:
- Fewer repeat incidents  
- Stronger infrastructure  
- Better team alignment  
- Increased trust  

You didn’t go through all that pain just to move on.  
**Ask why. Then ask again. And again.**  
That’s how you stop reacting — and start building **resilient systems**.
