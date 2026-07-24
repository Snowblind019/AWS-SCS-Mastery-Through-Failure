# Infrastructure as Code

Infrastructure as Code means every cloud resource is defined in a versioned file and created by an automated process rather than by a human in the console. The security consequence is that infrastructure inherits the properties of software: it can be reviewed before it exists, scanned by a static analyzer, approved by someone other than its author, tied to a commit and a person, and rebuilt identically after a compromise. That last property matters more than it sounds, because the practical answer to "rebuild this environment cleanly after an incident" is only available to teams whose environment is fully described in code. IaC also relocates the security control point. Instead of detecting a public S3 bucket after Config flags it, you reject the pull request that would have created it, which is the difference between a mean time to remediate measured in hours and a misconfiguration that never existed. The cost of that shift is that the pipeline becomes the most privileged thing in the account, since whatever role applies the templates can create IAM roles, and the ability to create IAM roles is the ability to become anything. The thing to hold onto is that IaC moves security review to pre-deployment and moves the primary attack surface to the pipeline and its credentials.

## How it works

- **Declarative definition.** CloudFormation templates in YAML or JSON, Terraform configurations in HCL, or CDK code in a general-purpose language that synthesizes to CloudFormation. All describe desired state; the engine computes the difference against actual state and applies it.

- **Preview before apply.** CloudFormation change sets and `terraform plan` show the exact set of creations, modifications, and deletions before anything happens. Requiring a reviewed plan before apply is the control that catches a replacement of a production database or an unintended IAM policy widening.

- **State and drift.** CloudFormation tracks stack state internally and offers drift detection comparing deployed resources against the template. Terraform keeps state in a backend, typically an S3 bucket with encryption, versioning, and a DynamoDB lock table. That state file contains resource attributes and sometimes secrets in plaintext, which makes it a sensitive artifact requiring the same protection as a credential store.

- **Deployment role.** The pipeline assumes a role to apply changes. CloudFormation supports a service role on the stack so the deploying principal does not need the underlying resource permissions itself, only `cloudformation:*` on that stack, which is a meaningful privilege separation. Terraform has no equivalent and runs entirely with the credentials given to it.

- **Static analysis.** Checkov, cfn-nag, tfsec, KICS, Trivy, and cfn-guard scan templates and plans for insecure patterns: open security groups, unencrypted volumes, public buckets, disabled logging, wildcard IAM policies. Running these as a blocking CI step is what makes IaC preventive rather than merely documented.

- **Policy as code at deploy time.** CloudFormation Hooks and Guard rules evaluate resource configurations during stack operations and can block non-compliant changes, which catches what static analysis missed and applies even to changes made outside the pipeline. Terraform's equivalents are Sentinel or Open Policy Agent gates on the plan.

- **Secrets handling.** Secrets never belong in template files or variable files. CloudFormation dynamic references pull from Secrets Manager or Parameter Store at deploy time. `NoEcho` on a parameter hides it in the console and events but does not remove it from the state or from CloudTrail in all cases. Terraform marks variables sensitive but still writes resolved values to state.

- **Multi-account deployment.** CloudFormation StackSets deploy to many accounts and Regions with either self-managed roles or service-managed permissions through Organizations, and can auto-deploy to new accounts as they join. This is how a security baseline such as CloudTrail configuration, Config recorders, and GuardDuty enablement reaches every account.

- **Protection mechanisms.** Stack policies restrict which resources in a stack can be updated or replaced. Termination protection prevents stack deletion. Deletion policies and retention policies keep a resource such as a database or a log bucket alive when a stack is removed.

- **Enforcing the pipeline as the only path.** SCPs denying resource-creating actions except when the principal is the pipeline role, using `aws:PrincipalArn` conditions, is what turns "we use IaC" from a convention into an enforced boundary. Without that, console edits create drift the engine will silently revert or fight over.

- **Audit chain.** CloudTrail records the resource creation under the deployment role. Correlating that back to a person requires the commit history, the pull request approval, and the pipeline execution record, so the audit trail spans Git and AWS rather than living in either.

## IaC tooling and control-point comparison

