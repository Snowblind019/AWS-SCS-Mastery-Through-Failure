# AWS Lambda

## What Is The Service

AWS Lambda is event-driven compute without servers to manage. You package code (or a container image), set memory/CPU, and Lambda runs it on demand when something happens: an HTTP request at API Gateway or ALB, a new object in S3, a message on SQS, a change stream in DynamoDB, a cron from EventBridge, and dozens more. Lambda scales automatically, bills per millisecond, and keeps you focused on business logic instead of capacity, OS patching, and idle waste.  
Why it matters: most workloads have spiky, bursty, or glue-style patterns—exactly where servers are either sleeping (wasting money) or you’re under-provisioned (hurting latency). Lambda turns that into pay-only-when-it-runs compute with built-in resilience, straightforward observability hooks, and a huge trigger ecosystem. For Snowy’s teams, Lambda becomes the connective tissue across services—ETL steps, webhooks, security automations, compliance fixers, fan-out processors, and “tiny brains” behind API endpoints—without the ceremony of a whole fleet.

---

## Cybersecurity And Real-World Analogy

**Security analogy.** A SOC with on-call analysts who wake only when a specific alarm fires. No one sits around burning budget. Lambda is that: it sleeps until a well-defined event (S3 object created, GuardDuty finding, Config violation) and then executes the exact playbook you wrote—least-privileged by IAM role, fenced by timeouts, VPC controls, and payload size. It’s detector-to-responder with almost no idle surface.  
**Real-world analogy.** A reliable valet switch on a machine: flip it (event), the tool spins up, does the job fast, then powers down. You’re not paying for the shop to stay open at 3 a.m.; when Blizzard-Checkout needs a PDF receipt rendered or Winterday-Inventory needs to normalize a CSV, Lambda appears, works, disappears.

---

## How It Works

### Execution Model (Quick Mental Picture)

- Invoke → Lambda allocates an execution environment (sandboxed micro-VM), loads your runtime + code, and calls your handler.
- That environment stays warm for a bit and may handle more invocations (reusing init state).
- On scale-out, Lambda creates more environments (horizontal).
- You control memory (which scales vCPU, networking) and timeout (max 15 minutes).
- `/tmp` ephemeral storage defaults to 512 MB; you can configure up to 10 GB.
- ARM64 and x86_64 are supported; ARM64 is often cheaper/faster-per-dollar if your deps are compatible.

### Cold Starts vs Warm

- **Cold start** = first run of a new environment: runtime start + init code + your handler.
- **Warm** = environment already initialized: only handler runs.  
You can mitigate cold starts via:
- Provisioned Concurrency (pre-warmed environments),
- Smaller packages, faster init paths,
- SnapStart (for Java): snapshot after init, rapid restore on invoke (with caveats for certain libraries/entropy).

### Triggers & Invocation Patterns

There are two broad modes:
- **Synchronous (request/response):** caller waits for the result. Good for APIs and user-facing latency.
- **Asynchronous (fire-and-handle):** Lambda queues internally and retries with backoff; you can route outcomes to Destinations or DLQ.  
Event source mappings (poll-based) connect streams/queues (SQS, Kinesis, DynamoDB Streams, MSK, Kafka) and handle batching, checkpointing, retries, partial failures.

**Common triggers by behavior**

| Trigger                         | Invoke Mode | Fan-out/Batching                            | Retries                    | Typical Use                               |
|---------------------------------|-------------|---------------------------------------------|----------------------------|-------------------------------------------|
| API Gateway / ALB               | Sync        | N/A                                         | Caller retries             | HTTP APIs, webhooks, lightweight backends |
| S3 (ObjectCreated)              | Async       | N/A                                         | 2 retries + DLQ/Destinations | ETL, thumbnails, metadata extraction      |
| EventBridge (rules / cron)      | Async       | N/A                                         | 2 retries + DLQ/Destinations | Schedules, automations, security responses|
| SQS                              | Poll (ESM) | Yes (1–10, up to 10k for FIFO w/ LBS)       | Until visibility timeout, then redrive | Queue workers, decoupled pipelines        |
| Kinesis / DynamoDB Streams      | Poll (ESM) | Yes (shard-based)                           | Retries until data expires | Ordered, at-least-once stream processing  |
| MSK / Kafka                     | Poll (ESM) | Yes                                         | Retries with backoff       | Kafka consumer without servers            |
| Cognito, CodeCommit, CloudWatch Logs | Async  | N/A                                         | 2 retries + routing        | Event-driven glue, enrichment             |

