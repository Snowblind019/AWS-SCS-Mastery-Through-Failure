# AWS CloudWatch

## What Is The Service
CloudWatch is AWS’s observability backbone: metrics over time, logs at scale, alarms that act, and event routing that ties detection to automation. It’s the place where your systems turn behavior into signals you can measure, reason about, and react to. Services feed CloudWatch—EC2, ALB/NLB, Lambda, API Gateway, DynamoDB, RDS, EKS/ECS, VPC Flow Logs, Route 53, and more—so you can track performance, user experience, cost anomalies, and security-relevant activity without stitching ten tools together.  
Why it matters is simple: production drifts. Latency creeps. Costs spike quietly. A single misconfigured retry loop can turn a good day into a pager-fest. CloudWatch moves you from “it feels off” to “error rate jumped from 0.2% → 3% right after the Winterday release at 14:07; rollback now.” That observe → understand → act loop is how teams keep systems boring in the best possible way.

---

## Cybersecurity and real-world analogy
**Security Analogy.** Picture a SOC floor where:
- Metrics are motion sensors (low-volume, constant, trend-friendly).
- Logs are badge swipes + camera footage (detailed, sometimes messy, crucial for forensics).
- Alarms are the pager system (escalation rules, composite conditions).
- EventBridge is dispatch (routes the right incident to the right runbook and the right humans).

When Blizzard-API starts timing out on a downstream dependency at 03:12, CloudWatch is the thing that notices first, proves it with data, and kicks off a response—instead of waiting for a customer tweet.

**Real-World Analogy.** A well-instrumented car:
- Speedometer and gauges = metrics,
- Check-engine light = alarm,
- OBD-II scan logs = logs.

You can drive with just the speedometer; you survive incidents because you have all three.

---

## How It Works

### Metrics
- **Shape.** Namespace + Metric name + Dimensions (e.g., InstanceId, LoadBalancer, Service, Environment) + Timestamp + Value.
- **Sources.** AWS services publish many metrics by default; you add custom metrics via SDK/CLI, the unified CloudWatch Agent, or Embedded Metric Format (EMF) (emit JSON logs that auto-become metrics).
- **Resolution & retention.** Standard (1-minute) and high-resolution (down to 1 second) datapoints. Finer granularity for newer data, rolled up over time for older windows.
- **Metric Math.** Derive rates, deltas, percentiles, error budgets, SLO burn, multi-series expressions.
- **Anomaly Detection.** CloudWatch learns a baseline band; you alert on deviations, not brittle fixed thresholds—useful for traffic with daily/weekly cycles.
- **Metric Streams.** Continuous export of metrics (near-real-time) to Firehose → S3/lake/other tools for cross-platform analytics.
- **Design note.** Prefer percentiles (p95/p99) over averages for latency; use dimensions that map to how your team thinks (e.g., Service=Snowy-Checkout, Tier=Blizzard-Backend).

### Logs
- **Structure.** Log Groups ≈ app/service; Log Streams ≈ individual emitters (EC2 instance, Lambda container, pod).
- **Ingestion.** CloudWatch Agent (system/application logs), Lambda (stdout/err), EKS/ECS (Fluent Bit), API Gateway access logs, VPC Flow Logs, Route 53 Resolver query logs, and more.
- **Retention.** Per-group policy—keep short for chatty debug logs; keep longer for audit/security trails. Archive or export to S3 if you need multi-year retention.
- **Subscriptions.** Stream a log group to Lambda/Kinesis/Firehose for enrichment, PII scrubbing, or forwarding to OpenSearch/Splunk/etc.
- **Logs Insights.** Fast, ad-hoc queries over log groups: parse JSON, extract fields, filter, aggregate, compute percentiles, render quick charts. Ideal for incident triage and hunt-style investigations.
- **Structure tip.** Log structured JSON with consistent fields: timestamp, level, service, env, traceId, spanId, userId, requestId, route, durationMs, errorType, errorMessage. Your future self will thank you.

