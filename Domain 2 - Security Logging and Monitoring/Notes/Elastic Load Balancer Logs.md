# Elastic Load Balancer (ELB) Logs  

## What Is It

In AWS, Elastic Load Balancers (ELBs) are the front doors to your applications. They sit in front of your EC2 instances, containers, or Lambda functions and distribute incoming traffic across them.  
When something breaks, is slow, or seems suspicious — ELB access logs are your first place to look.

These logs record every connection that passes through your load balancer, whether it’s a successful request, an error, or a sudden spike in traffic.  
If CloudTrail tells you what AWS resources are doing, ELB logs tell you what your users (or attackers) are doing at the application edge.

---

## Cybersecurity Analogy

Think of your ELB as the reception desk at a secure facility.  
Everyone passes through it — visitors, employees, couriers. The receptionist logs:

- Who came in  
- When they arrived  
- What they asked for  
- How long they stayed  
- Whether they were allowed in or turned away

If someone later reports an incident (“My bag is missing” or “Unauthorized person in the building”), the front desk logs are the first place you look.  

Same with ELB logs. They capture the real world application access patterns — who accessed what, when, from where, and what response they got.

## Real-World Analogy

Imagine you run a highway toll booth.  
You don’t know what people do after they pass the booth, but:

- You know when they drove by  
- What car and license plate  
- Whether the toll went through or failed  
- And how long they sat in the lane

That’s what ELB logs are. You don’t see inside the car (the payload), but you know the entire metadata trail of how people used your “digital highway.”

---

## What It Logs

When you enable access logging on an ELB (Application, Network, or Gateway), AWS delivers log files to an S3 bucket you specify. These files are gzipped text files, written in W3C extended log format (tab-delimited fields).

Each log entry includes:

| Field                  | Meaning                                                                 |
|------------------------|-------------------------------------------------------------------------|
| type                   | Type of ELB (app, net, gateway)                                         |
| timestamp              | Time the request was received                                           |
| elb                    | Name of the load balancer                                               |
| client:port            | IP and port of the client (usually public)                              |
| target:port            | IP and port of the backend (EC2/Container) that handled the request     |
| request_processing_time| Time the load balancer took to read the request headers                 |
| target_processing_time | Time the target took to process the request                             |
| response_processing_time| Time ELB took to send the response back to the client                  |
| elb_status_code        | HTTP response code from ELB (e.g., 200, 504, 502)                        |
| target_status_code     | HTTP response code from the backend (if reached)                        |
| received_bytes         | Size of the request from client                                          |
| sent_bytes             | Size of the response back to client                                      |
| request                | Full HTTP request line (e.g., GET https://myapp/api/v1/data HTTP/1.1)   |
| user_agent             | User-agent string from the client                                        |
| ssl_cipher / ssl_protocol | TLS details for HTTPS connections                                     |
| trace_id               | Unique identifier for tracing (useful across services)                  |
| domain_name            | DNS name requested by client (for multi-domain use cases)               |
| chosen_cert_arn        | If HTTPS, the specific TLS cert used for the connection                 |

---

## Security & Operational Use Cases

| Use Case                      | How ELB Logs Help                                                              |
|-------------------------------|--------------------------------------------------------------------------------|
| DDoS or volumetric attacks    | Sudden spike in requests, especially with same IP or empty user-agent         |
| Malware scanning or probing   | Lots of 404s, strange URLs, or unknown endpoints being requested              |
| Credential stuffing / brute force | Repeated login attempts, same IP hammering auth endpoints               |
| Performance troubleshooting   | Long target processing times or many 504/502 errors                           |
| TLS inspection                | See which ciphers/protocols are used — enforce minimums (e.g., TLS 1.2+)     |
| Misconfigured targets         | ELB returns 5xx errors if the target group has unhealthy instances            |
| Compliance tracking           | Demonstrate who accessed what, from where, at what time                       |
| GeoIP analysis                | Source IPs can be mapped to regions for geo-anomaly detection                 |

---

## Enabling and Managing Logs

- You must explicitly **enable access logging**.  
- Logs are delivered to **S3 buckets** you configure (must grant permissions to `logging.elb.amazonaws.com`).  
- For **Application Load Balancers (ALB)**, logs are delivered every 5 minutes (or more frequently during high traffic).  
- Format differs slightly for **Network Load Balancers (NLB)** — includes source/destination IPs and ports, TCP flags, etc.

Example S3 path:  
`s3://your-logs-bucket/AWSLogs/your-account-id/elasticloadbalancing/region/year/month/day/...`

---

## Sample Log Line (ALB)

```text
http 2025-09-22T20:15:34.123456Z my-load-balancer 198.51.100.1:62000 10.0.2.45:80 0.001 0.005 0.000 200 200 1024 2048 "GET https://myapp.com/api/v1/resource
```

You can immediately see:

- IP address of client (`198.51.100.1`)  
- Path accessed (`/api/v1/resource`)  
- Backend latency (`0.005`)  
- Status code (`200`)  
- TLS cert used  
- Trace ID for linking to **X-Ray**, **CloudWatch**, or **app logs**

---

## Best Practices

- Send logs to **S3 + Athena** for querying — or forward to your **SIEM**  
- Enable **log file integrity validation** on S3 to detect tampering  
- Tag logs by environment (`prod`, `dev`) to segment queries  
- Correlate with **CloudTrail** + **VPC Flow Logs** to trace full attack paths  
- Automate alerts for high error rates, unusual agents, or large payloads using **Athena** or **CloudWatch Logs Insights**

---

## Pricing Model

Access logging itself is **free**.

But you pay for:

- **S3 storage costs** (can be minimized with lifecycle rules)  
- **Athena or CloudWatch queries**  
- **Optional delivery processing** (if using Lambda or Firehose)

Compared to other telemetry, ELB logs are **high-value with minimal overhead**.

---

## Real-Life Example

**Winterday’s security team** received a tip that `api.customerapp.biz` was being hit by a malicious scanner.

They enabled **ELB access logging** and used **Athena** to find:

- Repeated `404`s to `/admin/config.php`, `/phpinfo.php`, `/wp-login.php`  
- Originating IPs tied to known **exploit campaigns**  
- **TLS handshake downgrades** from outdated clients (hints of old bots)

They:

- Blocked the IPs via **AWS WAF**  
- Rotated affected tokens  
- Tightened their health check exposure  

— all within 30 minutes.  

**Without ELB logs?** It would’ve been guesswork.

---

## Final Thoughts

**ELB logs give you what the firewall can’t see** — the real behavior of real clients as they interact with your app.

You can’t secure what you don’t understand.  
You can’t troubleshoot what you don’t observe.  
And you can’t prove what you never logged.

If you want **visibility** into how users (or bots, or threat actors) interact with your application — this is where you start.  
Not in the backend.  
Not in the app server.  

At the edge.  
At the gateway.  

**Your load balancer sees everything.**  
**Turn on logging and listen to what it has to say.**
