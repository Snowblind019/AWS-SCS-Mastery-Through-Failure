# AWS Elastic Beanstalk

Elastic Beanstalk is AWS's platform-as-a-service layer for web applications: you upload code, and it provisions an Auto Scaling group of EC2 instances, a load balancer, an S3 bucket for artifacts and logs, CloudWatch integration, and the IAM roles to tie them together. The critical thing to understand for security purposes is that Beanstalk is an orchestration wrapper, not an abstraction. Every resource it creates lives in your account, is visible in your console, appears in your Config recorder, and is your responsibility to harden. There is no managed boundary between your code and the host the way there is with Lambda or App Runner. Real EC2 instances with real instance metadata, real security groups, real EBS volumes, and a real instance profile sit underneath, and an application vulnerability gets an attacker a shell on a host holding credentials. What Beanstalk actually changes is who configured those resources: a developer running `eb create` provisions production infrastructure without writing a template or thinking about subnets. The thing to hold onto is that Beanstalk's defaults optimize for a working deployment rather than a secure one, so the security work is the same work you would do for EC2, plus catching the choices Beanstalk made for you.

## How it works

- **Applications, versions, and environments.** An application holds versioned deployable artifacts. An environment is a running instantiation of one version with a platform and a configuration. Environments are the unit of isolation, so separating dev, staging, and production means separate environments and ideally separate accounts.

- **Environment tiers.** A web server tier runs an EC2 Auto Scaling group behind a load balancer. A worker tier runs instances that poll an SQS queue through a local daemon, with no inbound load balancer, which is the more defensible topology when no HTTP ingress is needed.

- **Platforms.** Managed AMIs bundling an OS, a runtime, and a proxy such as nginx. AWS publishes new platform versions with OS and runtime patches, and platform versions reach end of support on a published schedule after which they stop receiving security updates.

- **Managed platform updates.** A configured maintenance window during which Beanstalk applies minor and patch platform version updates automatically, either immediately or with an instance replacement policy. This is off by default, and leaving it off means the fleet accumulates unpatched OS and runtime vulnerabilities indefinitely.

- **Two IAM roles.** The service role, assumed by Beanstalk itself to create and manage the environment's resources, and the instance profile attached to the EC2 instances. The instance profile is the one that matters, because it is what an application compromise inherits. The AWS-provided managed policies for it are broader than most applications need, and replacing them with a scoped customer-managed policy is the primary hardening step.

- **Networking.** Beanstalk can create resources in the default VPC with public instances if you let it. The correct configuration places instances in private subnets with a NAT gateway for egress, puts the load balancer in public subnets only when public ingress is genuinely required, and scopes the instance security group to accept traffic only from the load balancer security group.

- **Instance metadata.** IMDSv2 can be required through the environment configuration or a launch template. This matters more here than in most services, because a server-side request forgery in a web application on an instance with IMDSv1 enabled yields the instance profile's credentials directly.

- **Environment properties.** Beanstalk's mechanism for configuration, surfaced to the application as environment variables. They are stored in the environment configuration and readable by anyone with `elasticbeanstalk:DescribeConfigurationSettings`, and they appear in configuration history and in some logs. They are configuration storage, not secret storage. Secrets belong in Secrets Manager or Parameter Store, fetched at runtime by the application using the instance profile.

- **`.ebextensions` and Buildfile or Procfile.** Configuration files inside the deployment bundle that run commands, install packages, and modify the instance during deployment. They execute as root on every instance. This is where hardening such as file permissions, agent installation, and proxy configuration goes, and it is also an arbitrary code execution path controlled by whoever can push a deployment bundle.

- **Deployment policies.** All at once, rolling, rolling with additional batch, immutable, and traffic splitting. Immutable deploys launch a parallel Auto Scaling group and swap, which is the safest for rollback and the closest thing to blue/green within a single environment. Environment URL swap provides true blue/green across two environments.

- **The S3 bucket.** Beanstalk creates a bucket per Region holding application versions and, optionally, rotated logs. That bucket contains your deployment artifacts, which frequently contain configuration and occasionally credentials, so its policy and encryption configuration are part of the environment's posture.

- **Optional RDS integration.** A database provisioned inside the environment shares the environment's lifecycle, meaning terminating the environment deletes the database. External RDS referenced by connection details is the correct pattern for anything with persistent data.

- **Logging.** CloudTrail records environment creation, configuration updates, version deployment, and termination. CloudWatch Logs integration streams application logs and `eb-engine.log` when enabled, which is off by default. Enhanced health reporting adds an instance-level health agent and richer metrics.