### Alarms
- **What they watch.** A single metric or a Metric Math expression (including anomaly bands).
- **States.** OK / ALARM / INSUFFICIENT_DATA.
- **Noise control.** Evaluation periods, datapoints to alarm, treat missing data (good/bad/ignore), and Composite Alarms (e.g., High 5xx AND High p95).
- **Actions.** Notify SNS/Incident Manager, scale via Auto Scaling, trigger SSM Automation/Runbook, flip a Route 53 health check, or hand off to EventBridge for richer workflows.
- **Design note.** Tie alarms to actions, not just notifications. A quiet, correct, actionable alarm is worth ten noisy ones.

### Dashboards
Team-visible, account/region-aware boards with:
- Time series, single-value tiles, Logs Insights widgets, metric math results,
- Simple account/region switchers for multi-env views.

Golden signals to pin: Latency, Traffic, Errors, Saturation, plus 1–2 business KPIs you actually care about (e.g., orders/min for Winterday-Orders).

### Synthetics (Canaries) & RUM
- **CloudWatch Synthetics.** Headless checks that run scripted journeys—login, cart, checkout, TLS/SSL expiry, API flows. Catch outside-in breakages your backend metrics miss.
- **CloudWatch RUM.** Real-user monitoring for web apps (Core Web Vitals, JS errors, page loads) to correlate “what users feel” with backend signals.

### Distributed Tracing (ServiceLens + X-Ray)
- **ServiceLens** ties metrics/logs to X-Ray traces so you can hop from a red metric to the exact slow/failed hop, then pivot to the relevant logs.
- **OpenTelemetry** collectors and the CloudWatch Agent can export spans to X-Ray for end-to-end request flow maps.

### Application / Container / Host observability
- **Unified CloudWatch Agent.** System metrics (CPU, memory, disk, net), app logs, StatsD/collectd.
- **Container Insights.** EKS/ECS node, pod, service metrics and performance maps.
- **Application Insights.** Opinionated setup for common stacks (.NET, IIS, Java, SQL Server) with auto-configured alarms and dashboards.

### Events & Automation (EventBridge)
Event rules match patterns (service events, API calls, state changes) or schedules and route them to Lambda, Step Functions, SSM Automation, queues, or ticketing.  
This is where detection becomes mitigation. Think: “When Blizzard-Backend error rate > 2% for 3 minutes, run Rollback-WinterPlaybook and page Snowy-OnCall.”

### Cross-Account / Cross-Region
Observability across accounts. Share metrics/dashboards/Logs Insights centrally while keeping publisher isolation. Use Metric Streams and log subscriptions to land data in a central lake.

---

## Pricing Models
High-level: you pay for custom metrics, log ingestion/storage, queries, alarms, synthetics/RUM, and streams. Control cost by setting retention, sampling chatty logs, promoting key counts to metrics, and exporting archives to cheaper storage.

| Feature                | What you pay for                               | Practical guidance                                      |
|------------------------|-------------------------------------------------|---------------------------------------------------------|
| AWS-provided metrics   | Included                                        | Great for base signals; don’t duplicate.                |
| Custom metrics         | Per metric / month (hi-res adds premium)        | Use EMF to bundle; retire unused names.                 |
| Metric Streams         | Firehose + downstream storage/compute           | Centralize to S3/Lake; compress/partition.              |
| Logs ingestion         | $ per GB ingested                               | Filter at edge (Fluent Bit); avoid debug in prod.       |
| Logs storage           | $ per GB stored / month                         | Set per-group retention; export/archive to S3/Glacier.  |
| Logs Insights          | $ per GB scanned                                | Narrow time ranges; parse JSON fields to reduce scan.   |
| Alarms                 | Per alarm; composite/anomaly priced differently | Prefer composite to cut noise; tune periods.            |
| Synthetics (Canaries)  | Per canary run                                  | Target critical user journeys only.                     |
| CloudWatch RUM         | Per user session                                | Sample thoughtfully; protect PII.                       |
| Dashboards             | A few free; then per-dashboard/month            | Keep one “Steady State,” clone for incidents.           |

---

