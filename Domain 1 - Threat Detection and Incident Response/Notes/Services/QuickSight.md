# Amazon QuickSight

## What Is The Service

**Amazon QuickSight** is a cloud-native Business Intelligence (BI) and visualization service that enables you to create interactive dashboards, visual reports, and advanced analytics from diverse data sources — including **S3**, **Athena**, **Redshift**, **RDS**, and more.

For **security operations** and **cloud security engineering**, QuickSight transforms massive log volumes into intuitive dashboards, helping teams move from “log parsing” to **security insight**. You can:

- Track **IAM role usage** over time  
- Visualize **GuardDuty findings**  
- Chart **CloudTrail activity by region**  
- Spot **API anomalies or spikes**  
- Filter **VPC flow logs** for suspicious traffic patterns  

QuickSight becomes your **security visibility layer**, sitting on top of Athena, CloudTrail, or S3-based pipelines — without needing to manage BI infrastructure.

---

## Cybersecurity and Real-World Analogy

**Cybersecurity Analogy:**  
Imagine you're a CISO overseeing 100+ AWS accounts. Logs are flooding in: auth attempts, role assumptions, region access, S3 downloads. You can’t manually inspect it all.

QuickSight is your **Security Situation Dashboard** — distilling logs into visual patterns, surfacing spikes, and guiding your team’s next response.

**Real-World Analogy:**  
It’s like a NASA mission control dashboard. You don’t analyze every line of telemetry — you need the red light to flash when something’s off. QuickSight gives you **signal from the noise**, visually.

---

## Core Features and Capabilities

| Feature                | Explanation                                                                 |
|------------------------|-----------------------------------------------------------------------------|
| **Serverless & Scalable** | No infrastructure to manage. Scales from 1 user to thousands.            |
| **Pay-per-session pricing** | Only pay when someone views a dashboard.                            |
| **SPICE Engine**       | In-memory caching engine for fast filtering and querying.                   |
| **ML Insights**        | Built-in anomaly detection, forecasting, and natural language queries.      |
| **Embedded Dashboards**| Securely embed into internal tools or web apps.                             |
| **Row-Level Security** | Restrict data visibility per user/team/account.                             |
| **Federated SSO**      | Supports IAM federation, AD, and SAML (SSO integration).                    |

---

## QuickSight Workflow in a Security Context

### **Data Sources**

- CloudTrail, ALB, and VPC Flow Logs in **Amazon S3**  
- **Athena** queries over structured formats (Parquet, JSON, CSV)  
- **RDS/Redshift** for centralized SIEM-style storage  
- 3rd-party SIEM exports or data lake integrations

### **Data Preparation**

- Define datasets via QuickSight UI or CLI  
- Create **calculated fields** (e.g., `GeoLocationFromIP`)  
- Join datasets (e.g., GuardDuty findings + VPC logs)

### **SPICE Optimization**

- Load high-volume data into SPICE for speed  
- Schedule refreshes every 30 min or hourly

### **Visualizations**

- **Time series** (API calls over time)  
- **Heat maps** (geo-login attempts)  
- **Pie charts** (top users triggering GuardDuty)  
- **Tables** with conditional formatting (failed logins, deletions)

### **Publishing Dashboards**

- Share with **IAM** or **federated users**  
- **Embed** into internal SOC tools or portals  
- Use **Row-Level Security (RLS)** for scoped data views

---

## Security Use Cases

### **1. CloudTrail Event Visualization**
- Use Athena to query API call summaries
- Import into QuickSight to show:
  - Top-called APIs
  - Users doing `Delete*` operations
  - Usage in non-standard AWS regions

### **2. GuardDuty Findings Monitoring**
- Send findings to S3 (or via Kinesis Firehose)
- Visualize in QuickSight:
  - Top finding types
  - Malicious IPs
  - Finding frequency per account

### **3. IAM Role Assumption Patterns**
- Chart `AssumeRole` activity over time  
- Spot outliers (e.g., admin role used at 3 AM on a Saturday)

### **4. S3 Bucket Access Monitoring**
- Combine CloudTrail + access logs
- Track which roles/IPs access sensitive buckets

### **5. Login Activity Anomaly Detection**
- Correlate login failures across IPs, regions
- Use ML insights to detect **MFA brute-force spikes**

---

## SPICE vs Direct Query

| Engine         | Description                                                                 |
|----------------|-----------------------------------------------------------------------------|
| **SPICE**       | In-memory cache; fast, good for large data + filtering                     |
| **Direct Query**| Hits live source each time; real-time but slower and load-dependent        |

✅ **Use SPICE** for dashboards like daily GuardDuty trends  
✅ **Use Direct Query** for live dashboards (e.g., login spikes, ongoing attack monitoring)

---

## Security Features

### **IAM Integration**
- Access controlled via **IAM roles**, **SAML**, or **Active Directory**  
- Uses **STS** under the hood for secure session-based auth

### **Row-Level Security (RLS)**
- Share one dashboard, scope visibility  
- Example: Dev team only sees their own Lambda metrics

### **Audit Trail**
- QuickSight activity is logged in **CloudTrail**  
- Track dashboard views, exports, dataset modifications

### **Encryption**
- **Data at rest** encrypted using AWS **KMS**  
- **Data in transit** encrypted via **TLS**

---

## Pricing Overview

| Component        | Pricing Model                                                    |
|------------------|------------------------------------------------------------------|
| **Authors**      | Monthly per-user fee (can build dashboards, datasets)           |
| **Readers**      | Pay-per-session (30-min increments per active viewer)           |
| **SPICE Capacity** | Billed per GB/month (used for caching and speed)              |
| **ML Insights**  | Extra cost for built-in forecasting or anomaly detection        |

> **Tip:** Set usage alerts or view session consumption for cost monitoring.

---

## Benefits in a Cloud Security Architecture

- **Athena is your brain. QuickSight is your eyes.**  
- **Live reporting** to stakeholders without console access  
- **Secure embedded dashboards** for SOC or DevSecOps teams  
- **Multi-tenant dashboards** with RLS — useful in MSSP or multi-account orgs  

---

## Final Thoughts

QuickSight is the **visualization tier** of your AWS-native security pipeline.

While it doesn’t detect or ingest logs itself, it’s *essential* for:

- Communicating trends to leadership  
- Surfacing anomalies to responders  
- Allowing non-engineering stakeholders to engage with threat posture  

If you're already using Athena, CloudTrail, or GuardDuty, consider QuickSight your **real-time, secure, and shareable lens** into what’s happening in your AWS environment.

