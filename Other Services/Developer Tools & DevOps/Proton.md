# AWS Proton

AWS Proton is a platform engineering service that lets a central team publish versioned infrastructure and deployment templates which application teams then instantiate through a narrow, parameterized interface. The platform team owns the CloudFormation or Terraform underneath; developers supply a handful of inputs such as a repository URL, a service name, and a tag, and Proton provisions the full stack including networking, compute, pipeline, and observability wiring. The security proposition is preventive standardization: if every service in the estate is provisioned from a reviewed template, then encryption, logging, tagging, IAM boundaries, and network placement are properties of the template rather than of whoever wrote the stack that week. It also solves the update problem, since a template version bump can be rolled out to every environment and service instance that uses it, which is how a control change actually reaches a hundred deployments. The counterweight is concentration: the Proton service role and the environment roles it uses to provision have broad permissions across accounts, and a template is executable infrastructure code that anyone with publish permission can push. The thing to hold onto is that Proton moves the security review from every deployment to a small number of template versions, which only works if template authorship is restricted more tightly than the deployments it replaces.

## How it works

- **Environment templates.** Define shared infrastructure that many services run inside: VPC, subnets, security groups, cluster, load balancer, IAM roles, logging destinations, and shared secrets. Published as versioned major and minor releases, with minor versions intended to be backward compatible.

- **Service templates.** Define a deployable workload plus its pipeline, such as an ECS service behind an ALB or a Lambda function fronted by API Gateway. A service template declares which environment templates it is compatible with, so a service cannot be deployed into an environment it was not designed for.

- **Schema files.** Each template carries a schema declaring its parameters, types, defaults, and constraints. The schema is the security boundary between the platform team and the application team: anything not exposed as a parameter cannot be changed by the consumer. Over-exposing a parameter, such as making a security group ingress rule or a bucket policy configurable, silently reopens the gap the template was meant to close.

- **Environments and service instances.** An environment is an instantiated environment template in a specific account and Region. A service is an instantiated service template deployed into one or more environments as service instances. Each records the template version it was created from, which is what makes drift visible.

- **Provisioning roles.** Proton assumes a service role to manage resources. For CloudFormation-based provisioning it uses an environment role in the target account, which enables cross-account deployment where the platform team's Proton lives in one account and provisions into member accounts. For self-managed provisioning, Proton opens a pull request against a repository and your own pipeline applies it, which keeps the credentials in your CI system rather than in Proton.

- **Components.** A mechanism letting a developer attach a limited amount of their own infrastructure to a service instance, such as an additional queue or table, defined in their own template and constrained by the environment's component role. This is the pressure valve for legitimate cases the template did not anticipate, and it is also the place where the paved road becomes optional.

- **Repository connections.** Template bundles and application source are pulled from CodeConnections-linked GitHub, GitLab, or Bitbucket repositories. Template sync keeps published template versions in step with a Git branch, which puts template changes through code review.

- **Version rollout.** Publishing a new template version marks existing environments and service instances as out of date. The platform team can update them individually, in bulk, or automatically for minor versions, which is the mechanism for propagating a security fix across the estate.

- **IAM surface.** `proton:CreateEnvironmentTemplateVersion`, `proton:CreateServiceTemplateVersion`, and the corresponding publish actions are the high-value permissions, since a template version is code that will execute with the provisioning role's authority. `proton:CreateService` and `proton:UpdateServiceInstance` are the developer-facing actions and are far lower risk by design.

- **Logging.** CloudTrail records template creation and publication, environment and service creation, updates, and deletions, and rollout operations. The resources Proton provisions generate their own CloudTrail entries under the provisioning role, so attribution runs from the Proton API call to the role to the resource creation.

## Proton versus adjacent standardization approaches

