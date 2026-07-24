# AWS Step Functions

AWS Step Functions is a serverless workflow engine that executes state machines defined in Amazon States Language, coordinating Lambda, ECS, Glue, DynamoDB, SageMaker, and roughly every other AWS API through optimized and SDK integrations. From a security perspective it has three properties worth understanding separately. First, every call it makes to an AWS service goes over the AWS API with TLS 1.2 or higher enforced by the endpoint, so in-transit encryption between steps is not a configuration you can get wrong, with the single exception of HTTP Task states calling third-party endpoints. Second, the execution role is the concentration point: a state machine orchestrating ten services holds a role with permissions across all ten, which makes it one of the highest-value roles in an account and a favored persistence and privilege-escalation target. Third, execution data including every state's input and output is retained by the service and visible in execution history, which means sensitive values flowing through a workflow are stored somewhere you may not have accounted for. The thing to hold onto is that Step Functions gives you encrypted transit for free and hands you two problems in exchange: an over-broad execution role and a durable record of everything that passed through the workflow.

## How it works

- **State machine types.** Standard workflows are durable, exactly-once, retained for 90 days, and support all service integrations including callback patterns. Express workflows are at-least-once, run up to five minutes, are priced per execution and duration, and send execution history only to CloudWatch Logs rather than retaining it in the service.

- **Service integrations.** Optimized integrations wrap common services with purpose-built behavior such as `lambda:invoke` and `sns:publish`. AWS SDK integrations expose almost every AWS API directly from a Task state. HTTP Task states call third-party HTTPS endpoints. All AWS-directed calls use the AWS API endpoints with TLS enforced; no plaintext option exists.

- **Execution role.** The state machine assumes a single IAM role for the entire execution. Every action across every state runs with that role's permissions, so the role is the union of everything the workflow can do. Scoping it requires enumerating each state's needs, and optimized integrations often require additional permissions beyond the obvious one, such as `iam:PassRole` for services that assume roles themselves.

- **Integration patterns.** Request-response returns immediately. Run a job (`.sync`) waits for completion, which requires additional EventBridge and describe permissions on the execution role. Wait for callback (`.waitForTaskToken`) pauses until an external caller returns a task token, which is how human approval steps work and which makes the task token a bearer credential that resumes the workflow.

- **HTTP Task and EventBridge connections.** Third-party calls use an EventBridge API destination connection holding the credential, stored in Secrets Manager and referenced rather than embedded in the state machine definition. This is the mechanism that keeps API keys out of the workflow definition, which is otherwise readable by anyone with `states:DescribeStateMachine`.

- **Data flow between states.** State input and output are JSON passed through the service. `InputPath`, `Parameters`, `ResultSelector`, `ResultPath`, and `OutputPath` filter and reshape it. Dropping sensitive fields before they propagate is the mechanism for keeping them out of downstream states and out of execution history.

- **Encryption at rest and customer managed keys.** State machine definitions and execution data can be encrypted with a customer managed KMS key rather than an AWS owned key, configured per state machine and per activity. This is what lets you crypto-shred execution history and what gives you CloudTrail visibility into decryption of workflow data.

- **Logging.** Execution history is retained by the service for Standard workflows and queryable through `GetExecutionHistory`. CloudWatch Logs delivery is configured separately with a log level of ALL, ERROR, FATAL, or OFF, and an `includeExecutionData` flag that controls whether state input and output appear in the logs. X-Ray tracing captures the call graph and timings without payload content.

- **CloudTrail.** Records control plane operations such as `CreateStateMachine`, `UpdateStateMachine`, and `StartExecution`, including who started an execution and with what input if input logging is enabled. State transitions within an execution are not CloudTrail events.

- **Network.** Step Functions is an API service with no VPC placement. It is reachable through an interface VPC endpoint, and `aws:SourceVpce` conditions restrict where executions can be started from. Tasks that need VPC access, such as reaching a private database, run in Lambda or ECS with their own VPC configuration.

- **Error handling.** Retry with exponential backoff and jitter, and Catch for routing failures to a handler state. From a security standpoint, retry configuration determines how a transient failure in a remediation workflow behaves, and unbounded retries against a throttled API can turn a workflow into a self-inflicted denial of service.

## Step Functions versus adjacent orchestration options

