# Amazon CloudFront

---

## What Is the Service (And Why It’s Important)

**Amazon CloudFront** is AWS’s **Content Delivery Network (CDN)** — a globally distributed system that caches and serves content (static or dynamic) close to users to reduce latency and improve speed.

But CloudFront is more than just a performance booster. It’s a **security perimeter** and strategic **edge control point**:

- Terminates **SSL/TLS** connections
- Blocks malicious traffic via **AWS WAF**
- Enforces **geo-restrictions**
- **Logs** detailed access records
- Serves as the **edge layer** in zero-trust architectures

> CDNs like CloudFront are often the **first layer exposed to the internet**, making them a critical layer for performance, **security posture**, and **incident visibility**.

---

## Cybersecurity Analogy

Think of your origin servers (S3, EC2, ALB, etc.) as a **secure building**, and CloudFront as a **chain of security gates** in every major city.

Each gate can:
- ✅ Check IDs (auth tokens, headers, IPs)
- ✅ Block bad actors (WAF rules)
- ✅ Allow only guests from specific regions (geo-blocking)
- ✅ Keep an access log

Your building remains **hidden, clean, and protected** — while the gates handle the incoming traffic and threats.

## Real-World Analogy

CloudFront is like a **worldwide network of delivery lockers**.

You store your product (video, image, HTML page, API response) in the **closest locker to the customer**.

The customer:
- Gets **faster access**
- Doesn’t wait for **cross-continent delivery**
- Enjoys **secure, auditable delivery**
- Can be **tracked, validated, and blocked** if needed

If something suspicious happens (too many hits from one region, bot traffic, etc.), you can **lock the locker** or **reroute traffic** elsewhere.

---

## Key Concepts & Flow

### Origins
The original data sources (e.g., S3, ALBs, EC2, custom domains).  
CloudFront **pulls** from the origin and **caches** the content temporarily.

### Edge Locations
600+ global locations that:
- Cache and serve content
- **Terminate SSL**
- **Run WAF rules**
- **Log requests**

### Distributions
Configurations that define:
- Request routing
- Caching behavior
- Logging
- Security settings

### Behaviors and Rules
Per-path configurations that define:
- Which **origin** to use
- Caching strategies
- WAF associations
- Header/query forwarding
- Allowed HTTP methods
- Lambda@Edge or Function@Edge logic

---

## Security Features

| **Security Feature**        | **What It Does**                                                                 |
|----------------------------|-----------------------------------------------------------------------------------|
| **HTTPS Only**             | Forces all viewer traffic to use TLS                                            |
| **Signed URLs/Cookies**    | Restrict access to private content with expiration and user specificity         |
| **Geo-restrictions**       | Allow/block access based on viewer’s country                                    |
| **AWS WAF Integration**    | Blocks common exploits like SQLi/XSS, bad bots, and rate-based attacks          |
| **Shield & Shield Advanced** | DDoS protection with proactive mitigation and support                           |
| **Access Logs**            | Captures detailed request logs to S3                                            |
| **Origin Access Control (OAC)** | Restricts access to origins — replaces legacy OAI                          |
| **Field-Level Encryption** | Encrypt specific HTTP fields before they hit the backend                        |

---

## CloudFront Logging Options

| **Log Type**          | **Destination**                     | **Why It Matters**                                              |
|----------------------|--------------------------------------|-----------------------------------------------------------------|
| **Standard Access Logs** | S3 Bucket                        | Logs every viewer request (IP, URI, response code, etc.)        |
| **Real-Time Logs**    | Kinesis Data Streams                | Near-instant delivery to SIEM/SOC tools                         |
| **WAF Logs**          | Kinesis Firehose → S3/Elasticsearch | See which requests triggered which WAF rules                    |
| **Shield Metrics**    | CloudWatch                          | Track attack volume, mitigations, anomalies                     |

---

## Typical Use Cases

| **Use Case**             | **How CloudFront Helps**                                                 |
|--------------------------|---------------------------------------------------------------------------|
| Static Websites (S3)     | Fast global delivery + HTTPS + access control                            |
| API Protection           | WAF rules + IP filtering + low-latency edge delivery                     |
| Private Content Delivery | Signed URLs, cookie-based auth                                           |
| DDoS Mitigation          | Combines Shield, WAF, geo-blocking, and rate limits                      |
| Zero-Trust Microservices | Origin access restricted to CloudFront only                              |
| Compliance & Logging     | Full HTTP logs, real-time analytics, audit visibility                    |

---

## Pricing Model

CloudFront pricing is **usage-based**:

- **Data Transfer Out**: Charged by GB, region-dependent
- **HTTP/HTTPS Requests**: Priced per 10,000 or per million
- **Invalidation Requests**: First 1,000/month free
- **Real-Time Logs**: Charged per record sent to Kinesis
- **WAF & Shield**: Billed separately (if enabled)

> Example:  
> If your app serves **1 TB/month** + **100k HTTPS requests**:
> - You pay for **data transfer out**
> - You pay for **HTTPS requests**
> - Any log delivery or WAF usage adds to the total

---

## Real-Life Security Example

**Winterday’s e-commerce platform** hosts product images in S3 behind CloudFront.  
One night, they notice:

- 🔺 Traffic spike from a **single IP in Eastern Europe**
- 🔺 All requests go to `/images/private/`
- 🔺 User-agent is `curl/7.68.0`
- 🔺 90% of requests fail with **403 errors** (invalid signed URL)

### Their defenses:

- ✅ **CloudFront WAF** blocks rate over 2,000 reqs / 5 min  
- ✅ TLS 1.2+ enforced  
- ✅ IP range blocked  
- ✅ Cached objects invalidated  
- ✅ Signed URL keys rotated  
- ✅ **Lambda@Edge** deployed for deeper token validation  

**Result**: Threat neutralized before it reached the origin.

---

## Final Thoughts

**CloudFront isn’t just a CDN — it’s your programmable edge firewall, global cache, and shielded gateway.**

It gives you:
- Fast content delivery  
- Strong perimeter defense  
- Real-time visibility  
- Regional control  
- Edge-layer customization

> If your app touches the internet, **CloudFront is your first line of performance and security**.