| Approach | What is constrained | Who owns the definition | Enforcement of the paved road | Update propagation | Drift visibility |
|---|---|---|---|---|---|
| AWS Proton | Full stack plus pipeline, via schema-limited parameters | Platform team | Strong, consumers only supply declared inputs | Template version rollout to all instances | Out-of-date flag per instance |
| AWS Service Catalog | Provisioned products from CloudFormation or Terraform | Central catalog admins | Strong for products, but does not own the pipeline | New product version, manual update per provisioned product | Provisioned product version |
| Terraform modules in a registry | Whatever the module covers | Whoever writes the module | Convention only, consumers can fork or bypass | Version pin bump per consumer | Terraform plan drift |
| CDK constructs | Whatever the construct covers | Construct author | Convention only | Package version bump per consumer | CDK diff |
| Service Control Policies | Which API actions are permitted at all | Organization admin | Absolute, but coarse | Immediate on policy change | Not applicable, prevents rather than tracks |
| AWS Config with conformance packs | Whether existing resources match rules | Central compliance team | Detective, with optional remediation | Immediate on pack update | Compliance status per resource |
| Control Tower controls | Account baseline and guardrails | Landing zone owner | Preventive and detective at account level | Control enablement | Control compliance status |

## What gets tested

- **Preventive standardization versus detective compliance.** Proton and Service Catalog prevent non-conforming infrastructure from being created. Config rules and Security Hub detect it afterward. A question emphasizing that a misconfiguration must never be deployed points at the preventive side, and SCPs are the absolute backstop underneath either.

- **The schema is the control.** Restricting what a developer can change means not exposing it as a parameter. An answer proposing IAM restrictions on the developer to prevent them changing template internals misunderstands the model, since they never had access to the internals.

- **Template publish permission is the privileged action.** A template version executes with the provisioning role's permissions, so separating who may author and publish templates from who may deploy services is the core least-privilege split.

- **Self-managed provisioning** is the answer when the requirement is that Proton must not hold credentials to provision infrastructure directly, since it opens a pull request instead and your pipeline applies the change.

- **Cross-account provisioning uses an environment role in the target account**, assumed by the Proton service role. This is the same pattern as most multi-account tooling and the same confused-deputy considerations apply.

- **Components are the escape hatch.** If a question describes developers adding unreviewed infrastructure alongside a governed service, the answer involves restricting or disabling components, or constraining the component role.

- **Version rollout is how a security fix reaches every deployment.** Contrast with Terraform modules or CDK constructs where each consumer must bump their own version pin, which is why estates using those drift.

- **Proton does not replace guardrails.** It standardizes what is deployed through it. Anything deployed outside Proton is unaffected, so SCPs and Config remain necessary, and the exam-preferred answer for absolute prevention is always the SCP.

## Limitations

- AWS Proton availability for new customers has changed since launch, so confirm current AWS guidance before designing new architecture around it rather than around Service Catalog, CDK pipelines, or a Terraform module registry with a platform team behind it.

- It governs only what is provisioned through it. Resources created by any other path, console, CLI, an existing pipeline, are invisible to Proton and unaffected by its templates, so it is not a guardrail in the SCP sense.

- The provisioning role is broad by necessity. A role capable of creating VPCs, IAM roles, load balancers, and clusters across accounts is a high-value target, and Proton's own IAM surface must be tighter than the infrastructure permissions it wields.

- Template authorship is infrastructure code with production authority. Without repository-backed template sync and code review, a published template version is an unreviewed change applied at scale.

- Schema design errors are silent. A parameter that should not have been exposed will be used, and there is no mechanism that flags an over-permissive schema the way a Config rule flags an over-permissive security group.

- Components weaken the guarantee by design. Every component attached to a service instance is infrastructure the platform team did not review, deployed inside an environment they are accountable for.

- Template updates can fail or partially apply across many instances, and a failed rollout leaves the estate in mixed versions, which is worse for auditability than a consistent old version.

- The abstraction is opinionated. Workloads that do not fit the environment-plus-service model, or that need infrastructure the template did not anticipate, either force template proliferation or drive teams to deploy outside Proton entirely, which is the failure mode that ends most golden-path programs.