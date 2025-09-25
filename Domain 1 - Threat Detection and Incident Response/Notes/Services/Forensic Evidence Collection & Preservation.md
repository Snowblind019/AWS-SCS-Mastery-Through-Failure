# Forensic Evidence Collection & Preservation  

---

## Why This Matters

When an incident hits, your next 30 minutes determine everything.  
Do you preserve the truth — or trample over it?  
Do you collect evidence that stands up to scrutiny — or lose the trail forever?

Forensics isn’t just about “figuring out what happened.”  
It’s about capturing proof — in a way that’s reliable, untampered, and admissible (internally or legally).  
Without that, your investigation is guesswork.  
If you destroy logs, alter timestamps, or miss a memory snapshot — you lose context, credibility, and the ability to respond meaningfully.  
You only get one shot to capture clean evidence. Don’t waste it.

---

## Core Principles of Digital Forensics

### Preserve First, Analyze Later
Don’t log into the compromised host and start poking around.  
Every command you run can overwrite RAM, alter files, rotate logs, or destroy artifacts.  
Your first move should always be to freeze the state, not dissect it.

### Minimize Interaction
Don’t reboot. Don’t shut down. Don’t patch. Don’t uninstall anything.  
If the system is live, volatile data (RAM, active sessions, tokens) will vanish in seconds.

### Maintain Chain of Custody
For any evidence collected, you need to be able to prove:
- Who touched it
- When it was touched
- What was done to it

If you can’t prove that, even internal findings can be challenged.  
**Chain of custody builds trust into the investigation.**

---

## Types of Forensic Evidence (and How to Collect It Right)

### 1. Snapshots / Disk Images

**Cloud:**
- Take an EBS snapshot or disk image ASAP (for EC2, ECS, etc.)
- Use automation (Lambda, SSM runbook) to reduce human error
- Tag the snapshot with: incident ID, timestamp, investigator initials

**On-prem:**
- Use tools like `dd`, `dcfldd`, or **FTK Imager** to clone the full disk
- Save to read-only or offline media
- Hash the image before and after copying

**Goal:** A bit-for-bit copy of the system at compromise time — untouched, unmodified

### 2. Memory Dumps (RAM)

**RAM can contain:**
- Decryption keys  
- Malware running in memory  
- Tokens, sessions, passwords  
- Network connections  
- Shell history (before being flushed)  

**Cloud:**
- Use scripts like `capture-memory.ps1` via SSM for Windows
- For Linux, capture with **EC2 Rescue** or live snapshot tooling

**Local:**
- Use **Volatility**, **LiME**, or **FTK Imager Live** for memory capture

**Always document:**
- Tool used
- Exact timestamp
- Hash of the dump file
- System context (uptime, active processes)

> Memory is golden — and vanishes instantly if you reboot.  
> Get it fast, or it’s gone.

### 3. Logs (With Integrity!)

**Logs tell you:**
- Who logged in  
- What APIs were called  
- What changed  
- When lateral movement began  
- Where exfiltration happened  

But logs are fragile:
- They can be deleted or rotated
- Attackers tamper with timestamps or entries
- A misconfigured log level might hide critical details

**Preserve them by:**
- Pulling from centralized logging (CloudWatch, S3, SIEM)
- Hashing every log file right after collection  
  `sha256sum /var/log/syslog > syslog.sha256`
- Locking them in immutable buckets (S3 Object Lock, versioning)

> If it’s not hashed and immutable, it’s not trustworthy.

### 4. Metadata (Context Is Everything)

Beyond logs and snapshots, you need to capture the environment itself:
- EC2 instance ID, tags, AMI used
- IAM role and permissions at the time
- Security group + NACL rules
- Auto-scaling activity (was it newly deployed?)

This is what lets you recreate the system later and understand:
- Was it misconfigured at birth?
- Was it drifted by deploy tooling?
- Was it changed manually?

> Without context, evidence is just noise.

---

## Chain of Custody

**Chain of custody** is your audit trail for the evidence itself.  
You must be able to prove:
- Who collected the data  
- When they collected it  
- Where it was stored  
- Who accessed it later  
- That it wasn’t modified  

**In practice:**
- Keep an evidence log (manual or automated)
- Hash all artifacts (logs, images, dumps) before and after copying
- Store originals in cold or read-only storage (S3 with Object Lock, write-once media)
- Only analyze copies, never originals
- Store hash records in a tamper-proof system (external vault or SIEM)

> Bad chain = tainted evidence.  
> And tainted evidence = a broken investigation.

---

## Example: Cloud-Based Incident Timeline

**Detection:** GuardDuty flags suspicious outbound traffic from EC2  
**Initial Response:** Use SSM to isolate the host (remove from ASG + security group)  
**Collection:**
- EBS snapshot  
- Memory dump via prebuilt script  
- Pull CloudTrail, VPC Flow, and application logs  
- Capture instance metadata, IAM role, SG state  

**Preservation:**
- SHA256 hash of everything  
- Stored in S3 with versioning + object lock  
- Update chain of custody log  

**Analysis:** All investigation done on copied volumes and logs only

---

## Final Thoughts

Anyone can reboot a server and say “fixed.”  
But that’s not response — that’s amnesia.

Real security teams don’t just stop the bleeding — they preserve the story.  
They think in terms of evidence, timeline, intent, and verification.

**Forensics is slow. Methodical. Precise.**  
But when it’s time to brief your CISO, answer legal, or testify in court…  
**the truth is in the evidence.**  
And evidence only survives if you had the discipline to collect and preserve it right.