| Tool | Language | State location | Privilege separation on deploy | Drift handling | Multi-account deployment | Policy enforcement at apply |
|---|---|---|---|---|---|---|
| CloudFormation | YAML or JSON | Managed by the service | Stack service role separates deployer from resource permissions | Native drift detection, no auto-remediation | StackSets with Organizations integration | Hooks and cfn-guard |
| AWS CDK | TypeScript, Python, Java, others | CloudFormation state | Same as CloudFormation | Same as CloudFormation | CDK Pipelines plus StackSets | Same as CloudFormation, plus construct-level defaults |
| Terraform | HCL | Backend you configure, typically S3 with DynamoDB locking | None, runs with the credentials it is given | `terraform plan` shows drift, apply reconciles | Workspaces and provider aliases, or Terraform Cloud | Sentinel or OPA on the plan |
| Pulumi | General-purpose languages | Pulumi service or self-managed backend | None natively | Refresh and preview | Stack references | CrossGuard policy packs |
| Service Catalog | CloudFormation or Terraform products | Provisioned product records | Launch constraint role separates user from resource permissions | Product version drift | Portfolio sharing across accounts | Template constraints |
| Console and CLI | None | None | Not applicable | Everything is drift | Manual | None |

## What gets tested

- **Preventive versus detective.** Scanning templates in CI blocks a misconfiguration before it exists. Config rules and Security Hub detect it after. A question emphasizing that the resource must never be created points at the pipeline gate or a CloudFormation Hook, and SCPs are the absolute backstop.

- **Terraform state is sensitive.** The backend bucket needs encryption with a customer managed key, versioning, blocked public access, restricted bucket policy, and a DynamoDB lock table. Any answer treating state as ordinary build output is wrong.

- **CloudFormation service role is the privilege separation answer.** It lets a deployer with only stack permissions apply a template that creates IAM roles, without that person holding IAM permissions directly.

- **Restricting resource creation to the pipeline** is an SCP with a `aws:PrincipalArn` condition, combined with removing standing console permissions. This is the answer to "developers keep making manual changes."

- **StackSets with Organizations service-managed permissions** is the answer for deploying a security baseline to every account including future ones, contrasted with per-account manual deployment.

- **Dynamic references to Secrets Manager** are the answer for keeping credentials out of templates. `NoEcho` is a distractor, since it obscures display without removing the value from the deployment path.

- **The pipeline role is a privilege escalation path.** A role that can create IAM roles and pass them can create an administrator. Constraining it with a permissions boundary applied to every role the pipeline creates is the standard mitigation, along with `iam:PassRole` conditions.

- **Drift detection reports, it does not fix.** If a question wants automatic reversion of a manual change, the answer is Config with an SSM Automation remediation, or reapplying the stack, not drift detection alone.

- **Deletion policies and termination protection** are the answers for preventing a stack teardown from destroying a log archive, an audit bucket, or a KMS key.

- **Signed commits and required reviews** are the controls tying an infrastructure change to a verified identity, since CloudTrail will only ever show the pipeline role.

## Limitations

- The pipeline role concentrates privilege. Whatever applies the templates must be able to create the resources described, which for a security baseline means IAM, KMS, and organization-level permissions, making the CI system a higher-value target than any single account administrator.

- IaC amplifies mistakes at the same rate it amplifies good configuration. A bad module referenced by forty stacks deploys the same flaw forty times, faster than any human could.

- Static analysis catches known insecure patterns and misses logic errors, misapplied conditions, and anything expressed through indirection such as a variable resolved at deploy time. A clean scan is not a security review.

- Drift is inevitable in practice. Emergency console changes during an incident, AWS-managed service-linked resources, and resources modified by other automation all produce differences the code does not know about, and reconciling them can be destructive.

- Terraform state contains resolved secret values in plaintext for many resource types regardless of how carefully the input was handled, so the state backend is effectively a secrets store.

- Code coverage is rarely complete. Most estates have a long tail of manually created resources predating the IaC adoption, and those are exactly the ones with no review history and the oldest misconfigurations.

- Rollback is not always possible. A stack update that fails after replacing a database or deleting a resource may leave the environment in a state neither the old nor the new template describes.

- Reviewing infrastructure changes requires reviewers who can read the diff and understand its security implications. A pull request approval from someone who cannot evaluate an IAM policy is a process control with no substance behind it.

- Module and provider dependencies are a supply chain. A third-party Terraform module or a CDK construct library executes with the pipeline's authority and can introduce resources the reviewer never read.