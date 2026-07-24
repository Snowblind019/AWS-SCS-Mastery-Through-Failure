# AWS Lambda

AWS Lambda is event-driven compute where AWS owns the host, the runtime, and the scaling, and you own a handler function, an IAM execution role, and a set of triggers. There is no instance to harden, no OS to patch, and no long-lived process for an attacker to persist in, which removes an entire class of workload security problems. What replaces them is identity. A Lambda function is fundamentally a piece of code bound to an IAM role, so the security posture reduces to what that role can do, who is permitted to invoke the function, and what the code does with the credentials the runtime hands it through environment variables. Lambda is also the default remediation primitive in AWS security architecture, which means a function triggered by GuardDuty or Config frequently holds permissions to isolate instances, revoke sessions, or modify security groups, making it one of the most privileged and least visible pieces of code in an account. The thing to hold onto is that a Lambda function is an IAM role with a trigger attached, so invocation permission plus execution role scope is the entire security question.

## How it works

- **Execution environment.** Each concurrent invocation runs in an isolated Firecracker micro-VM with its own filesystem and memory. Environments are reused across invocations while warm, which means global state, cached credentials, and anything written to `/tmp` persist between invocations of different callers on the same environment. That reuse is the reason `/tmp` must never hold sensitive data past the handler and why per-request state must not live in globals.

- **Execution role.** Assumed by the service, with temporary credentials injected into the environment as variables. Everything the function does to AWS uses this role. Scoping it to specific resource ARNs, and constraining KMS usage with `kms:ViaService`, is the primary least-privilege work.

- **Resource-based policy.** Separate from the execution role, this governs who may invoke the function. Service principals such as `s3.amazonaws.com`, `events.amazonaws.com`, and `apigateway.amazonaws.com` need explicit statements, guarded with `aws:SourceArn` and `aws:SourceAccount` to prevent a confused deputy where another account's bucket or rule triggers your function.

- **Invocation modes.** Synchronous, where the caller waits and handles its own retries. Asynchronous, where Lambda queues internally, retries twice with backoff, and routes outcomes to a destination or dead-letter queue. Event source mappings, where Lambda polls SQS, Kinesis, DynamoDB Streams, MSK, or Kafka on your behalf using the execution role, handling batching, checkpointing, and partial batch failures.

- **Environment variables and secrets.** Environment variables are encrypted at rest with an AWS managed key by default or a customer managed key. They are visible in the console and through `GetFunctionConfiguration` to anyone with that permission, so they are configuration storage, not secret storage. Secrets belong in Secrets Manager or Parameter Store, retrieved at runtime, ideally through the Parameters and Secrets extension which caches them locally.

- **VPC attachment.** A function attached to subnets gets Hyperplane ENIs and can reach private resources such as RDS or ElastiCache. It then has no internet access unless you provide a NAT gateway, and reaching AWS APIs privately requires interface endpoints. VPC attachment is how you both reach private data and constrain egress.

- **Code signing.** A code signing configuration backed by AWS Signer requires deployment packages to carry a valid signature from a trusted publisher, and can be set to reject or warn on expired or revoked signatures. This is the supply chain control preventing an unauthorized package from being deployed to a function.

- **Concurrency controls.** Reserved concurrency both guarantees and caps a function's concurrent executions, which isolates it from noisy neighbors and, more importantly, bounds the damage a runaway or attacker-triggered function can do. Setting reserved concurrency to zero is the fastest way to stop a function entirely during an incident. Provisioned concurrency pre-warms environments for latency.

- **Layers and extensions.** Layers share libraries across functions. Extensions run alongside the function for telemetry, secret caching, or agents. Both are code executing inside your environment with access to the execution role's credentials, so a layer from an untrusted publisher is a supply chain risk with the same blast radius as the function itself.

- **Packaging.** Zip archives up to 50 MB compressed or 250 MB uncompressed, or container images up to 10 GB from ECR. Container images bring image scanning through ECR and Inspector into the picture, which zip packages lack.

- **Logging and tracing.** CloudWatch Logs receive every invocation's output, with the log group encryptable by a customer managed key. CloudTrail records control plane operations including `UpdateFunctionCode`, `UpdateFunctionConfiguration`, and `AddPermission`, and can record `Invoke` as a data event when enabled. X-Ray captures the call graph. Lambda Insights adds host-level metrics.

- **Runtime management.** AWS patches the managed runtime. The runtime version update policy can be set to auto, function update, or manual, which controls whether a runtime patch reaches your function immediately or on your next deployment. Deprecated runtimes eventually stop receiving security patches and then block updates entirely.