Destinations (for async): on success/failure, route metadata to SNS, SQS, EventBridge, or another Lambda for audit/compensation flows. DLQ is supported for async (SNS/SQS) when you just want failures parked.

### Packaging, Runtimes, and Dependencies

- **Zip functions:** upload code + deps (or use layers). Great for small artifacts.
- **Container images:** up to 10 GB images from ECR; use base Lambda runtimes or custom. Best for heavy native deps (FFmpeg, wkhtmltopdf, ML libs).
- **Runtimes:** Node.js, Python, Java, .NET, Go, Ruby; custom runtime via Runtime API if you need something else.
- **Layers:** share libraries across functions (mind versioning and size).
- **Extensions:** sidecar-like processes for telemetry/agents (e.g., OpenTelemetry Collector).  
**Tip:** keep init work minimal; lazy-load heavy deps inside the handler or use Init Phase wisely (benefits warms, costs colds).

### Concurrency, Scaling, and Backpressure

- Unreserved concurrency scales automatically; a per-region account limit caps total concurrent executions (raise as needed).
- Reserved concurrency on a function guarantees capacity and caps its max (isolate noisy neighbors).
- Provisioned Concurrency pre-warms N environments for stable latency (APIs, p99 SLOs).
- SQS & streams scale with batch size and shard/partition parallelism; use partial batch responses to avoid reprocessing good records when a few fail.  
**Failure/backpressure:**
- **SQS:** adjust visibility timeout > function timeout; use DLQ on the queue.
- **Kinesis/DDB Streams:** a single failing record blocks the shard; isolate by routing or fix-and-replay.

### Networking, Storage, and State

- **VPC access:** attach Lambda to subnets for private resources (RDS, ElastiCache); use VPC endpoints for AWS APIs without NAT.
- **`/tmp`:** ephemeral disk up to 10 GB per environment; good for transient files.
- **EFS:** mount a shared file system for larger state or models; mind latency and throughput modes.
- **State:** keep functions stateless between invocations; persist to DynamoDB, S3, RDS, or caches.
- **Env vars & secrets:** store configs in env vars (with KMS encryption) and retrieve secrets at runtime (Secrets Manager/Parameter Store) with a short TTL cache.

### Reliability, Retries, and Exactly-Once(ish)

- **Sync:** caller handles retries and idempotency keys.
- **Async:** built-in retries with exponential backoff; on exhaustion, send to DLQ or Destination.
- **Streams/queues:** at-least-once delivery; design idempotent handlers (dedupe table, idempotency keys, conditional writes).

### Observability

- **CloudWatch Logs:** every invocation’s logs go to a log group; use structured JSON for friendly search.
- **Metrics:** Invocations, Errors, Duration, Throttles, IteratorAge (streams), DeadLetterErrors, ConcurrentExecutions.
- **X-Ray / OpenTelemetry:** traces and spans; ServiceLens to see upstream/downstream.
- **Lambda Powertools (per language):** opinionated logging/metrics/tracing/idempotency/middleware that saves a lot of time.

### IAM and Security

- **Execution role:** what the function can do (read S3, write DynamoDB, call KMS, etc.). Keep it least-privilege; consider policy templates per service.
- **Resource policy:** who may invoke the function (API Gateway, EventBridge, another account).
- **Networking:** VPC + SGs for private access; VPC endpoints for egress to AWS APIs; restrict internet unless needed.
- **Code signing (optional):** enforce signed artifacts; runtime updates handled by AWS.
- **Timeouts & memory:** part of security-by-default—fewer stuck workers, bounded work.

---

### Cost Model

You pay for:
- Requests (per million requests), and
- Compute duration (GB-seconds), based on memory setting (which scales CPU/network proportionally). Provisioned Concurrency adds a warm capacity charge. ARM64 can reduce cost for compatible workloads. There’s also a free tier (requests + GB-seconds) per month.

**Practical levers to lower cost**
- Right-size memory for faster finish (often cheaper to give more RAM/CPU and finish sooner).
- Use ARM64 where possible.
- Batch with SQS/streams (do more work per invocation).
- Cache config/secrets and connections across warms.
- Offload long work to Step Functions or Fargate if consistently near 15-min cap.

### Handy Comparison Tables

**Invocation Behavior Cheat-Sheet**

