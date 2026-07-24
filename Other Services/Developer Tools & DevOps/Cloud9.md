# AWS Cloud9

AWS Cloud9 is a browser-based IDE with an integrated terminal, backed either by an EC2 instance Cloud9 provisions in your account or by an existing server you connect to over SSH. Its security relevance has two sides. As a development tool it removes local credential sprawl: nobody stores access keys on a laptop, the environment can be placed in a private subnet reachable only through Systems Manager, and the whole workspace can be destroyed when a contractor leaves. As a piece of infrastructure it is a persistent EC2 instance with an attached EBS volume, an instance profile, a browser-accessible root shell, and the ability to receive forwarded user credentials, which makes it one of the more powerful footholds in an account and a favored persistence mechanism for an attacker who obtains developer permissions. Its credential model is unusual and worth understanding on its own: AWS managed temporary credentials inject the calling user's own permissions into the terminal, minus a fixed deny list, rather than using the instance profile. The most important operational fact is that AWS closed Cloud9 to new customers, so it now appears mainly in existing estates and in exam questions rather than in new designs. The thing to hold onto is that a Cloud9 environment is a long-lived EC2 instance with a shell, so its security posture is EC2's posture plus a credential injection mechanism unique to the service.

## How it works

- **Environment types.** An EC2 environment provisions and manages an instance in your account. An SSH environment connects to a server you already run, on-premises or elsewhere, with Cloud9 acting only as the editor. The security surface differs completely between the two.

- **No-ingress EC2 environments.** The recommended configuration connects through Systems Manager Session Manager rather than inbound SSH. The instance sits in a private subnet with no inbound rules and no public IP, and the instance profile carries `AWSCloud9SSMInstanceProfile`. This removes port 22 exposure entirely.

- **AWS managed temporary credentials.** By default Cloud9 injects credentials derived from the console user's own identity into the terminal, refreshed automatically. They carry the user's permissions minus a published deny list that blocks a set of IAM and organization-altering actions. Disabling AMTC makes the terminal fall back to the instance profile, which is the correct choice when the environment should have a fixed, scoped identity rather than the interactive user's full permissions.

- **Instance profile.** The EC2 instance carries a role separate from the injected user credentials. With AMTC on, the instance profile is mostly for SSM connectivity. With AMTC off, the instance profile becomes the environment's only identity, and scoping it is the primary control.

- **Environment sharing.** Environments can be shared with other IAM principals as read-only or read-write members. A read-write member gets a shell on the instance, which means sharing an environment is functionally granting shell access with whatever credentials that environment holds.

- **Persistent storage.** An EBS volume holds the workspace and survives stop and start. Whatever a developer clones, downloads, or writes persists there, including repository contents and any credentials saved to disk.

- **Auto-stop.** Environments stop after a configurable idle period, which limits cost and reduces the window in which a running instance is reachable, but the volume and its contents persist.

- **Network placement.** The instance lives in a VPC and subnet you choose with a security group you control. Private subnets with VPC endpoints for the AWS APIs the environment needs is the hardened configuration, with no NAT gateway if internet package installation is not required.

- **IAM controls.** `cloud9:CreateEnvironmentEC2`, `cloud9:CreateEnvironmentMembership`, and `cloud9:UpdateEnvironment` are the meaningful actions. Restricting environment creation, and especially membership creation, is what prevents a shell from being handed to another principal. `cloud9:EnvironmentUserArn` and tag conditions allow scoping.

- **Logging.** CloudTrail records Cloud9 control plane operations including environment creation, membership changes, and deletion, and records every AWS API call made from the terminal under the user's identity when AMTC is in use. It does not record shell commands. Session Manager logging can capture the session when connecting through SSM, which is the only path to command-level audit.

## Cloud9 versus adjacent development and access environments

