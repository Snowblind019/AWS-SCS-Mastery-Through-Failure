# Amazon Aurora DSQL

Aurora DSQL is a serverless, distributed, PostgreSQL-compatible relational database with active-active multi-Region writes and strong consistency. The DSQL in the name is distributed SQL, not dynamic SQL. For security work the important thing is how much of the traditional database security surface it removes: there are no instances to patch, no parameter groups, no engine version to upgrade, no master password, and no password authentication at all. Every connection authenticates with a short-lived token derived from IAM credentials, so database access becomes an IAM authorization decision with CloudTrail attribution, and the entire category of leaked, shared, and never-rotated database credentials disappears. What remains is the part AWS cannot take over: authorization inside the database, the network path to the endpoint, and the application code building queries. The thing to hold onto: Aurora DSQL moves database authentication fully into IAM, which means the security questions shift from credential management to IAM policy design and in-database privilege grants.

## How it works

**Authentication is IAM-only, with expiring tokens.** Clients generate an authentication token signed with their AWS credentials and present it as the password. Tokens are short-lived by default, so there is nothing durable to steal from a config file. The relevant IAM actions are `dsql:DbConnect` for regular database roles and `dsql:DbConnectAdmin` for the `admin` role, and they are scoped per cluster ARN.

**Database roles are linked to IAM roles explicitly.** Beyond `admin`, a database role is created in PostgreSQL and then associated with an IAM role using the `AWS IAM GRANT` statement. This is the mapping that lets an application role in IAM correspond to a database role holding only the table privileges it needs, which is the least-privilege pattern the service is designed around.

**Authorization inside the database is still PostgreSQL.** `GRANT` and `REVOKE` on schemas and tables govern what an authenticated principal can read and write. IAM decides who may connect and as which database role; SQL privileges decide what happens after that. Both layers must be designed, and neither substitutes for the other.

**Encryption is on by default.** Data at rest is encrypted with KMS using either an AWS owned key or a customer managed key chosen at cluster creation, and connections require TLS. A customer managed key is what enables key policy control, grant-based revocation, and CloudTrail visibility of key usage.

**Network access is through a Regional endpoint, optionally private.** There is no cluster sitting in your VPC. Private connectivity is achieved with an interface VPC endpoint over PrivateLink, and the endpoint policy plus security groups become the network control. Without that, access is over a public endpoint controlled purely by IAM.

**Multi-Region clusters are active-active with strong consistency.** Two peered Regions accept writes, coordinated by a witness Region, with no failover step and no replica lag to reason about. Availability improves, and the security consequence is that data residency is now a cluster topology decision: a peered Region is a Region where the data lives.

**Serverless removes the patching surface.** Capacity scales automatically, storage grows automatically, and there is no maintenance window, no minor version decision, and no OS to harden. Vulnerability management for the database layer stops being a customer responsibility.

**Concurrency uses optimistic control.** Conflicting concurrent transactions fail at commit and must be retried by the application rather than blocking. This is an availability and correctness property rather than a security one, but it changes how applications must be written.

**Logging is control plane plus in-database.** CloudTrail records cluster and connection-related API activity including token generation authorization. Database-level auditing is thinner than on RDS or Aurora provisioned, so detection of anomalous query behavior generally lives in the application tier or in a proxy, not in the engine.

## Comparison

| Property | Aurora DSQL | Aurora provisioned or Serverless v2 | RDS PostgreSQL |
| --- | --- | --- | --- |
| Authentication | IAM tokens only, no passwords | Password, or optional IAM database authentication | Password, or optional IAM database authentication |
| Patching responsibility | AWS, no versions exposed | Customer selects and schedules | Customer selects and schedules |
| Multi-Region writes | Active-active, strongly consistent | Global Database, single writer with replicas | Read replicas only |
| Network placement | Regional endpoint, PrivateLink for private access | In your VPC, subnet group and security groups | In your VPC, subnet group and security groups |
| Encryption at rest | On by default, AWS owned or customer managed KMS key | Optional at creation, cannot be added later | Optional at creation, cannot be added later |
| PostgreSQL feature coverage | Subset, several standard features unsupported | Near complete | Complete for the version |
| Backups | Continuous, managed, point in time | Automated backups and snapshots | Automated backups and snapshots |

## What gets tested

- **DSQL means distributed SQL.** Dynamic SQL is an application coding technique, unrelated to the service. Scenario wording about distributed writes, serverless scaling, or multi-Region consistency is about the service.
- **No password authentication.** Any answer proposing to store the database password in Secrets Manager and rotate it is wrong for Aurora DSQL, because there is no password. Secrets Manager remains correct for RDS and Aurora provisioned, which is exactly why the distractor appears.
- **The two connect actions.** `dsql:DbConnectAdmin` for the `admin` role and `dsql:DbConnect` for other roles, scoped to the cluster ARN. Granting admin connect to an application is the over-privilege error the question is testing.
- **Two authorization layers.** IAM controls connection and role selection; SQL `GRANT` controls table access. A complete least-privilege answer includes both, and answers relying on IAM alone are incomplete.
- **Customer managed KMS keys.** Choose one when the requirement is control over key policy, revocation of access to encrypted data, or auditability of key usage. The key is selected at cluster creation.
- **Private connectivity.** PrivateLink interface endpoints, not subnet groups, since there is no instance in a VPC. Endpoint policies are part of the answer for restricting access paths.
- **Data residency.** A peered Region stores data. When a scenario constrains where data may reside, multi-Region peering is a compliance decision, not just an availability one.
- **Patch management.** Vulnerability and patching questions resolve to AWS responsibility for the engine, which changes the shared responsibility answer relative to RDS.
- **Injection is still the application's problem.** Parameterized queries and input validation, plus least-privilege database roles to contain the damage. No managed database feature prevents SQL injection.

## Limitations

- PostgreSQL compatibility is partial. Several standard features are unsupported, including foreign key constraints, triggers, sequences, and most extensions, so it is not a drop-in destination for arbitrary existing schemas.
- In-engine audit logging is limited compared with RDS and Aurora provisioned, which makes query-level detection and forensic reconstruction harder.
- No password authentication also means no path for tools and third-party systems that cannot obtain IAM credentials and generate tokens.
- Token expiry requires client-side regeneration logic. Long-lived connection pools and legacy drivers frequently handle this badly.
- The default endpoint is public, so private access is an explicit design step rather than a property of where the cluster sits.
- Optimistic concurrency pushes retry handling into the application, and an application that does not retry correctly fails under contention.
- Customer managed key selection happens at creation. Encryption decisions are not something to revisit later.
- Regional and cluster-scoped. Organization-wide governance still comes from SCPs, RCPs, and Config rather than from anything the service provides itself.