# Amazon Q Developer in Chat Applications

Amazon Q Developer in chat applications brings the AWS-aware assistant into Slack and Microsoft Teams through AWS Chatbot, so engineers can query account state, diagnose errors, and run a scoped set of AWS operations from the channel they already work in. It is not a standalone product so much as the chat surface of Q Developer plus AWS Chatbot's channel integration, which is why its security model is really Chatbot's security model with a conversational front end. That distinction is the whole point for a security engineer: the assistant operating in a channel acts under an IAM role that AWS Chatbot assumes, not under the individual asking the question, and that role, combined with the channel guardrail policy, is the entire boundary on what anyone in the channel can do to your AWS accounts. A channel is a shared space, often with broad membership, so a permissive channel role turns every member into a holder of those permissions and every piece of returned output into something visible to everyone present. The thing to hold onto is that authorization in chat is the channel role and its guardrail policy, not the identity of the person typing, so the channel itself becomes the security principal.

## How it works

- **AWS Chatbot as the substrate.** The integration is configured through AWS Chatbot, which connects a Slack workspace or Teams tenant to AWS accounts. Q Developer's conversational and diagnostic capability rides on top of that connection, and Chatbot is where the roles, channels, and guardrails are defined.

- **Channel configurations.** Each configured channel is associated with one or more IAM roles and a set of guardrail policies. A channel role defines what actions are possible from that channel, and the guardrail policies are the maximum permission ceiling that no channel role can exceed, functioning like a permissions boundary for chat.

- **Channel role versus user identity.** Commands and actions invoked in the channel execute under the channel's IAM role, assumed by the AWS Chatbot service, not under the AWS identity of the person who typed. This is the single most important fact about the model. Attribution of who typed a command comes from the chat platform's logs and from CloudTrail's record of the Chatbot-assumed session, not from an individual IAM principal.

- **User-level roles.** Chatbot can additionally map chat users to individual IAM roles so that a command runs under that user's role rather than a shared channel role, which restores per-user attribution and per-user scoping at the cost of configuration effort. This is the hardened pattern when a channel must support privileged actions.

- **Read paths and notifications.** The channel receives CloudWatch alarms, Security Hub findings, and other events routed through SNS, and the assistant answers questions about resources and logs by querying AWS on the channel role's behalf. Diagnostic answers can include resource identifiers, IP addresses, and configuration details, all of which become visible to every channel member.

- **Command execution.** Running AWS CLI commands from chat is permitted only within the intersection of the channel role and the guardrail policy. Guardrails can restrict to read-only operations, which is the safe default, or permit specific mutating actions where a channel is trusted and its membership controlled.

- **Approval workflows.** Sensitive actions can be gated so they require confirmation, and custom actions can be defined with buttons that invoke specific, pre-approved operations rather than free-form CLI, which narrows what the channel can do to a reviewed set.

- **Data handling.** Content is not used to train foundation models, and processing stays within the AWS boundary. Answers are generated from your account state and AWS knowledge rather than being sent to a third-party model provider.

- **Logging.** CloudTrail records the actions taken by the Chatbot-assumed role, including the CLI commands run from chat, which is the AWS-side audit trail. The chat platform's own audit log records who typed what in the channel. Correlating an action back to a person requires both, since CloudTrail alone shows the role, not the individual.

- **Network and identity.** The chat platform connection is authorized once at configuration time. There is no per-message AWS authentication from the individual user unless user-level roles are configured, so the trust is established at the channel and workspace level.

## Q in chat versus other operational access paths

| Path | Who the action runs as | Attribution to an individual | Mutating actions possible | Where output is visible | Native command audit |
|---|---|---|---|---|---|
| Q Developer in chat via Chatbot | Channel role, or mapped user role | Only with user-level roles or chat logs | Yes, within guardrail policy | The whole channel | CloudTrail on the assumed role |
| AWS CloudShell | The console user's session | Yes, per user | Yes, full identity permissions | The individual session | CloudTrail on the API calls |
| AWS Console | The signed-in principal | Yes | Yes | The individual session | CloudTrail |
| Session Manager | Instance role plus caller identity | Yes | On the instance | The individual session | Full session logging |
| Direct CLI with SSO | Short-lived user credentials | Yes | Yes | The individual terminal | CloudTrail |
| SNS to email or ticketing | Not applicable, read-only delivery | Not applicable | No | Recipients | Not applicable |

## What gets tested

- **Actions run as the channel role, not the user.** The core security fact. Any question about what someone can do from Slack is answered by the channel role plus the guardrail policy, and any question about limiting it is answered by scoping those, not by the individual's IAM permissions.

- **Guardrail policies are the ceiling.** They cap what any channel role can do, functioning as a permissions boundary for chat. Read-only guardrails are the safe default, and permitting mutation is a deliberate escalation tied to controlling channel membership.

- **User-level roles restore attribution.** When a requirement is that privileged actions from chat must be traceable to an individual, mapping chat users to individual IAM roles is the answer, not relying on the shared channel role.

- **Channel membership is an access control.** Because output and command capability are shared with everyone in the channel, restricting who is in the channel is part of the security posture, and a public or broadly shared channel with a mutating role is the exposure.

- **CloudTrail records the assumed role, not the person.** Full attribution requires correlating CloudTrail with the chat platform's audit log. A question expecting CloudTrail alone to identify who ran a command is testing whether you know the assumed-role gap.

- **Read-only diagnostics still disclose data.** Returned answers containing resource identifiers, IPs, and configuration are visible channel-wide, so even a read-only channel is a data exposure surface if its membership is not controlled.

- **Custom actions and approval gates** narrow free-form command capability to a reviewed set, which is the answer when a channel needs some operational power without granting broad CLI access.

- **Content is not used for training and stays in the AWS boundary**, which is the data governance answer, consistent with the rest of the Q family.

- **This is Q Developer's chat surface, distinct from Q Business.** Q in chat apps diagnoses AWS resources and runs operations. Q Business answers questions over enterprise documents. The two are offered as distractors for each other.

## Limitations

- The channel is effectively the security principal. Without user-level role mapping, everyone in the channel shares the channel role's permissions and every returned answer, which is a coarse model for anything privileged.

- Attribution is split across two systems. CloudTrail shows the Chatbot-assumed role and the chat platform shows the typist, and neither alone identifies who did what, so a complete audit requires correlating both.

- Output is broadcast to the channel. There is no per-user response visibility, so a diagnostic answer containing sensitive configuration is seen by every member present.

- Mutating actions from a shared channel are inherently risky. Even with guardrails, permitting changes from a space with fluid membership widens the set of people who can alter production.

- The trust is established at configuration time at the workspace and channel level, so the security of the AWS connection depends partly on the security of the Slack or Teams tenant, which is outside AWS controls.

- Availability, feature depth, and supported operations differ between Slack and Teams and evolve, so a capability present in one may be absent in the other.

- The convenience encourages operational work in a lower-audit, higher-visibility surface than the console, which can normalize running commands in a shared space rather than in a properly attributed session.

- Guardrail and role misconfiguration fails toward more access, since a channel role broader than intended silently grants that breadth to the whole channel with no per-action review unless approval workflows were configured.