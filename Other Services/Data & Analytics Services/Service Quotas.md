# AWS Service Quotas

AWS Service Quotas is the central console and API for viewing the resource and rate limits that apply to an account, tracking usage against them, and requesting increases. Every service enforces its own quotas independently; Service Quotas is the single surface that reports them and brokers change requests. Its security relevance is mostly indirect but real in two directions. Defensively, a quota is a hard ceiling on how much infrastructure any principal can create, so it caps the blast radius of a compromised credential, a runaway automation loop, or a cryptomining deployment, and quota exhaustion alarms are often the earliest signal that something is creating resources it should not be. Operationally, quotas are an availability risk: a service hitting a limit fails closed, and a security control that cannot scale is a control that stops working during exactly the event it exists for, whether that is CloudTrail trail count, GuardDuty finding volume, KMS request rate, or Lambda concurrency for a remediation function. The thing to hold onto is that Service Quotas reports and requests, it does not enforce anything itself, so quotas are a ceiling you inherit rather than a policy you author.

## How it works

- **Quota scope.** Most quotas are per account per Region. Some are per account globally, and some are per resource, such as rules per security group or security groups per ENI. An increase applies only to the Region it was requested in, which is a recurring operational surprise in multi-Region deployments.

- **Adjustable versus non-adjustable.** Adjustable quotas can be raised through a request. Non-adjustable quotas are fixed by the service for architectural or safety reasons and no support case will move them. The Service Quotas console shows which is which per quota, and designing around a non-adjustable limit rather than requesting it is the correct instinct.

- **Applied quota value versus default.** Each quota has an AWS default and an applied value for your account. The applied value is what is enforced, and it can differ from the default after a prior increase, which means reading the default from documentation rather than the applied value from the API gives the wrong answer.

- **Usage tracking.** Service Quotas reports current utilization for quotas where the service publishes usage metrics, exposed through CloudWatch in the `AWS/Usage` namespace. Not every quota has usage data, so some limits are only observable through failed API calls.

- **CloudWatch alarms on utilization.** From the console or the API you can create an alarm on the ratio of usage to the applied quota value, typically firing at 70 or 80 percent. This is the mechanism for getting warned before a limit causes a failure, and it is also how quota consumption becomes a detection signal.

- **Increase requests.** Submitted through the console, CLI, or API with `RequestServiceQuotaIncrease` specifying a service code, a quota code, and a desired value. Some requests are approved automatically, others route to a support engineer for review. Request history is retrievable, which matters for change evidence.

- **Quota request templates.** With Organizations integration and a delegated administrator, a template defines quota increases automatically requested for every new account as it joins the Organization. This is how a landing zone ensures new accounts start with adequate limits for logging, security tooling, and workload capacity rather than tripping over defaults months later.

- **IAM surface.** `servicequotas:*` actions govern viewing quotas, requesting increases, and managing templates. `servicequotas:RequestServiceQuotaIncrease` is the meaningful one to restrict, since raising a limit removes a ceiling, and template management should be restricted to the delegated administrator.

- **Logging.** CloudTrail records quota increase requests, template changes, and association or disassociation of the Organizations template. `LimitExceeded` and `ThrottlingException` errors from the underlying service appear in that service's CloudTrail entries as failed calls, which is where quota exhaustion actually becomes visible in an investigation.

- **Trusted Advisor overlap.** Trusted Advisor's service limits checks report a subset of quotas with utilization and are available on Business and Enterprise Support, which predates and partially duplicates Service Quotas.

## Service Quotas versus other resource-constraining controls

| Control | What it constrains | Who sets it | Enforcement point | Prevents or reports | Scope |
|---|---|---|---|---|---|
| Service Quotas | Count and rate of resources per service | AWS, adjustable on request | The service itself, at API call time | Prevents, but the ceiling is AWS's | Account and Region |
| Service Control Policies | Which API actions are permitted at all | Organization management or delegated admin | IAM authorization in member accounts | Prevents | Organization, OU, or account |
| IAM policy conditions | Which resources a principal may create, with what attributes | Account administrator | IAM authorization | Prevents | Principal |
| AWS Budgets and budget actions | Spend, with optional automated response | Account owner | Billing data, then an action | Reports, then acts after the fact | Account or Organization |
| AWS Config rules | Whether existing resources match a desired state | Account or Organization | Evaluation after creation | Reports, with optional remediation | Account and Region |
| Resource-level service limits (for example Lambda reserved concurrency) | A specific service's internal allocation | You | The service | Prevents | Per function or resource |

## What gets tested

- **Service Quotas does not enforce your policy, it reports AWS's limits.** If a question asks how to stop a team from creating more than a set number of resources, the answer is an SCP or an IAM condition, not a quota. Quotas are the wrong tool for expressing intent.

- **Quotas cap blast radius.** Conversely, when a scenario describes a compromised credential spinning up large numbers of instances, the existing quota is what bounded the damage, and the remediation discussion includes reviewing whether increases were granted without justification.

- **Increases are per Region.** A multi-Region failover that fails because the standby Region has default limits is a classic scenario, and the answer is requesting increases in every Region in advance, ideally through a quota request template.

- **Quota request templates plus delegated administrator** is the answer for ensuring new Organization accounts start with adequate limits, tested alongside Control Tower and landing zone questions.

- **Non-adjustable quotas require architectural change.** If the limit cannot be raised, the answer is sharding across accounts or Regions, not a support case.

- **CloudWatch alarms on the `AWS/Usage` namespace** are the answer for proactive warning. Answers relying on catching `LimitExceeded` errors are reactive by definition.

- **Security tooling quotas matter during an incident.** Trail count per Region, KMS requests per second, Lambda concurrency, GuardDuty and Security Hub ingestion, and Config rule evaluations all have limits, and hitting one during a response degrades the control silently.

- **Restricting who can request increases** is the governance control, since an increase permanently removes a ceiling and there is no automatic mechanism to lower an applied value back down without a support request.

## Limitations

- It reports and requests, it does not enforce anything you define. There is no mechanism to set a lower internal limit through Service Quotas, so "cap this team at five instances" has to be built with IAM, SCPs, or tagging plus Config.

- Usage data is not available for every quota. Some limits can only be discovered by hitting them, which makes proactive alarming impossible for that subset.

- Increase requests are asynchronous and some require human review, so they are not a remedy during an incident or a scaling emergency. Headroom must be requested before it is needed.

- Applied values do not automatically propagate across Regions or to new accounts unless a quota request template is configured, and templates apply only to accounts joining after the template exists.

- Lowering an applied quota back down requires a support case. There is no self-service path to reduce a limit you previously raised, which means a generous increase is effectively permanent.

- Quota codes are opaque strings that differ per service and change rarely but not never, so automation referencing them needs the codes looked up rather than hardcoded from documentation.

- Not all services participate. Some publish quotas only in their own documentation or console, so Service Quotas is close to but not actually a complete inventory of every limit that can break a deployment.

- Rate quotas and throttling behave differently from count quotas. A count quota fails a creation call, while a rate quota returns a throttling error that a retrying client may mask entirely until the retry budget is exhausted, which delays detection.