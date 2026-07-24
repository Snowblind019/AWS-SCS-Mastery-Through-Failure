# Amazon Managed Grafana

Grafana is an open-source visualization and alerting layer that queries other systems on demand rather than storing telemetry itself. Amazon Managed Grafana is AWS's hosted version of it, with the operational burden removed and, more importantly for security work, the identity and network layers replaced by AWS primitives: authentication through IAM Identity Center or a SAML IdP, data source access through IAM roles rather than stored keys, and optional PrivateLink so the workspace is never reachable from the internet. Its security relevance is that it is the correlation surface across CloudWatch, Athena over CloudTrail and VPC Flow Logs, OpenSearch, Prometheus, and X-Ray, which means one workspace often has read access to nearly every telemetry source in an estate. That concentration is the risk as much as the value: the workspace IAM role is a cross-service read grant, and dashboards render data that individual viewers may not be entitled to see at the source. The thing to hold onto is that Grafana authorizes at the dashboard and folder level while the data source role authorizes at the AWS level, so anyone who can edit a panel can query anything the workspace role can reach.

## How it works

- **Workspaces.** A workspace is the isolated Grafana instance. It is created with an authentication method, a set of permitted data sources and notification channels, an IAM role or service-managed permissions, and optionally a network access control list and VPC configuration. Everything else is configured inside the Grafana application.

- **Authentication.** Two options: IAM Identity Center, or SAML 2.0 against an external IdP such as Okta or Entra ID. There are no local Grafana user accounts to manage or leak. Users and groups are mapped to Grafana roles at assignment time.

- **Grafana roles.** Admin, Editor, and Viewer, assigned per user or group. Admin manages data sources, plugins, and permissions. Editor creates and modifies dashboards and, critically, can author arbitrary queries against any configured data source. Viewer can only see what has been built and shared with them.

- **Fine-grained permissions.** Folder and dashboard level permissions restrict which teams see which dashboards. Data source permissions can restrict which roles may query a given source. This is the mechanism for separating, say, a platform team's metrics dashboards from a security team's CloudTrail dashboards inside one workspace.

- **Data source authentication.** Managed Grafana assumes an IAM role in the workspace account and, through cross-account role chaining, roles in other accounts. Requests are SigV4-signed. There are no long-lived access keys stored in the workspace for AWS sources, which is the primary hardening advantage over self-hosted Grafana.

- **Service-managed versus customer-managed permissions.** Service-managed permissions have AWS provision and maintain the workspace role's policies, optionally scoped to an Organizational Unit for multi-account access. Customer-managed permissions give you a role you author yourself, which is what you use when the read scope needs to be narrower than the AWS-provided managed policies.

- **Network isolation.** Workspaces support a network access control list restricting inbound access by VPC endpoint or prefix list, so a workspace can be made reachable only from inside your network. Outbound connectivity into a VPC lets Grafana query private data sources such as a self-managed Prometheus or an OpenSearch domain with no public endpoint.

- **Grafana version and plugin management.** AWS pins supported Grafana major versions and controls upgrades, which removes the self-hosted patching burden for the many Grafana CVEs that have historically involved authentication bypass and path traversal. Plugin installation is restricted to an allowed set rather than arbitrary community plugins.

- **Alerting.** Grafana-managed alert rules evaluate on a schedule and route through contact points such as SNS, Slack, PagerDuty, or a generic webhook. Contact point credentials are stored in the workspace configuration, which makes them a secret to rotate and a reason to prefer SNS with IAM over a static webhook token.

- **API keys and service accounts.** Workspaces issue tokens for programmatic access, used for dashboard-as-code pipelines. These are bearer credentials with a role attached and an expiry, and they bypass the SSO path entirely, which makes their lifecycle a real control.

- **Logging.** CloudTrail records workspace-level API calls such as creation, permission updates, and data source configuration. Grafana application logs including login events and alerting activity can be exported to CloudWatch Logs, which is what gives you an audit trail of who logged in and what changed. Dashboard version history records edits inside the application.

## Managed Grafana versus adjacent visualization and analysis surfaces