## Lambda versus adjacent compute options

| Option | Patching responsibility | Identity model | Max execution time | Network placement | Supply chain scanning |
|---|---|---|---|---|---|
| AWS Lambda | AWS patches the runtime, you own dependencies | Execution role plus resource policy | 15 minutes | Optional VPC attachment, no internet by default when attached | ECR and Inspector for container images, none for zip |
| AWS Fargate | AWS patches the platform, you own the image | Task role plus task execution role | Unbounded | Always in a VPC | ECR image scanning, Inspector |
| Amazon ECS on EC2 | You patch the host and the image | Task role plus instance profile | Unbounded | Always in a VPC | ECR plus host scanning |
| Amazon EC2 | You patch everything | Instance profile | Unbounded | Always in a VPC | Inspector on the instance |
| AWS App Runner | AWS patches the platform | Instance role | Unbounded | VPC connector optional | ECR scanning |
| Step Functions | Not applicable, no code | One execution role per state machine | One year for Standard | None | Not applicable |

## What gets tested

- **Execution role versus resource-based policy.** The execution role is what the function can do. The resource policy is who can invoke it. Questions consistently offer the wrong one as a distractor, particularly for cross-account invocation, which requires a resource policy statement.

- **`aws:SourceArn` and `aws:SourceAccount` on service principal grants.** A permission granted to `s3.amazonaws.com` without a source condition lets any account's bucket invoke the function, which is the canonical Lambda confused deputy.

- **Environment variables are not secrets.** Anyone with `lambda:GetFunctionConfiguration` reads them, encrypted at rest or not. The answer for credentials is Secrets Manager or Parameter Store retrieved at runtime, with the execution role scoped to the specific secret ARN.

- **Reserved concurrency set to zero** is the containment answer for stopping a compromised or misbehaving function immediately without deleting it and losing forensic evidence.

- **VPC attachment removes internet access.** A function that suddenly cannot reach a third-party API after being attached to a VPC needs a NAT gateway, and a function that needs AWS APIs privately needs interface endpoints. This exact scenario recurs.

- **Code signing is the answer for ensuring only approved artifacts deploy.** Combined with `lambda:UpdateFunctionCode` restrictions, it addresses the "an attacker with deploy permissions replaced our function code" scenario.

- **CloudTrail `Invoke` is a data event**, off by default. Knowing who invoked a function requires enabling Lambda data events on the trail.

- **`UpdateFunctionCode` and `AddPermission` are the persistence actions to alert on.** Modifying a privileged remediation function's code, or adding an invoke permission for an external account, are both quiet ways to establish control.

- **Layers and extensions execute with the execution role.** A question about third-party observability agents or shared libraries is testing whether you recognize them as code running inside your trust boundary.

- **`kms:ViaService` condition** scopes a function's KMS permissions to use through a specific service, which is the standard hardening for an execution role that needs decrypt for one purpose only.

- **Runtime deprecation.** An unmaintained runtime stops receiving patches, which is a vulnerability management finding, and the answer is migrating the function, not requesting an exception.

## Limitations

- Fifteen minute maximum execution time. Longer work belongs in Step Functions orchestrating multiple invocations, Fargate, or Batch.

- Environment reuse means state leaks between invocations if the code is careless. Globals, connection pools, cached credentials, and `/tmp` contents all persist, and two different tenants' requests can land on the same warm environment.

- Environment variables are readable by anyone with configuration read permission, so encryption at rest provides no protection against the most likely disclosure path.

- Zip-packaged functions have no built-in vulnerability scanning. Dependency risk must be handled in the build pipeline, unlike container images which get ECR and Inspector coverage.

- Package size limits of 50 MB zipped and 250 MB unzipped push heavy dependencies toward container images or layers, and layers themselves count toward the unzipped limit.

- Attaching to a VPC introduces ENI management, subnet IP consumption at scale, and the loss of default internet egress, all of which surprise teams that attach a function to reach one database.

- At-least-once delivery from event source mappings and asynchronous retries mean duplicate execution is normal, so any function taking a destructive or non-idempotent action needs its own deduplication.

- A single failing record blocks a Kinesis or DynamoDB Streams shard until it expires or is skipped, so a poison record can halt an entire stream's processing.

- Account-level concurrency is a regional quota shared across all functions. One function scaling aggressively throttles everything else, including security remediation functions, unless reserved concurrency isolates them.

- Cold starts add latency that is worst for VPC-attached functions with large packages, and provisioned concurrency solves it at a standing cost that partly defeats the pay-per-use model.