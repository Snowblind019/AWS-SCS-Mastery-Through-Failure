# AWS Shield

---

## What Is The Service

**AWS Shield** is a managed Distributed Denial of Service (DDoS) protection service built directly into AWS. It’s always on, requires no configuration, and provides **automatic mitigation** against the most common types of DDoS attacks — like UDP floods, SYN/ACK floods, and reflection-based amplification attacks.

It’s designed to help you keep your AWS-hosted applications **available**, even when under attack.

> Shield is not an optional service — it’s silently protecting you behind the scenes the moment you use supported services like **CloudFront**, **Route 53**, or **ALBs**.

There are two tiers:
- **Shield Standard** (free, always on)
- **Shield Advanced** (premium, opt-in, deeper protection — covered separately)

For now, we’re focusing on **Shield Standard**.

---

## Cybersecurity Analogy

Imagine you’re in charge of a massive concert venue.  
Outside, there’s a **security perimeter** with guards whose job is to detect and deflect any chaos — like mobs of people trying to rush in without tickets.

**AWS Shield** is like those guards:
- You don’t see them on stage
- You don’t have to train them
- But they’re always outside, watching for patterns and stepping in when something unusual happens

They can’t stop *every* threat, but they’ll absorb the brunt of the chaos **before it hits your front door**.

## Real-World Analogy

Picture an **automatic surge protector** wired into the foundation of your house.  
You plug in your devices, not really thinking about it.

Then one day — a massive electrical surge from a lightning storm threatens your home.  
But your surge protector kicks in without needing your permission — it **absorbs and reroutes** the excess voltage, protecting your electronics.

You never even knew it saved your system.  
That’s AWS Shield — a built-in **circuit breaker for traffic volume**, and it’s already running behind the scenes.

---

## How It Works

Shield works by detecting **traffic anomalies** and filtering them before they hit your application. It uses a **global set of sensors** across AWS infrastructure and is **tightly integrated** with services like CloudFront, Route 53, and ELB.

### Flow Overview

1. **Traffic Hits AWS Edge Locations**  
   - For services like CloudFront and Route 53, traffic first hits AWS’s global edge network  
   - **Shield operates at the edge**

2. **Anomaly Detection**  
   - Monitors traffic volume, packet patterns, and sources  
   - Looks for DDoS signatures: SYN floods, DNS query floods, UDP reflection, etc.

3. **Automated Mitigation**  
   - Deploys mitigation rules: blocks IPs, drops bad packets, rate-limits traffic

4. **No Configuration Required**  
   - No setup, no tuning — it’s just always on for supported services

---

## What AWS Shield Standard Protects Against

| **Attack Type**        | **Description**                                         |
|------------------------|---------------------------------------------------------|
| SYN Floods             | Overwhelms a server with half-open TCP connections      |
| UDP Reflection Attacks | Spoofs traffic via open servers to amplify volume       |
| NTP / DNS Amplification| Exploits misconfigured servers to send large data bursts|
| Slowloris-style Attacks| Holds connections open indefinitely                     |
| Volumetric Floods      | Large-scale bandwidth exhaustion attempts               |

> These are blocked **before they even reach** your EC2 instance or application layer.

---

## What It Doesn’t Do

- Shield Standard **does NOT** handle **application-layer attacks** (e.g., HTTP floods, credential stuffing)  
  → Use **AWS WAF** or **Shield Advanced** for that

- No dashboards, alerts, or reports  

---


## What Services Does It Protect?

Shield Standard protects **only certain AWS services** — intentionally, so AWS can intercept traffic at the edge.

| **Protected Service**    | **Notes**                                   |
|--------------------------|---------------------------------------------|
| Amazon CloudFront        | Full protection at global edge locations    |
| Amazon Route 53          | DNS attack mitigation                       |

| Elastic Load Balancers   | ALB, NLB, CLB — protected at network layer  |
| AWS Global Accelerator   | Automatically benefits from Shield          |

> Deploying your app **directly on EC2 with a public IP**?  
> You’re **not protected** by Shield unless you use a fronting service like **CloudFront** or **ALB**.

---

### What Shield Standard Won’t Help With

- Brute-forcing login pages  
- Bots flooding your checkout API  
- Misconfigured firewalls exposing private endpoints  

For those threats, you need:
- **WAF**
- **Rate-based rules**
- **Shield Advanced**

---

## Pricing Model

| **Component**     | **Cost**                |
|-------------------|-------------------------|
| Shield Standard   | Free (100%)             |
| Opt-in required?  | No                      |
| Setup required?   | No                      |

> One of its greatest strengths: you’re protected **by default** if you use AWS-native architecture.

---

## Final Thoughts

You don’t always notice **AWS Shield**.  
That’s the point.

It’s part of AWS’s **quiet baseline security** — absorbing common volumetric attacks and letting your applications breathe.

If you’ve ever experienced **random traffic spikes but stayed online**… it may have been Shield at work.

---

Still — **Shield Standard** is like having 24/7 bouncers outside your AWS nightclub.  
It doesn’t handle the drama on stage, but it’s constantly **keeping the rabble from breaking down your front gate**.

> Use it. Rely on it.  
> But don’t assume it’s your entire defense.

