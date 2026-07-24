# AWS CloudShell

AWS CloudShell is a browser-based shell launched from the AWS Console, pre-authenticated as the principal already signed in and preloaded with the AWS CLI, Python, Node.js, Git, and common tooling. The security argument for it is credential elimination: nobody needs long-lived access keys on a laptop, nobody configures a profile, and there is no key to leak, rotate, or find in a shell history file on a stolen machine. Every API call made from the shell runs under the console identity and lands in CloudTrail with that identity attached. The security argument against treating it as universally safe is the mirror image: CloudShell converts console access into full programmatic access. A role granted console permissions on the assumption that a human clicking buttons is slower and more constrained than a script gets a script anyway, and the shell has internet egress plus file upload and download, which makes it a data movement path that does not traverse any VPC you control. The thing to hold onto is that CloudShell does not grant any permission the principal did not already have, but it removes every practical friction between having a permission and exercising it at scale.

## How it works

- **Session model.** Launching CloudShell provisions a managed compute environment in the selected Region, running as a container with the signed-in principal's session credentials injected. Environments are per user, per Region, and are torn down after a period of inactivity.

- **Persistent home directory.** One gigabyte of storage per Region persists across sessions, holding files, shell history, and any tooling you installed. It is deleted after an extended period of inactivity, typically 120 days, and its contents are the one durable artifact of a session.

- **Credentials.** The environment receives temporary credentials derived from the console session. They are refreshed automatically and expire with the session. There is no static key material in the environment, but a user can retrieve the temporary credentials from the environment and use them elsewhere for their remaining lifetime.

- **Network modes.** By default CloudShell runs in an AWS-managed network with internet access and no reachability to your VPCs. VPC environments launch the shell into your own VPC with your subnets and security groups, which both gives access to private resources such as an RDS instance and lets you remove internet egress. VPC environments have no internet access unless your subnet routing provides it.

- **File transfer.** The console offers file upload and download to and from the environment, backed by presigned URLs. This is the data movement path, and it is what turns an S3 read permission into a local download without any bucket-level egress control noticing anything unusual.

- **IAM controls.** `cloudshell:CreateEnvironment` and `cloudshell:StartEnvironment` govern whether a principal can launch a shell at all. `cloudshell:PutCredentials` governs whether the environment receives forwarded credentials. `cloudshell:GetFileDownloadUrls` and `cloudshell:GetFileUploadUrls` govern file transfer, which can be denied independently to allow shell use while blocking data movement through it.

- **Restricting access.** A common pattern denies CloudShell entirely for high-privilege roles, denies it outside approved Regions with `aws:RequestedRegion`, or allows the shell while denying the file transfer actions. An SCP can apply any of these organization-wide.

- **Logging.** CloudTrail records CloudShell control plane events including `CreateEnvironment`, `StartEnvironment`, `PutCredentials`, `DeleteEnvironment`, `GetFileDownloadUrls`, and `GetFileUploadUrls`. It does not record the commands typed in the shell. What it does record is every AWS API call the CLI makes from the shell, attributed to the user's identity, which is where the actual activity trail lives.

- **Session Manager comparison in the same estate.** CloudShell reaches AWS APIs. Systems Manager Session Manager reaches your EC2 instances and containers with full session logging to S3 and CloudWatch. They solve different problems and CloudShell has no equivalent to Session Manager's keystroke logging.

## CloudShell versus other CLI access paths

| Path | Credential type | Where it runs | Command-level audit | Reaches private VPC resources | Data egress control |
|---|---|---|---|---|---|
| AWS CloudShell | Temporary, from the console session | AWS-managed environment or your VPC | No, only the resulting API calls | Only with a VPC environment | File transfer actions can be denied by IAM |
| Local CLI with access keys | Long-lived, stored on disk | User's machine | No | Depends on the machine's network | None |
| Local CLI with IAM Identity Center | Temporary, short-lived | User's machine | No | Depends on the machine's network | None |
| Bastion host with SSH | Whatever is on the host | Your EC2 instance | Only if you configure it | Yes | Host and network controls |
| Systems Manager Session Manager | Instance role plus caller identity | Your instance or container | Yes, full session logging to S3 and CloudWatch | Yes | Network controls plus session logs |
| CI/CD pipeline role | Temporary, per run | Managed build environment | Pipeline logs the commands | Depends on configuration | Pipeline artifacts and network controls |

## What gets tested

- **CloudShell grants no new permissions.** It exercises the caller's existing permissions. If a question implies CloudShell as a privilege escalation vector by itself, the framing is wrong. The real concern is that it makes existing over-permissioning trivially exploitable.

- **Console-only restrictions are not real restrictions if CloudShell is available.** Any control model assuming a console user cannot script against the API needs `cloudshell:CreateEnvironment` denied.

- **Denying file transfer while allowing the shell** is the answer when the requirement is CLI access without a bulk data egress path. `cloudshell:GetFileDownloadUrls` is the specific action.

- **VPC environments** are the answer for reaching a private RDS instance, ElastiCache cluster, or internal endpoint from a shell without provisioning a bastion, and for a shell with no internet egress.

- **CloudTrail does not log shell commands.** If a requirement is recording exactly what an operator typed, CloudShell cannot satisfy it, and Session Manager with session logging is the answer for instance access. For AWS API activity, the CLI's calls in CloudTrail are the trail.

- **Region restriction with `aws:RequestedRegion`** applies to CloudShell the same as anything else, and the persistent home directory is per Region, which is a data location consideration.

- **Eliminating long-lived access keys** is the security benefit to cite. CloudShell plus IAM Identity Center is the standard answer for "how do we stop developers storing access keys on laptops."

- **The persistent home directory holds whatever was written to it**, including query results, downloaded objects, and credentials a user manually saved. It is durable storage attached to a user identity and is worth treating as such.

## Limitations

- No command-level audit. CloudTrail sees the API calls but not the session, so a local script that reads files, transforms data, and writes one summarized output leaves a much thinner trail than the activity warrants.

- File upload and download bypass the network path you monitor. Data pulled from S3 into the shell and then downloaded through the console never traverses a VPC endpoint, a NAT gateway, or anything a flow log would show.

- The persistent home directory is durable storage with no encryption key you control, no lifecycle policy, and no visibility into its contents.

- Temporary credentials can be extracted from the environment and used elsewhere for their remaining validity, so the environment's isolation is not a credential containment boundary.

- One gigabyte per Region and modest CPU and memory make it unsuitable for real workloads. Long-running processes are terminated on idle timeout, so it is not a substitute for a build agent or a jump host.

- Default network mode has internet access and cannot reach your VPCs. VPC environments fix both but require subnet and security group configuration and have their own Region and quota constraints.

- Regional availability and preinstalled tooling vary, so a script depending on a specific tool version is not reliably portable across Regions.

- It is only as constrained as the identity behind it. An administrator with CloudShell is an administrator with a terminal, and the mitigation is reducing the identity's permissions rather than restricting the tool.