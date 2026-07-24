# Amazon Managed Service for Prometheus

Prometheus is an open-source time-series metrics system built around a pull model: a server scrapes HTTP endpoints exposing metrics in a text format, stores the samples in a local time-series database, and evaluates PromQL queries and alerting rules against them. Amazon Managed Service for Prometheus is the hosted version, replacing the self-managed server and its storage with a workspace that speaks the same remote-write and query APIs, and replacing Prometheus's near-total absence of built-in security with AWS primitives: SigV4-signed requests authorized by IAM, encryption at rest with a customer managed KMS key, and reachability restricted to a VPC through an interface endpoint. That substitution is the entire security story. Open-source Prometheus has no authentication, no authorization, no transport encryption, and no multi-tenancy by default, so a self-hosted deployment is only as protected as the network and reverse proxy in front of it. Managed Prometheus also matters for detection engineering because runtime signals such as restart storms, CPU saturation, and abnormal egress volume show up in metrics well before they show up in findings. The thing to hold onto is that Prometheus security is entirely perimeter and identity work, so the managed service's value is that IAM replaces the reverse proxy you would otherwise have to build and defend.

## How it works

- **Workspaces.** A workspace is the isolated tenant boundary: its own metric store, its own alert manager definition, its own IAM resource ARN, and optionally its own KMS key. Separating teams or environments means separate workspaces, since there is no in-workspace tenancy model.

- **Ingestion through remote write.** Managed Prometheus does not scrape. A collector does the scraping and forwards samples over the Prometheus remote-write API. The collector is either an AWS managed scraper for EKS, an OpenTelemetry Collector or ADOT distribution, or a self-managed Prometheus server in agent mode. Every remote-write request is SigV4-signed with `aps:RemoteWrite`.

- **AWS managed collector for EKS.** A fully managed scraper that discovers pod and service targets in an EKS cluster and writes to a workspace, removing the need to run and secure a Prometheus server in the cluster. It uses an ENI in your VPC and a service-linked role, and requires the cluster's `aws-auth` or access entry configuration to grant it discovery permissions.

- **Query and IAM authorization.** PromQL queries go through `aps:QueryMetrics` on the workspace ARN. There is no user database and no API token. Grafana, a CLI, or an application queries with an IAM identity, which means query access is granted and revoked with policy rather than with a shared credential.

- **Encryption.** At rest with an AWS owned key by default or a customer managed KMS key specified at workspace creation and immutable thereafter. In transit with TLS on both ingestion and query paths, with no plaintext option.

- **Network isolation.** The workspace API is reachable through an interface VPC endpoint, and the endpoint policy plus an IAM condition on `aws:SourceVpce` is how you restrict ingestion and query to your network. Without that, the workspace is reachable from any network by any principal holding the right IAM permissions.

- **Rules and alerting.** Recording rules precompute expensive expressions, and alerting rules fire when a PromQL expression holds for a duration. Both are uploaded as rule group namespaces. The managed Alertmanager routes firing alerts to SNS, and from SNS to Lambda, Chatbot, email, or a paging system. Alertmanager configuration including inhibition, grouping, and silences is uploaded as a definition.

- **Logging.** CloudTrail records workspace creation, KMS key configuration, rule namespace changes, and Alertmanager definition changes. Ingestion and query API calls at volume are not individually logged, so attribution for a specific query is not available. Workspace logs including rule evaluation failures can go to CloudWatch Logs.

- **Retention and scale.** Metrics are retained 150 days with no configuration. Ingestion, active series, and query limits are per-workspace quotas that can be raised, and cost is driven by samples ingested, storage, and query samples processed.

- **Exporters and instrumentation.** The metrics themselves come from exporters: `node_exporter` for host metrics, `kube-state-metrics` for Kubernetes object state, cAdvisor for container resource usage, `blackbox_exporter` for synthetic probes, and application SDK instrumentation. Each exporter is an HTTP endpoint that, unprotected, discloses detailed internal topology and resource state, which is why exporter exposure is a real finding and not a hygiene note.

## Managed Prometheus versus adjacent metrics and telemetry services

