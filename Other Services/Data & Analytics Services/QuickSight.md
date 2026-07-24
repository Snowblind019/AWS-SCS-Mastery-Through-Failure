# Amazon QuickSight

Amazon QuickSight is AWS's managed business intelligence service: datasets defined over Athena, Redshift, RDS, S3, or an uploaded file, visualizations built on top of them, and dashboards shared with people who never touch the AWS console. In a security context it is the reporting tier over an existing detection pipeline, turning CloudTrail in Athena, GuardDuty findings in S3, and VPC Flow Log summaries into trend views for leadership and non-engineering stakeholders. Its own security model deserves attention because QuickSight is one of the few AWS services with a real user directory of its own: it maintains users and groups, it can authenticate them through IAM, IAM Identity Center, SAML, or its own accounts, and it can share a dashboard with someone who has no IAM identity at all. It also caches data. SPICE holds a materialized copy of query results inside QuickSight, which means data governed by Lake Formation at the source may sit unfiltered in a cache with a completely separate permission model. The thing to hold onto is that QuickSight has its own identity store and its own copy of the data, so source-level access controls do not automatically follow the data into a dashboard.

## How it works

- **Account and namespaces.** QuickSight is enabled per AWS account in a chosen home Region. Namespaces provide hard tenant isolation, with users, groups, and assets in one namespace unable to see or be shared with another. This is the mechanism for multi-tenant deployments such as an MSSP serving separate customers.

- **Identity model.** Users are QuickSight principals, provisioned through IAM federation, IAM Identity Center, SAML 2.0 against an external IdP, an Active Directory connection on Enterprise edition, or QuickSight-native accounts on Standard edition. Roles are Admin, Author, and Reader, with Reader being view-only and Author able to create datasets and therefore able to query the data source.

- **Data source connections.** A data source stores connection details and credentials. For AWS sources it uses either the QuickSight service role or an explicitly specified IAM role, which is the more controllable option. For private VPC sources it uses a VPC connection with an ENI in your subnets and a security group, which is required to reach an RDS instance or Redshift cluster with no public endpoint.

- **SPICE.** An in-memory columnar cache holding a materialized copy of the dataset, refreshed on a schedule or incrementally. Queries hit the cache rather than the source. SPICE is encrypted at rest, on Enterprise edition with a customer managed KMS key. The consequence for security is that the cache is a second copy of the data with its own lifetime and its own access path.

- **Direct query.** The alternative, sending every visual's query to the source at view time. Slower and load-generating, but the data is never copied and source-level controls stay in the path.

- **Row-level and column-level security.** Enterprise edition. RLS attaches a rules dataset mapping user or group names to permitted field values, filtering rows per viewer. Column-level security restricts which fields a principal can see at all. Both are enforced by QuickSight, not by the source, and both apply to SPICE and direct query datasets.

- **Asset-level permissions.** Datasets, analyses, dashboards, and folders each carry a permissions list naming users and groups. Sharing a dashboard does not share the underlying dataset, so a Reader can see the visuals without being able to build new queries against the data.

- **Embedding.** Dashboards can be embedded in an application. Registered-user embedding issues a short-lived embed URL for a known QuickSight principal. Anonymous embedding issues a URL for a session with no identity, scoped by session tags that drive RLS. Anonymous embedding is how you serve a dashboard to end users with no AWS or QuickSight account, and the session tag becomes the entire authorization boundary.

- **Encryption.** In transit with TLS everywhere. At rest, SPICE and QuickSight-managed metadata are encrypted, with customer managed KMS keys supported on Enterprise edition. Source data remains under its own encryption.

- **Network.** VPC connections for private data sources. IP restriction rules can limit which source addresses may access the QuickSight account, and Enterprise edition supports access through an interface VPC endpoint so users reach QuickSight without traversing the internet.

- **Logging.** CloudTrail records QuickSight API activity including user provisioning, dataset and dashboard creation, permission changes, and, notably, dashboard views and data exports, which is what makes viewing attribution possible. CloudTrail is the only audit surface; there is no separate QuickSight audit log.

## QuickSight versus adjacent presentation and query surfaces

