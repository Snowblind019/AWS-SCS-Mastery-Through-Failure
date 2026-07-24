# AWS License Manager

AWS License Manager tracks and enforces software license entitlements across EC2, on-premises servers, and multi-account Organizations. You define a license configuration expressing a vendor's licensing terms, such as a core, socket, vCPU, or instance count limit plus rules about tenancy and Region, associate it with AMIs or instances, and the service counts consumption against it in real time. Its usual framing is cost and vendor-audit risk, but the security value is governance enforcement at launch time: with hard enforcement enabled, a launch that would breach the configuration is blocked outright, which makes License Manager one of the few AWS services that prevents a resource from existing rather than reporting on it afterward. It also produces the asset inventory that most compliance frameworks require, covering which licensed software runs where, on which host, in which account, and since when, including hybrid servers registered through Systems Manager. The thing to hold onto is that License Manager is a preventive control expressed as a counting rule, so its enforcement power comes from association with an AMI or launch path, and anything launched outside that path is invisible to it.

## How it works

- **License configurations.** The core resource. Specifies the license counting type (vCPU, core, socket, or instance), the entitlement limit, whether to enforce hard limits, and optional rules constraining allowed tenancy (shared, dedicated host, dedicated instance), minimum and maximum vCPUs or cores, and allowed license affinity for host-bound licenses.

- **Resource association.** A configuration is attached to AMIs, to launch templates, or to instances directly. Launches from an associated AMI count against the configuration automatically. Manual association covers instances already running, and automated discovery rules can associate resources matching specified criteria.

- **Hard versus soft enforcement.** With hard enforcement, a launch that would exceed the limit fails. With soft enforcement, the launch succeeds and the configuration is marked non-compliant, generating a notification. Hard enforcement is the preventive control, soft enforcement is the detective one.

- **Automated discovery.** Scans Systems Manager inventory data for software matching a product name and version, then associates matching hosts with a configuration. This is how shadow deployments and instances launched outside an associated AMI eventually get counted, but only if the host is SSM-managed.

- **Hybrid and on-premises tracking.** Servers registered as SSM managed nodes report inventory that License Manager counts, so the same entitlement pool can span cloud and data center. This requires the SSM agent, a hybrid activation, and an IAM role for the on-premises node.

- **Dedicated Host management.** License Manager can allocate and manage Dedicated Hosts on your behalf, including host affinity, which is what most BYOL Windows and Oracle terms actually require. It handles host allocation, placement, and release rather than requiring manual host management.

- **Organizations integration.** With a delegated administrator account, license configurations and usage aggregate across all member accounts. Cross-account resource discovery requires enabling the integration and a service-linked role in each account.

- **Seller-issued licenses and license conversion.** Marketplace and vendor-issued licenses can be received as grants into your account and redistributed to member accounts, with each grant tracked and revocable. License Conversion changes an instance's license type between AWS-provided and BYOL without relaunching, which matters because the wrong license type is both a cost and a compliance problem.

- **Linux subscriptions.** A separate view that discovers commercial Linux subscriptions such as RHEL across accounts and Regions, addressing the visibility gap for subscriptions purchased outside AWS.

- **IAM surface.** `license-manager:*` actions govern creating and modifying configurations, associating resources, and reading usage. The service uses service-linked roles for discovery and for Dedicated Host management. Separating who may create a configuration from who may modify or delete one is the meaningful split, since raising a limit or disabling enforcement silently removes the control.

- **Logging and notification.** CloudTrail records configuration creation and modification, resource association, and grant operations. License Manager publishes to an SNS topic on limit violations and can drive EventBridge rules for automated response. Usage data is queryable for reporting, and license configuration reports can be generated on a schedule to S3.

## License Manager versus adjacent governance controls

| Control | Enforcement point | Prevents or detects | Covers on-premises | Basis of the rule | Typical use |
|---|---|---|---|---|---|
| License Manager | EC2 launch path, via associated AMI or launch template | Prevents with hard enforcement, detects otherwise | Yes, through SSM inventory | Counted entitlement plus tenancy and sizing rules | Software licensing entitlement and vendor audit |
| Service Control Policies | IAM authorization for every API call in the Organization | Prevents | No | IAM action, resource, and condition | Organization-wide guardrails such as Region or service denial |
| IAM policy conditions | The calling principal's request | Prevents | No | Request context such as instance type or tag | Restricting what a specific role can launch |
| AWS Config rules | Resource state after creation | Detects, with optional remediation | Only for managed hybrid nodes | Desired configuration | Drift and compliance posture |
| Service Catalog | Provisioning through a curated product | Prevents by restricting the path | No | Approved product and template | Standardized, approved launches |
| Systems Manager Inventory | Agent reporting from the host | Detects | Yes | Installed software and patch state | Asset inventory and patch compliance |
| Cost anomaly and Budgets | Billing data | Detects after spend | No | Cost threshold | Financial control, not compliance |

## What gets tested

- **License Manager is the answer for entitlement counting and vendor audit evidence.** SCPs and IAM conditions can block instance types or Regions but cannot count how many cores of a product are in use across accounts.

- **Hard enforcement blocks the launch.** If a scenario requires that a non-compliant deployment never happen rather than be flagged, hard enforcement plus AMI association is the answer. A Config rule with remediation is the distractor, since it acts after the resource exists.

- **Enforcement only applies through associated launch paths.** An instance launched from an unassociated AMI is not counted until automated discovery finds it through SSM. Expect the gap to be the point of the question, with the remediation being discovery rules plus SSM coverage.

- **Dedicated Host management with affinity** is the answer for BYOL terms requiring the license to remain bound to specific physical hardware, such as Windows Server or SQL Server under license mobility restrictions.

- **Hybrid tracking requires SSM managed nodes**, meaning agent, hybrid activation, and an IAM role. Any answer implying agentless on-premises discovery is wrong.

- **Delegated administrator plus Organizations integration** is the answer for aggregating license usage across accounts. Per-account configurations that must be manually reconciled is the distractor.

- **Who can modify a configuration is the real control.** Raising a limit or switching from hard to soft enforcement is an unmonitored bypass unless the action is restricted by IAM and alerted on through CloudTrail and EventBridge.

- **Grants for seller-issued licenses** are the mechanism for distributing entitlements to member accounts, and revoking a grant is how you cut off consumption without touching the resources.

## Limitations

- Enforcement is EC2-centric. Containers, Lambda, and licensed software running inside a container image are not counted, so a workload that shifts to ECS or EKS falls outside the control.

- Automated discovery depends entirely on Systems Manager inventory. Instances without the agent, with the agent stopped, or in accounts where the integration is not enabled are invisible.

- The counting rules encode a simplified model of licensing terms. Real vendor agreements include clauses about virtualization, cluster-wide licensing, disaster recovery instances, and audit rights that License Manager does not model. Its output supports an audit response, it does not settle one.

- Hard enforcement can break legitimate deployments during a scaling event or a failover, since a launch blocked at the entitlement limit is indistinguishable to the application from any other capacity failure.

- License configurations are Region-scoped in their association behavior, and cross-Region visibility depends on the Organizations aggregation being enabled everywhere it needs to be.

- The service does not detect installed software on its own. It relies on AMI association or SSM inventory, so software installed post-launch on an unassociated instance is not counted until inventory reports it.

- Marketplace and third-party integration coverage is partial, and vendor-issued license grants only exist for products that support the mechanism.

- Reporting is usage-oriented rather than forensic. It shows current and historical consumption, not who attempted a blocked launch, which requires CloudTrail correlation.