| Source                | Ordering   | Delivery        | Failure Path          | Notes                                   |
|----------------------|------------|-----------------|-----------------------|-----------------------------------------|
| API Gateway / ALB    | N/A        | Sync            | Caller sees error     | Low-latency APIs; consider Provisioned Concurrency |
| S3 / EventBridge     | N/A        | Async           | Retries → DLQ/Destination | Great for decoupled pipelines        |
| SQS                  | N/A        | At-least-once   | Redrive to DLQ        | Use partial batch response and idempotency |
| Kinesis / DDB Streams| Per shard  | At-least-once, ordered per shard | Stuck on bad record | Monitor IteratorAge; isolate hot shards |
| MSK/Kafka            | Per partition | At-least-once, ordered per partition | Retries with backoff | Tune batch sizes and timeouts         |

**Choosing Packaging**

| Packaging     | Use When                         | Pros                                  | Cons                                    |
|---------------|----------------------------------|----------------------------------------|-----------------------------------------|
| Zip           | Small deps, fast deploys         | Simple, small artifact, quick cold start | Native deps are painful                |
| Container image | Heavy native libs, ML, toolchains | Familiar Docker flow, up to 10 GB     | Watch image size; slower pulls if large |

---

### Operational & Design Best Practices (Snowy’s Checklist)

- Design for idempotency. Use request IDs and conditional writes; store a “seen” key for dedupe.
- Keep handlers sharp. Minimal init, small artifacts, lazy-load heavy deps, prefer ARM64 when possible.
- Tune timeouts and visibility timeouts (SQS) correctly; never let the queue win.
- Use Provisioned Concurrency for p99-sensitive APIs; keep others on-demand.
- Batch smartly on SQS/streams; enable partial batch response.
- VPC endpoints for AWS APIs; private subnets for DBs; avoid NAT costs when possible.
- Emit structured logs; adopt Powertools for logging/metrics/tracing/idempotency.
- Least-privilege roles; separate read/write; scope KMS with `kms:ViaService`.
- Measure before optimizing. Look at Duration, InitDuration, Throttles, IteratorAge, ConcurrentExecutions.
- Know when not to use Lambda. If it’s CPU-bound for hours, needs >15 minutes, or requires sticky state—consider Fargate, ECS, or EC2.

---

## Real-Life Example (End-to-End, Winter Names)

**Scenario.** Winterday-Orders ingests e-commerce orders. Requirements: accept HTTP requests, validate, enqueue for processing, enrich with inventory data, and produce a PDF invoice—without running servers.

**API front door**  
API Gateway (HTTP API) → Lambda `orders-ingest` (ARM64, 1024 MB, 5s timeout).  
Validates payload, writes to SQS `orders-queue`, returns 202 Accepted.  
Provisioned Concurrency set to 10 for consistent p99.

**Order pipeline**  
Lambda `orders-consumer` (event source mapping from SQS, batch size 10).  
Reads batch, enriches from DynamoDB inventory, calculates totals, writes normalized record to S3 and DynamoDB orders.  
If one record fails, uses partial batch response to ack good ones and only retry the bad. Idempotency key = `orderId`.

**Invoices**  
S3 put event on normalized order → Lambda `invoice-renderer` (container image with wkhtmltopdf, 4096 MB, `/tmp` 4 GB).  
Renders PDF to `/tmp`, uploads to S3 `invoices/...`, sets pre-signed URL back on orders row.

**Observability & guardrails**  
Powertools for logs/metrics/tracing.  
Alarms on Throttles, Errors, SQS age of oldest, and InitDuration spikes.  
Execution roles least-privilege; KMS decrypt constrained with `kms:ViaService`.  
Destinations: on async failure, send to EventBridge bus `blizzard-failures` for triage (and page Snowy-OnCall if rate > threshold).

**Outcome:** No fleets to patch, linear scale with traffic, cost tied to actual usage, clear signals when things tilt.

---

## Final Thoughts

Lambda shines when you build small, well-aimed units of work tied to clear events, with idempotent behavior and tight IAM. Pair it with SQS/streams for decoupling, API Gateway/ALB for HTTP, and Step Functions when workflows get real. Keep the artifacts lean, the logs structured, the alarms pointed at user impact, and the network private by default. Do that, and Snowy’s serverless stack stays calm in steady state, sharp under load, and honest about what it’s doing—one quick, exact piece of work at a time.