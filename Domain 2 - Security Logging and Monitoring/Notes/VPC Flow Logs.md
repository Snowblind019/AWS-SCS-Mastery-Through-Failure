# VPC Flow Logs

## What Is The Service

VPC Flow Logs is an AWS feature that allows you to capture IP-level network traffic going into and out of network interfaces within your Virtual Private Cloud (VPC). These logs record metadata about the traffic — not the full packet contents, but details like source and destination IP, ports, protocol, traffic direction, and accept/deny status.  
You can enable Flow Logs at different levels:

- VPC-level (entire virtual network)  
- Subnet-level (specific segments)  
- Elastic Network Interface (ENI)-level (specific EC2, Lambda, Load Balancer, etc.)

Once captured, the logs can be published to:

- Amazon CloudWatch Logs (for real-time analysis, dashboards, alerts)  
- Amazon S3 (for long-term storage, bulk analysis, compliance)

These logs are critical for:

- **Network visibility**: You can see how traffic moves within your environment.  
- **Troubleshooting**: Diagnose “why can’t X reach Y?” types of issues.  
- **Security**: Detect unexpected communication (e.g., EC2 beaconing to known C2 servers, exfiltration, port scanning, etc.)

It's not deep packet inspection — but it's like having a syslog for your network layer.

---

## Cybersecurity Analogy

Imagine you're the head of security for a large corporate office with hundreds of doors and hallways. You can’t listen to every conversation or read every letter (that would be packet inspection), but you can install motion sensors and badge scanners on every entrance and exit.  
These sensors log:

- Who entered  
- What door they used  
- When they came in or out  
- Whether the door was locked or unlocked  

That’s what VPC Flow Logs does. It doesn’t record the contents of the communication, but it records:

- Source/destination IPs (who talked to who)  
- Ports and protocols (what kind of conversation it was)  
- Timestamps (when it happened)  
- Whether it was allowed or blocked  

This allows your security team to reconstruct movement, detect anomalies, and investigate suspicious behavior — even if they never saw the actual conversation.

## Real-World Analogy

Think of VPC Flow Logs like your cell phone call log. It doesn’t record the conversation itself, but it shows:

- Who you called  
- When the call started and ended  
- How long it lasted  
- Whether the call connected or failed  

If your phone bill showed a bunch of late-night calls to unknown international numbers, you'd know something was off — even without hearing what was said. That’s how Flow Logs help you detect weird traffic like port scans, unusual egress, or traffic to blacklisted IPs.

---

## How it Works / What It Captures

Here’s a sample log entry format (default format):

```text
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

Example:
```text
2 123456789010 eni-abc123 10.0.0.1 10.0.0.2 443 49152 6 10 840 1620225600 1620225660 ACCEPT OK
```

**Explanation:**

| Field         | Meaning                                       |
|---------------|-----------------------------------------------|
| srcaddr       | Source IP address                             |
| dstaddr       | Destination IP address                        |
| srcport       | Source port                                   |
| dstport       | Destination port                              |
| protocol      | Protocol number (e.g., 6 = TCP)               |
| action        | ACCEPT or REJECT                              |
| log-status    | OK, NODATA, SKIPDATA                          |
| interface-id  | The ENI through which traffic flowed          |
| start / end   | Time window for the log entry                 |

You can also use custom log formats to select the fields you care about — like instance IDs, subnet IDs, or TCP flags.

---

## Security Use Cases

| Use Case              | Description                                                             |
|-----------------------|-------------------------------------------------------------------------|
| Detect exfiltration   | Watch for large outbound traffic to external IPs or strange ports       |
| Spot lateral movement | Unexpected communication between unrelated subnets or EC2s              |
| Catch port scanning   | Burst of failed inbound connections from a single source IP             |
| IAM abuse             | EC2 communicating with unauthorized services or APIs                    |
| Geo-restrictions bypass | Outbound traffic to IPs in restricted countries or untrusted locations |
| Zero-day response     | Retrospective analysis after an IOCs list is released                   |

---

## Limitations of VPC Flow Logs

- **Not real-time**: Data is batched in intervals (~10 minutes delay to appear in CloudWatch/S3).  
- **No packet contents**: You cannot inspect actual payloads, headers, DNS, etc.  
- **Some traffic may be excluded**:
  - Traffic to/from Amazon DNS (if using default settings)  
  - VPC endpoints  
  - EC2 metadata (`169.254.169.254`)  

---

## Best Practices

- Enable VPC Flow Logs on **ALL** VPCs — production and dev/test.  
- Use **centralized logging architecture**: Send to a dedicated log account or SIEM.  
- Use **filters to monitor anomalies**: Detect outbound traffic spikes or failed connection attempts.  
- **Combine with GuardDuty or Athena**: Flow logs become actionable when joined with threat intel feeds.  
- Use **lifecycle policies**: Store logs in S3, then transition to Glacier to save cost while preserving forensic value.  
- **Tag ENIs**: Helps associate logs with app or environment.

---

## Pricing Model

Pricing depends on:

- Data volume captured (per GB)  
- Whether you're delivering to CloudWatch Logs or S3  
- Custom fields used (more fields = larger logs)

| Destination       | Pricing Notes                                          |
|-------------------|--------------------------------------------------------|
| **S3**            | Lower cost, better for archival or batch analytics     |
| **CloudWatch Logs** | Higher cost, better for real-time dashboards and alerts |

You pay for:

- Data ingestion  
- Optional metrics or alerts (CloudWatch charges)

---

## Real-Life Example

A team at **Winterday Financial** enabled VPC Flow Logs across their dev and prod environments after a suspicious incident involving EC2 traffic spikes.  
A few weeks later, a junior SOC analyst noticed strange outbound connections from a dev EC2 instance going to a known **crypto mining pool**.

Thanks to VPC Flow Logs:

- They confirmed the instance was communicating every few minutes to a **TOR exit node**  
- They traced the **ENI** to a specific instance created by a rogue **IAM user**  
- The logs showed **lateral connections** to RDS and EFS — luckily blocked by security groups  
- They generated an **IOC timeline** and correlated with **GuardDuty** and **CloudTrail**  

Flow Logs gave them the **map of motion**, even when no packets were left behind.

---

## Final Thoughts

VPC Flow Logs aren’t just a checkbox — they are your **network black box recorder** in the cloud.  
They don’t give you the entire conversation, but they tell you **who spoke to whom, when, and over what protocol**.

In security, that’s gold.

Used wisely, they allow you to:

- Hunt for threats  
- Prove activity  
- Trace compromise  
- Validate segmentation  
- Meet compliance  

And when paired with **CloudTrail**, **DNS logs**, and **threat feeds** — VPC Flow Logs become the **foundation for modern cloud forensics and network detection**.

> Don’t just turn them on.  
> **Instrument them. Analyze them. And most importantly — read the story they’re telling.**
