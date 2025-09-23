# **AWS CloudWatch — Monitoring & Observability for Your AWS**

**AWS CloudWatch** is a **monitoring and observability** service. It watches over your AWS environment (and even your applications), collecting **logs**, **metrics**, and **events** from your resources. You can set **alarms**, **dashboards**, and **automated responses** based on those insights.  
Think of it as your **central nervous system** for operations visibility — performance, availability, and even security trends. **Without CloudWatch, you’re flying blind.**

---

## **Cybersecurity & Real-World Analogy**

### **Cybersecurity Analogy**
Amazon has named this service quite well with a self explanatory name which is nice, but I still like using my analogies to remember everything. In cybersecurity, **CloudWatch is like a SIEM-like tool.**

- In a security stack, you might use something like **Splunk** or **QRadar** to collect and visualize logs, metrics, and events.  
- **CloudWatch** gives you that **within AWS** — collecting everything from **EC2 CPU usage**, to **Lambda errors**, to **API calls**, and more.  
- If **GuardDuty** gives you **alerts**, **CloudWatch** gives you the **telemetry** that leads up to those alerts.

### **Real World Analogy**
For a real world analogy, imagine you live in a high-tech house:

- **Thermostat shows temperature** = **CPU Usage**  
- **Water meter tracks leaks** = **Error Logs**  
- **Door sensors track open/close** = **Event Logs**  
- **Cameras show motion** = **Metric dashboards**  
- **You can get a text message if someone opens the door at 2 AM** = **Alarms + Notifications**

**CloudWatch acts just like this — but for your AWS infrastructure instead of your house.**

---

## **How It Works**

**CloudWatch has 3 core components** (and two more pillars you’ll use constantly):

### **1) Metrics (Numbers)**
- Things like **CPU usage, memory, network in/out, Lambda invocation time**, etc.  
- Automatically collected from **AWS services** (e.g., **EC2, RDS, Lambda**).  
- You can create **custom metrics** too.  
- You can **graph** them on a **dashboard**, or **set alarms** on thresholds.

> “**Notify me if EC2 CPU usage goes over 85% for 5 minutes**”

### **2) Logs (Text)**
Collects **log data** from:

- **EC2 instances** (via **CloudWatch Agent**)  
- **Lambda** function logs  
- **API Gateway** access logs  
- **VPC Flow** logs  
- **Custom application** logs

You can **search logs**, **filter** them, and **trigger alerts**.  
This is **log aggregation** — similar to **Logstash** or a basic **SIEM** pipeline.

### **3) Events (Things that happen)**
**EventBridge** (formerly **CloudWatch Events**) captures:

- **State changes**  
- **Cron jobs**  
- **Service events** (like “EBS volume created”)

You can route events to **Lambda**, **SNS**, **SSM**, etc.

> “**If a new EC2 is launched in us-west-2, tag it with 'Production' and notify Slack.**”

### **4) Alarms**
Set **thresholds** for any metric (even **custom** ones). Alarms can trigger:

- **Email/SMS** (via **SNS**)  
- **Auto-scaling**  
- **Lambda functions**  
- **EC2 actions** *(stop, reboot, terminate)*

**Example:** “**Terminate this instance if memory stays above 90% for 10 mins**”

### **5) Dashboards**
Fully customizable dashboards with **metrics, graphs, KPIs**.  
You can monitor **multi-account, multi-region** setups.  
This is your **visual overview** of **system health and performance**.

---

## **Other Important Details**

### **CloudWatch Agent (For EC2 + On-Prem)**
For **EC2 instances** or **hybrid/on-prem** setups, you install the **CloudWatch Agent** to collect metrics/logs **beyond the defaults**.

**Example:** Collect **disk usage**, **memory pressure**, **app-specific logs** from an EC2 instance.

### **Custom Metrics**
You can push your own data points (from apps, containers, etc.) to **CloudWatch**.

```bash
aws cloudwatch put-metric-data --namespace "AppStats" --metric-name PageLoadTime --value 3.4
```

## **Unified with Security Tools**

**CloudWatch Logs** can be a source for:

- **GuardDuty**
- **Security Hub**
- **Inspector** _(for agent-based logs)_

Central to building **detection pipelines** in security workflows.

---

## **Pricing Models**

**CloudWatch pricing has four main buckets:**

| **Feature**            | **Pricing Notes**                                                                 |
|------------------------|------------------------------------------------------------------------------------|
| **Metrics**            | Free for default metrics. _Custom metrics cost extra_ (by the number and granularity). |
| **Logs**               | Charged by **ingestion (GB)** and **storage (GB/month)**.                          |
| **Events (EventBridge)** | Some features free, others priced per **event rule** or **invocation**.          |
| **Dashboards**         | **3 free dashboards**, then charged **per dashboard/month**.                       |
