# AWS Migration Hub

AWS Migration Hub is the coordination and tracking layer for migrations to AWS. It does not move any data itself; it aggregates discovery data and migration status from the tools that do, including Application Migration Service, Database Migration Service, and supported partner tools, into one dashboard spanning a whole migration program. Its security relevance is evidentiary and organizational rather than protective. A large migration is a period of unusual risk, sensitive data crossing network boundaries, temporary access grants, servers running in a half-migrated state, and Migration Hub is where you establish what is in scope, what has moved, when it moved, and what depends on what, which is exactly the record a compliance auditor asks for when regulated data changes location. Its dependency mapping also prevents the ordering mistakes that cause insecure intermediate states, such as a backend cut over before its database or a service exposed before its access controls are reapplied. The thing to hold onto is that Migration Hub is a read-and-track control plane whose value is a coordinated, auditable record of a migration, not a data mover and not an enforcement point.

## How it works

- **Home Region.** Migration Hub operates from a single home Region you select, where all discovery and tracking data is aggregated, even though the migrations themselves target many Regions. That home Region is a data location decision, since discovery data describing your on-premises estate concentrates there.

- **Discovery.** AWS Application Discovery Service feeds the Hub. The agent-based path installs a discovery agent on each server to collect configuration, performance, running processes, and network connections. The agentless collector runs as a VM gathering VM inventory and utilization from vCenter without touching each guest. The data reveals your internal topology, which is sensitive reconnaissance material and a reason to control who can read it.

- **Applications and grouping.** Discovered servers are grouped into applications, which is how dependencies are visualized and how migration is planned as coherent units rather than isolated servers. Dependency mapping surfaces which servers communicate, which is what prevents migrating a component before what it depends on.

- **Migration tracking.** Migration Hub receives status updates from the migration tools. Application Migration Service for lift-and-shift server replication, Database Migration Service for databases, and integrated partner tools all report progress, so the dashboard shows each server, application, and database's state regardless of which tool is doing the work.

- **Migration Hub Orchestrator.** Workflow templates that sequence and automate migration steps for an application, coordinating the tasks across tools rather than running them manually. This is where ordering is enforced in an automated migration, so the database moves before the backend by design.

- **Strategy Recommendations.** Analyzes the discovered estate and recommends a migration strategy per application, rehost, replatform, refactor, which informs whether a workload is lifted as-is or modernized, a decision with security implications since a lift-and-shift carries existing misconfigurations forward.

- **Encryption.** Discovery and tracking data is encrypted at rest with KMS and in transit with TLS. The discovery agents transmit collected data to the service over TLS.

- **IAM surface.** `mgh:*` actions govern the Hub, and `discovery:*` governs Application Discovery Service. The meaningful split is between principals who may view migration status and discovery data and those who may configure discovery, group applications, or drive orchestration. Discovery data is sensitive, so read access itself is worth scoping.

- **Logging.** CloudTrail records Migration Hub and Discovery Service API activity, including who configured discovery, who grouped applications, and who initiated orchestration steps. This is the audit trail proving what was migrated, when, and by whom, which is the compliance artifact a regulated migration requires.

## Migration Hub versus the tools it coordinates

| Component | Role in a migration | Moves data | Produces the audit record of what moved | Security-relevant output |
|---|---|---|---|---|
| AWS Migration Hub | Central tracking and coordination | No | Yes, the aggregated timeline and status | Dependency map, migration record, discovery data |
| Application Discovery Service | Inventory and dependency collection | No | Feeds the record | Detailed internal topology, a sensitive dataset |
| Application Migration Service (MGN) | Rehost by block-level replication | Yes | Reports status to the Hub | Replicated servers, carrying existing config |
| Database Migration Service (DMS) | Database migration and replication | Yes | Reports status to the Hub | Data in transit, requires its own encryption and IAM |
| Migration Hub Orchestrator | Workflow automation across tools | No | Sequences and records steps | Enforced ordering of migration steps |
| DataSync | Bulk file transfer | Yes | Its own CloudWatch logs | Verified, encrypted file movement |

## What gets tested

- **Migration Hub is the tracking and coordination answer, not a mover.** A question asking which service moves the data points at MGN, DMS, or DataSync. A question asking which service tracks all migrations across teams and tools in one place points at Migration Hub.

- **Dependency mapping prevents insecure migration ordering.** When a scenario describes an application failing because a component moved before its dependency, the answer is Migration Hub's dependency visualization and Orchestrator's sequencing.

- **CloudTrail plus Migration Hub is the compliance audit answer** for proving what data moved, when, and who initiated it, which is the record regulators require when PII or PHI changes location.

- **Discovery data is sensitive reconnaissance.** It describes your internal topology in detail, so read access to Migration Hub and Discovery Service is worth restricting, and the home Region is where that data concentrates.

- **Agent versus agentless discovery** is a coverage and intrusiveness tradeoff. Agent-based gives per-server process and connection detail; agentless gives VM inventory without touching guests. A requirement to avoid installing software on each server points at the collector.

- **Strategy Recommendations informs rehost versus refactor**, and the security consideration is that a rehost carries existing misconfigurations into AWS unchanged, so a security-sensitive migration may favor replatform or refactor.

- **The Hub itself is free**, and cost comes from the underlying tools, which occasionally appears as a cost-optimization framing.

- **Home Region selection** is a data residency consideration for the discovery data, tested alongside other single-home-Region services.

## Limitations

- It moves nothing and enforces nothing on the data. Migration Hub is a coordination and visibility layer, so every actual security control during a migration lives in the tool doing the work: DMS encryption, MGN replication security, DataSync integrity, and the destination's own controls.

- The record is only as complete as the tools reporting into it. A migration performed with an unintegrated tool or by hand does not appear, leaving a gap in the very audit trail the Hub exists to provide.

- Discovery data is a detailed map of your internal environment concentrated in one Region, which is a valuable target and a data residency consideration, and its collection requires either agents on servers or a collector with access to vCenter.

- Dependency mapping is inferred from observed network connections over the discovery window, so a dependency that did not occur during observation is missed, and the map can be incomplete for infrequent interactions.

- It coordinates but does not remediate. Identifying that a post-migration configuration drifted from the source is a detection, and fixing it is separate work in another tool.

- Lift-and-shift migrations tracked through the Hub carry existing security posture forward unchanged. The Hub shows that a server moved, not that it was hardened in the process.

- Orchestrator workflows automate sequencing but are themselves infrastructure that can be misconfigured, and a flawed workflow can automate an insecure ordering as readily as a correct one.

- The service is oriented around a migration event with a beginning and end, so its value diminishes once the migration completes, and ongoing configuration governance belongs in Config, Security Hub, and the steady-state controls rather than in Migration Hub.