## Operational And Security Best Practices
1. Start from user impact. Track latency, errors, traffic, saturation per service/tier. Add 1–2 business KPIs (Winterday-Orders/min, Blizzard-SignIns/min).
2. Prefer percentiles. p95/p99 reflect tail pain; averages hide it.
3. Use Anomaly Detection for cyclical workloads; Composite Alarms to gate pages behind multiple bad signals.
4. Log JSON with consistent fields; include trace IDs and request IDs. Emit EMF for counters you already know matter.
5. Tag aggressively: Service, Team, Environment, CostCenter. Use tags in dimensions and dashboard filters.
6. Retention with intent. Short for noisy ops logs; long for audit/security. Export long-tail archives to S3.
7. Stream security-relevant logs (VPC Flow, CloudTrail, Route 53 Resolver) to analysis; don’t rely only on ad-hoc queries.
8. Wire alarms to actions. Page via Incident Manager; run SSM Automation for restarts/rollbacks; open change tickets for non-urgent drift.
9. Test alarms like code. Induce a known failure in a sandbox; verify: metric → alarm → action → runbook → resolution.
10. Two dashboards. One Steady State for daily health; one Incident Template you clone and tune mid-outage.

## Real-Life Example
A Winterday deployment goes out at 14:00. Five minutes later, Snowy-Dashboard shows p95 latency climbing 220 ms → 600 ms and 5xx from 0.2% → 3% on Blizzard-Checkout. A Composite Alarm (High5xx AND HighP95Latency) engages Incident Manager and pages Blizzard-OnCall.

Snowy opens Logs Insights:
- Filters `service="Blizzard-Checkout" level="ERROR",`
- Parses JSON to pull `traceId`, `route`, `downstreamStatus`, `durationMs`,
- Sees timeouts to Winterday-Inventory.

ServiceLens map highlights the Inventory hop; X-Ray traces show a retry storm after the new client added a too-aggressive timeout. EventBridge rule triggers SSM Automation: roll back to Blizzard-Checkout@prev, bump Winterday-Inventory capacity by 2, and clear a bad cache key. Metrics stabilize within two minutes; alarm flips to OK. Post-incident, the team:
- Adds an Anomaly Detection alarm on downstream latency,
- Deploys a Synthetics canary that runs the exact cart-to-checkout path,
- Tightens client timeouts and retry jitter,
- Tags the runbook with Service=Blizzard-Checkout so alarms can auto-attach it.

---

## Reference Designs

### Golden Signals By Tier
| Tier / Signal   | Latency (p95)         | Errors (rate)        | Traffic (throughput)       | Saturation (resource) |
|-----------------|------------------------|-----------------------|----------------------------|-----------------------|
| Edge / ALB      | TargetResponseTime     | HTTP 5xx / Target 5xx | Requests/Second            | HealthyHostCount      |
| Service (Snowy) | Route duration (p95)   | 5xx by route          | Requests/Second by route   | Thread pool / queue depth |
| DB / Cache      | Query/Op duration      | Error/Throttle counts | QPS / Ops/sec              | CPU / Connections / Evict |

### Alarm Design
| Concern                 | Signal/Expression                         | Alarm Type        | Action                                 |
|------------------------|-------------------------------------------|-------------------|----------------------------------------|
| Sudden outage          | 5xxRate > 2% AND p95 > SLO                | Composite         | Page Snowy-OnCall, start Incident      |
| Slow burn latency creep| Anomaly on p95                            | Anomaly Detection | Create ticket, notify channel          |
| Backlog spike          | ApproxAgeOfOldestMessage (SQS) rising fast| High-res threshold| Scale out consumers via Auto Scaling   |
| Error budget burn      | SLO math over 1h window                   | Math Alarm        | Gate deploys; alert channel            |
| Canary flow broken     | Synthetics step failure                   | Canary Alarm      | Rollback via SSM Automation            |

---

## Final Thoughts
CloudWatch is most effective when it mirrors how your team makes decisions. The mechanics—metrics, logs, alarms, events—are just building blocks. The craft is choosing meaningful signals, shaping them into clear thresholds or learned baselines, and wiring them to reliable actions you trust at 2 a.m. Do that, and CloudWatch stops being “a page with graphs” and becomes part of how you run reliable, secure, quietly boring systems: calm during steady state, sharp when it matters, and precise when you look back and ask what exactly happened and why.