| Option | Underlying compute | Identity in the terminal | Command-level audit | Persistence | Inbound network requirement | Current availability |
|---|---|---|---|---|---|---|
| AWS Cloud9 | EC2 in your account, or your own SSH host | Injected user credentials, or the instance profile | No, unless via Session Manager logging | EBS volume persists | None with no-ingress SSM | Closed to new customers |
| AWS CloudShell | AWS-managed environment or your VPC | Console session credentials | No | 1 GB home directory per Region | None | Available |
| EC2 with Session Manager | Your instance | Instance profile | Yes, full session logging | Instance storage | None | Available |
| Bastion host with SSH | Your instance | Whatever is configured | Only if you build it | Instance storage | Port 22 from somewhere | Available |
| SageMaker Studio | Managed instances, VPC-attached | Execution role | No | EFS-backed workspace | None | Available |
| Local IDE with IAM Identity Center | Developer's machine | Short-lived SSO credentials | No | Local disk | None | Available |
| CI/CD build environment | Managed build container | Build role | Pipeline logs the commands | Ephemeral | None | Available |

## What gets tested

- **No-ingress EC2 environments through Session Manager** are the answer for removing inbound SSH from a development environment, and they pair with a private subnet and VPC endpoints.

- **AWS managed temporary credentials carry the user's permissions.** If the requirement is that an environment operate with a fixed, narrow identity regardless of who opens it, the answer is disabling AMTC and scoping the instance profile.

- **Sharing an environment is granting shell access.** A read-write member can run commands with the environment's credentials, so `cloud9:CreateEnvironmentMembership` is the permission to restrict when the concern is lateral credential access.

- **Cloud9 is a persistence mechanism.** An attacker with developer permissions who creates an environment gets a long-lived instance with a shell and credentials in an account, and it looks like ordinary developer activity. Alerting on `CreateEnvironmentEC2` and on membership changes is the detection.

- **CloudShell versus Cloud9.** CloudShell is ephemeral, has no instance in your account, and is available for ad hoc CLI work. Cloud9 is a persistent instance with an IDE and is the answer only when a real development workspace with files and a debugger is required.

- **CloudTrail sees the API calls, not the commands.** Command-level audit requires connecting through Session Manager with session logging enabled to S3 or CloudWatch Logs.

- **The EBS volume retains everything.** Offboarding a contractor means deleting the environment, not just revoking their IAM access, since the volume holds repository contents and anything else written to disk.

- **Cloud9 is closed to new customers**, so a current-architecture question should be answered with CloudShell, Session Manager, a managed build environment, or a local IDE with IAM Identity Center rather than Cloud9.

- **SSH environments put Cloud9 outside your account boundary.** The editor connects to a host you manage, so AWS-side controls apply to almost none of it.

## Limitations

- Closed to new customers. Existing environments continue to work, but this is not a service to design new architecture around, and its feature development has effectively stopped.

- It is a persistent EC2 instance with all of that instance's attack surface: OS patching, package vulnerabilities, instance metadata, an EBS volume, and a shell that runs as a user who can escalate to root.

- AWS managed temporary credentials give the terminal the interactive user's permissions, so a developer with broad permissions gets a broadly permissioned shell, and the deny list blocks only a narrow set of actions.

- No native command-level logging. Without Session Manager session logging, there is no record of what was executed, only of the resulting API calls.

- The EBS volume is durable storage holding whatever the developer put there, with no lifecycle policy, no content visibility, and no encryption key distinction from any other volume unless you configure one.

- Auto-stop reduces cost but not exposure. The volume, the environment, and its membership persist while stopped, and starting it is a single click.

- Shared environments have no per-member scoping beyond read-only and read-write, so there is no way to give someone the editor without giving them the terminal.

- Instance-level hardening such as IMDSv2 enforcement, agent installation, and patching is not handled by Cloud9 and must be applied to the instance the same way it would be for any EC2 workload.

- SSH environments shift nearly all responsibility outside AWS, so the security benefits of the managed model do not apply to them at all.