| Option | Collection model | Authentication | Encryption at rest | Retention | Multi-tenancy | Query language |
|---|---|---|---|---|---|---|
| Amazon Managed Service for Prometheus | Remote write from a collector | IAM SigV4, no user accounts | KMS, customer managed key optional | 150 days, fixed | One workspace per tenant | PromQL |
| Self-hosted Prometheus | Direct scrape by the server | None natively, proxy required | Whatever the volume provides | Configurable, local disk bound | None, separate servers required | PromQL |
| CloudWatch metrics | Push from agents and services | IAM | AWS managed | 15 months, rolled up over time | Account and namespace | Metric math and Logs Insights |
| Amazon Managed Grafana | Queries other sources | IAM Identity Center or SAML | Not a store | Not a store | Folders, dashboards, data source permissions | Per data source |
| OpenSearch | Indexed ingestion | Fine-grained access control, IAM, SAML | KMS | Configurable, storage bound | Index and document level | Query DSL and PPL |
| ADOT and OpenTelemetry | Collector, agent or gateway | Depends on the backend | Depends on the backend | Depends on the backend | Depends on the backend | Depends on the backend |

## What gets tested

- **Managed Prometheus over self-hosted when the requirement mentions authentication, encryption, or eliminating an unauthenticated endpoint.** Open-source Prometheus has no auth, so the self-hosted answer always requires building a reverse proxy with OIDC or mTLS in front of it, which is the more complex distractor.

- **IAM is the only authorization mechanism.** `aps:RemoteWrite` for ingestion, `aps:QueryMetrics` for reads, scoped to the workspace ARN. There is no per-metric or per-label authorization, so isolating a tenant means a separate workspace.

- **Cross-account ingestion uses role assumption**, with the collector assuming a role in the workspace account. A shared credential answer is wrong.

- **The KMS key is set at workspace creation and cannot be changed.** Same pattern as the FinSpace environment key and RDS at-rest encryption. Rotating to a different CMK means a new workspace.

- **VPC endpoint plus endpoint policy** is the answer for keeping ingestion and query traffic off the public internet, and an `aws:SourceVpce` condition is what actually denies access from elsewhere.

- **Exposed exporter endpoints are a real exposure.** `/metrics` on `node_exporter` or `kube-state-metrics` reveals host inventory, container images, resource limits, and cluster topology. Remediation is network policy and security groups restricting the scrape path to the collector, not authentication on the exporter, which most exporters do not support.

- **Alert routing is through SNS.** Anything requiring an alert to trigger automated remediation goes SNS to Lambda or SNS to EventBridge, and the SNS topic policy plus KMS encryption on the topic is part of the answer.

- **Metrics for detection, logs for evidence.** Prometheus answers "is this workload behaving abnormally right now." It does not answer "who did this," which requires CloudTrail, and it does not retain the record long enough for most forensic requirements.

## Limitations

- Retention is fixed at 150 days with no configuration and no cold tier. Anything needing longer requires exporting to S3 through a separate pipeline.

- No query-level audit. You cannot determine which principal ran which PromQL query against which workspace, so a workspace containing sensitive operational data has weaker attribution than a logging service would.

- Authorization granularity stops at the workspace. There is no label-based or metric-based access control, so any principal with `aps:QueryMetrics` sees every series in that workspace.

- Metrics are aggregate and dimensional by design. High-cardinality labels such as user ID, request ID, or source IP are an anti-pattern that degrades performance and cost, which limits how far metrics can substitute for logs in an investigation.

- Managed Prometheus does not scrape. A collector still has to be deployed, secured, and monitored, and if the collector fails there is no data, with the gap visible only if you alert on the absence of samples.

- The KMS key is immutable per workspace, and workspace-level quotas on active series and ingestion rate become hard operational limits during an incident when metric volume spikes.

- Alertmanager configuration is uploaded as a definition rather than managed through a console workflow, and a malformed definition can silence alerting without an obvious failure signal.

- Exporters run with meaningful host or cluster visibility. `node_exporter` typically needs host filesystem and process access, and cAdvisor needs container runtime access, so the collection layer itself is privileged software that expands the attack surface it was deployed to observe.