| Option | Data handling | Identity | Per-viewer data filtering | Reaches non-technical viewers | Audit of views and exports |
|---|---|---|---|---|---|
| Amazon QuickSight | SPICE cache or direct query | Own user store, federated via IAM, Identity Center, or SAML | Row and column level security, session tags for anonymous embedding | Yes, that is the point | CloudTrail includes views and exports |
| Amazon Managed Grafana | Queries live, stores nothing | IAM Identity Center or SAML | None for AWS sources, dashboard permissions only | Possible but operationally oriented | Grafana logs to CloudWatch plus CloudTrail |
| Athena directly | Queries S3 in place | IAM | Lake Formation row, column, and cell filters | No, requires console or SQL client | CloudTrail plus Lake Formation events |
| CloudWatch dashboards | CloudWatch data | IAM | None, IAM of the viewer applies | Requires console access | CloudTrail |
| Security Hub | Findings store | IAM | None beyond account and Region | Console only | CloudTrail |
| OpenSearch Dashboards | Indexed copy in the domain | Fine-grained access control, IAM, SAML | Document and field level | Yes, with dashboards | Domain audit logs |

## What gets tested

- **SPICE is a copy of the data.** Any question about Lake Formation or source permissions not applying in a dashboard is answered by that fact. To keep source enforcement in the path, use direct query. To enforce per-viewer scoping regardless, use QuickSight row-level security.

- **Row-level security is the multi-tenant answer within one dashboard.** Namespaces are the answer when tenants must not even share an asset space, which is the stronger isolation and the correct choice for an MSSP or a customer-facing deployment.

- **Anonymous embedding relies entirely on session tags.** The application generating the embed URL is the authorization decision point, so the answer to preventing cross-tenant data exposure is correct session tag assignment plus RLS keyed to those tags, not a network control.

- **Author role is a data access grant.** An Author can create a new dataset against any data source they can see and query it freely. Restricting sensitive data means restricting the data source or the dataset, not assigning fewer dashboards.

- **Private data sources need a VPC connection**, with an ENI and a security group. A public endpoint on RDS is always the wrong remediation.

- **Customer managed KMS key for SPICE is Enterprise edition.** Any requirement for control over the encryption key of cached analytics data rules out Standard edition.

- **CloudTrail captures dashboard views and exports**, which is what satisfies "we must know who viewed the compliance report." That is a genuine differentiator against Grafana, where per-view attribution is weaker.

- **IP restriction rules and the VPC endpoint** are the answers for limiting where QuickSight can be reached from, and they operate at the account level rather than per dashboard.

- **QuickSight is not a detection service.** If a question asks for finding generation, correlation, or alerting on security events, the answer is GuardDuty, Security Hub, or EventBridge. QuickSight reports on what those produce.

## Limitations

- SPICE creates a second copy of the data outside the governed source, with its own refresh schedule, its own key, and its own permissions. Deleting a record at the source does not remove it from SPICE until the next refresh, which is a real gap for data deletion requests.

- Row-level and column-level security, customer managed keys, namespaces, VPC endpoints, and Active Directory integration are Enterprise edition only. Standard edition is inadequate for most regulated deployments.

- QuickSight enforces RLS itself. A misconfigured rules dataset fails open in the sense that missing rules for a user typically means no rows rather than all rows, but an over-broad rule silently grants access with no error.

- SPICE has per-dataset size and row limits and refresh frequency limits, so very large or near-real-time security datasets push you to direct query, which then loads the source during every dashboard view.

- Direct query cost and load fall on Athena or the database. A widely shared dashboard on a short refresh can become an expensive and throttling workload on the same Athena workgroup used for incident response.

- Reader pricing is per session, so broad distribution has a variable cost that occasionally pushes teams toward shared accounts, which destroys the view attribution CloudTrail would otherwise give.

- Cross-Region behavior is awkward: the QuickSight account has a home Region, and querying data in other Regions works but adds transfer cost and latency, while some features remain Region-bound.

- The QuickSight user directory is separate from IAM. Deprovisioning a user in the corporate IdP does not necessarily remove the QuickSight user or their asset ownership, so offboarding requires an explicit step.