## Beanstalk versus adjacent compute abstractions

| Option | Who patches the host | What a compromise reaches | Network placement | Deployment artifact | Secrets handling | Scaling model |
|---|---|---|---|---|---|---|
| Elastic Beanstalk | AWS publishes platform versions, you apply them | EC2 host plus instance profile credentials | Your VPC, your subnets, configurable | Zip or WAR bundle, plus `.ebextensions` | Environment properties are visible, use Secrets Manager | Auto Scaling group |
| Amazon EC2 | You, entirely | Host plus instance profile | Your VPC | AMI or user data | Your choice | Auto Scaling group |
| AWS App Runner | AWS | Container plus instance role | Managed, VPC connector optional | Container image or source repo | Secrets Manager and Parameter Store references | Managed, request-based |
| AWS Fargate | AWS patches the platform | Container plus task role | Your VPC | Container image | Secrets injected from Secrets Manager | ECS or EKS service |
| AWS Lambda | AWS patches the runtime | Function environment plus execution role | Optional VPC attachment | Zip or container image | Secrets Manager at runtime | Automatic, per invocation |
| AWS Proton or Service Catalog | Depends on the template | Depends on what is provisioned | Template-defined | Template plus parameters | Template-defined | Template-defined |

## What gets tested

- **Beanstalk resources are your resources.** Config rules, Security Hub, Inspector, and GuardDuty all see the underlying EC2 instances, security groups, and buckets. There is no exemption because Beanstalk created them, and remediation applies to the environment configuration rather than to the resource directly, since Beanstalk will otherwise revert manual changes.

- **The instance profile is the blast radius.** An application vulnerability yields the instance profile's permissions, so the answer to limiting compromise impact is a scoped customer-managed policy on the instance role, not a WAF rule.

- **IMDSv2 enforcement** is the specific control for server-side request forgery against a web application, which is exactly the workload Beanstalk hosts.

- **Environment properties are not secrets.** Anyone with describe permissions on the environment reads them. Secrets Manager or Parameter Store with `SecureString`, fetched at runtime, is the answer.

- **Managed platform updates are off by default.** A question about an environment running an unpatched runtime is answered by enabling managed updates within a maintenance window, and by tracking platform end-of-support dates.

- **Private subnets with a load balancer in public subnets** is the standard topology answer. Instances with public IPs is the misconfiguration, and the fix is environment network configuration, not a security group change alone.

- **`.ebextensions` runs as root during deployment**, so restricting who can deploy is equivalent to restricting who can run arbitrary commands on the fleet. `elasticbeanstalk:CreateApplicationVersion` and the deployment actions are the permissions to scope.

- **Beanstalk-managed RDS shares the environment lifecycle.** Any scenario mentioning data loss on environment teardown points at this, and the answer is an external RDS instance.

- **Immutable or blue/green deployment** is the answer when a bad deployment must be rollable back without downtime, and traffic splitting is the answer for canary validation.

- **The service role versus the instance profile** distinction appears in permission-troubleshooting questions the same way Lambda's execution role versus resource policy does.

## Limitations

- Not secure by default. The quick-start path produces public instances in the default VPC with broad managed policies on the instance profile, and nothing warns you.

- Patching is your responsibility to schedule. AWS publishes platform versions but does not apply them unless managed updates are enabled, and platform end-of-support silently leaves environments on unpatched runtimes.

- Full EC2 attack surface remains. SSH access, instance metadata, EBS volumes, host processes, and lateral movement from the instance are all in play, which is the tradeoff against Lambda or Fargate.

- Environment properties provide no secret protection, and there is no native mechanism equivalent to Fargate's secret injection from Secrets Manager.

- Configuration drift is fought rather than detected. Manual changes to Beanstalk-managed resources are reverted on the next deployment or configuration update, which makes ad hoc remediation unreliable and forces changes through the environment configuration.

- The service is a thin wrapper, so troubleshooting frequently requires understanding CloudFormation, Auto Scaling, and the load balancer underneath anyway, which erodes the abstraction's value without reducing the operational knowledge required.

- Worker tier environments run an SQS polling daemon on the instance, which is an additional component with its own configuration and its own failure modes compared to a native Lambda or ECS consumer.

- Beanstalk's own S3 bucket accumulates every application version ever deployed, including artifacts with embedded configuration, and its lifecycle is not managed for you.

- Platform customization beyond `.ebextensions` means building custom platform AMIs, at which point the operational burden approaches running EC2 directly.