| Option | Coordination model | Identity used per step | State and payload retention | Encryption of stored state | Typical security fit |
|---|---|---|---|---|---|
| Step Functions Standard | Durable state machine, exactly-once | One execution role for the whole workflow | 90 days of execution history in the service | AWS owned or customer managed KMS key | Multi-step remediation and approval workflows |
| Step Functions Express | At-least-once, up to five minutes | One execution role | CloudWatch Logs only | Log group encryption | High-volume event processing |
| EventBridge rules | Single event to target | Target's own role or resource policy | None | Not applicable | Simple detect-and-act routing |
| Systems Manager Automation | Runbook with steps | Automation role, optionally per-step | Execution history in SSM | KMS on parameters | Instance and resource remediation |
| Lambda calling services directly | Code | Function execution role | Whatever you log | Log group encryption | Single-purpose logic |
| Managed Workflows for Apache Airflow | DAG scheduler | Environment execution role plus connections | Metadata database | KMS on the environment | Data pipeline orchestration |
| ECS or Batch job chains | Job dependencies | Task role per task | Task logs | Log group encryption | Containerized batch processing |

## What gets tested

- **The execution role is the least-privilege problem.** One role spans every state, so the answer to over-permissioning is decomposing the workflow, using nested state machines with narrower roles, or having Task states invoke Lambda functions that carry their own scoped roles.

- **`iam:PassRole` in the execution role is a privilege escalation path.** A workflow permitted to pass a role to another service can pass a more privileged one unless the permission is constrained with an `iam:PassedToService` condition and specific role ARNs.

- **In-transit encryption between Step Functions and AWS services is automatic and not configurable.** Questions offering a way to enable TLS on a service integration are testing whether you know it is already enforced. The exception is an HTTP Task, where the endpoint URL determines the protocol.

- **Credentials for third-party calls belong in an EventBridge connection backed by Secrets Manager**, never in the state machine definition, which is readable by anyone with describe permissions and is stored in plaintext absent a customer managed key.

- **Execution history contains state input and output.** If a workflow handles PII, credentials, or tokens, the answers are filtering with `ResultSelector` and `OutputPath`, setting `includeExecutionData` to false, and encrypting with a customer managed key. Express workflows shift that exposure into CloudWatch Logs instead.

- **Customer managed KMS key on the state machine** is the answer for controlling access to workflow definitions and execution data, and for making that data unreadable by disabling the key.

- **Task tokens are bearer credentials.** Anyone holding a token can call `SendTaskSuccess` and advance the workflow, so an approval step gated by a token needs the token treated as a secret and the caller's identity verified independently.

- **Standard versus Express** turns on durability and idempotency. Anything where a step must not run twice, such as a destructive remediation action, requires Standard. Express is at-least-once and needs idempotent targets.

- **Step Functions for security automation** is the answer when a response requires multiple steps, conditional branching, waiting for a human approval, or reliable retry. A single Lambda triggered by EventBridge is the answer when the response is one action.

## Limitations

- A single execution role for the whole workflow makes true least privilege awkward. The role is always the union of every state's requirements, and narrowing it means restructuring the workflow rather than tightening a policy.

- Execution history retention is 90 days for Standard workflows and is not configurable. It is neither long enough for most forensic retention requirements nor short enough to ignore for data minimization purposes.

- Express workflow history exists only in CloudWatch Logs, so if logging is off or the log group is deleted there is no record of what a workflow did.

- No VPC placement. Reaching private resources requires a Lambda or ECS task in the path, which adds a component with its own role, its own logging, and its own failure modes.

- Payload size is capped at 256 KB per state transition, which pushes large data into S3 and turns the workflow into a pointer-passing exercise where the actual data access control moves to bucket policy.

- HTTP Tasks leave the AWS trust boundary entirely. Certificate validation, endpoint verification, and response handling are yours, and a compromised third-party endpoint receives whatever the workflow sends it.

- Unbounded or aggressive retry configuration can amplify a transient failure into sustained load against a throttled API, and the retry happens with the execution role's permissions regardless of why the first attempt failed.

- The state machine definition is readable by any principal with `states:DescribeStateMachine`, which discloses the full architecture of a workflow including resource ARNs and integration targets.

- At-least-once delivery on Express workflows means duplicate execution is expected behavior, and a non-idempotent remediation step will occasionally run twice.