| Option | Data model | Identity | Data access authorization | Native audit of user activity | Cross-account and cross-source correlation |
|---|---|---|---|---|---|
| Amazon Managed Grafana | Queries sources live, stores nothing | IAM Identity Center or SAML | Grafana roles plus one workspace IAM role per data source | Grafana logs to CloudWatch, plus CloudTrail on the workspace | Strong, many source types in one panel set |
| Self-hosted Grafana | Same | Local users, LDAP, OAuth, or SAML you configure | Whatever credentials you store on the instance | Whatever you configure | Same, but you own patching and secrets |
| CloudWatch dashboards | CloudWatch metrics and Logs Insights | IAM only | IAM policy of the viewing principal | CloudTrail | CloudWatch cross-account observability, AWS sources only |
| Amazon QuickSight | Cached SPICE or direct query | QuickSight users, IAM, or IdP | Row and column level security inside QuickSight | CloudTrail plus QuickSight events | Good for business data, weaker for live telemetry |
| OpenSearch Dashboards | Indices in the domain | Fine-grained access control, IAM, or SAML | Index, document, and field level | Domain audit logs | Only what is indexed in the domain |
| Security Hub and Detective | Findings and behavior graphs | IAM | IAM | CloudTrail | Purpose-built for security findings, not general telemetry |

## What gets tested

- **Managed Grafana over self-hosted when the requirement mentions no long-lived credentials, SSO, or reduced patching burden.** The workspace assumes IAM roles rather than storing access keys, which is the standard answer to "eliminate static credentials in the dashboard layer."

- **Editor role is a data access grant, not just an authoring grant.** Anyone who can create a panel can query the full scope of the workspace data source role. Restricting sensitive telemetry to a subset of users means either data source permissions, a separate workspace, or a narrower customer-managed role, not Viewer-only assignment on a dashboard.

- **Customer-managed permissions when the workspace read scope must be narrower than the AWS managed policies.** Service-managed permissions are convenient and broad, which is exactly the distractor.

- **Network access control lists plus PrivateLink** are the answer for making a workspace unreachable from the internet. There is no security group on the workspace itself.

- **Cross-account observability** uses role chaining from the workspace role into a role in each target account, or service-managed permissions scoped to an OU when the accounts are in the same Organization.

- **Grafana is not a SIEM and not a log store.** If a question asks for retention, correlation rules, or a system of record for findings, the answer is Security Hub, OpenSearch, or a SIEM. Grafana is the presentation layer over those.

- **API keys and service accounts bypass SSO.** Any question about an unattributable dashboard change or a leaked automation token points at token expiry, rotation, and restricting who can mint them.

- **Contact point secrets** such as Slack webhooks are stored credentials. Prefer SNS with IAM authorization over a static webhook when the question emphasizes credential management.

- **Enabling Grafana log export to CloudWatch** is the answer for auditing user logins and dashboard changes. CloudTrail alone covers the workspace API, not in-application activity.

## Limitations

- Grafana stores no data, so retention, integrity, and immutability are properties of the underlying source. A dashboard cannot answer a question the source no longer retains.

- Authorization inside Grafana is not the same as authorization at the source. The workspace role queries with its own permissions regardless of who is viewing, so a viewer can see data they could not query directly in the console. Achieving true per-user data scoping requires separate workspaces or separate data sources.

- No row-level or field-level security equivalent. Grafana can hide a dashboard but cannot filter query results by viewer identity for AWS data sources.

- Alerting is threshold and query based. There is no correlation engine, no entity resolution, and no case management, so it detects the shape of a problem rather than reasoning about an incident.

- Per-user licensing means the cost model discourages broad read access, which sometimes pushes teams toward shared accounts, which defeats attribution. Watch for that as an anti-pattern rather than a solution.

- Query cost is charged by the data source. A dashboard on a short refresh interval fanning out to CloudWatch Logs Insights or Athena is a recurring bill and can throttle the source during an incident, exactly when you need it.

- Supported Grafana versions and plugins lag the open-source project, so a required community plugin may simply not be available.

- Data source connectivity into private networks requires the workspace VPC configuration, and each additional private source expands the network path that has to be maintained and reviewed.