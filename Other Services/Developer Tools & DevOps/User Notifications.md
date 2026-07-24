# AWS User Notifications

AWS User Notifications is a delivery and routing layer that turns EventBridge events into human-directed alerts: a console Notification Center, email, mobile push, and chat through AWS Chatbot into Slack, Microsoft Teams, or Amazon Chime. It sits above the event producers rather than beside them, so GuardDuty findings, Security Hub findings, CloudWatch alarms, Health events, Budgets thresholds, and Support case updates all route through one configuration instead of a separate SNS topic and subscription per source. Its security relevance is delivery assurance. Detection controls fail operationally far more often through undelivered or ignored alerts than through missed detection, and the common failure mode is a shared mailbox or a busy Slack channel where a high-severity finding scrolls past unread. User Notifications addresses that by making the destination a person or a specific channel with per-user console visibility, and by aggregating duplicate events so a finding storm produces one summary rather than four hundred messages. The thing to hold onto is that it is a routing and presentation layer over EventBridge, so it changes who sees an alert and how, not what is detected.

## How it works

- **Notification configurations.** The top-level resource. Each one has a name, an aggregation setting, and a set of event rules plus a set of delivery channels. A configuration is the unit you would create per audience, such as one for security on-call and one for cost owners.

- **Event rules.** Define the source, event type, and Regions to match, expressed as an EventBridge event pattern. Filtering can narrow to specific detail types, severities, or resource attributes, which is what keeps a security configuration from also delivering every routine deployment event.

- **Delivery channels.** Email addresses, the AWS Console Mobile Application for push, and AWS Chatbot channel configurations for Slack, Teams, and Chime. Email destinations require confirmation by the recipient before delivery begins.

- **Aggregation.** Configurable off, short, or long. When on, similar events within a window are collapsed into a single summary notification. This is the control that prevents a GuardDuty finding burst or a mass alarm state change from producing hundreds of individual messages, which in practice is what causes people to mute a channel.

- **Notification Center.** The console bell icon, showing notifications for the signed-in principal across accounts and Regions in the Organization where configurations apply. This is the per-user view that email and chat cannot provide.

- **Organizations integration.** With a delegated administrator, notification configurations can be managed centrally and applied across member accounts, which removes the per-account setup that made SNS-based alerting so inconsistent in large estates.

- **Notification Hubs.** You select Regions to act as hubs where notification data is stored and processed. This is a data residency decision, since notification content including finding details is replicated to and retained in those Regions.

- **Managed notifications.** AWS-published notification categories such as Health events and account-level operational messages, subscribed to as a set rather than defined by an event rule. Some are mandatory and cannot be unsubscribed.

- **IAM surface.** `notifications:*` actions govern creating and modifying configurations, event rules, and channel associations, and `notifications-contacts:*` governs email contact management. Restricting who can modify or delete a configuration matters, because deleting a rule silently stops delivery with no failure signal.

- **Logging.** CloudTrail records configuration and rule changes, channel associations, and hub registration. There is no delivery receipt log, so proving that a specific notification reached a specific person is not directly supported.

## User Notifications versus other alert delivery paths

| Mechanism | Intended consumer | Per-user targeting | Filtering granularity | Deduplication | Chat integration | Cost |
|---|---|---|---|---|---|---|
| AWS User Notifications | Humans | Yes, console per principal plus per-channel | EventBridge patterns on source, type, Region | Built-in aggregation | Native through Chatbot | Free, underlying services bill separately |
| Amazon SNS | Applications and humans | No, per subscription endpoint | Message attribute filter policies | None | Requires Chatbot or a Lambda | Per message and per delivery |
| Amazon EventBridge | Applications | No | Full event pattern matching, most expressive | None | Through a target | Per event on custom buses, rules are free |
| AWS Chatbot | Chat channels | Channel level, not user level | Whatever the upstream sends | None | Native | Free, upstream billed |
| Security Hub with EventBridge | Applications and ticketing | No | Finding fields via event pattern | None | Through a target | Per finding ingestion |
| Incident Manager | On-call responders | Yes, escalation plans and contacts | Triggered by alarms and events | Incident level | Through engagements | Per incident and per engagement |

## What gets tested

- **User Notifications is delivery, not detection.** If a question asks how findings are generated, correlated, or scored, the answer is GuardDuty, Inspector, or Security Hub. User Notifications is the answer when the requirement is that a specific person or team is reliably informed.

- **Aggregation is the answer to alert fatigue** from a finding storm, contrasted with SNS which delivers every message and has no deduplication.

- **Chatbot is how chat delivery happens**, and Chatbot itself requires an IAM role governing what it may do in response to a message. If a scenario involves running commands from Slack, the Chatbot channel role is the control being tested.

- **Delegated administrator plus Organizations integration** is the answer for consistent alerting across a multi-account estate, replacing per-account SNS topics and subscriptions.

- **Notification hub Region selection is a data residency decision.** Notification content is stored in the hub Regions, which matters when findings contain resource identifiers or data subject to residency constraints.

- **For programmatic response, use EventBridge directly.** User Notifications targets people. Automated remediation goes EventBridge to Lambda, Systems Manager Automation, or Step Functions, and any answer routing remediation through User Notifications is wrong.

- **For paging and escalation with acknowledgement**, the answer is Incident Manager or a third-party paging service, not User Notifications, which has no escalation policy, no acknowledgement, and no on-call schedule.

- **Deleting a notification configuration stops delivery silently.** That makes `notifications:DeleteNotificationConfiguration` a control worth restricting and alerting on, since disabling alerting is a common step in an intrusion.

## Limitations

- No delivery confirmation or read receipt. There is no artifact proving a notification was received or seen, so a compliance requirement to evidence alert delivery and acknowledgement needs a ticketing system or Incident Manager instead.

- No escalation, no acknowledgement, no on-call rotation. It notifies once and stops. Anything requiring "if unacknowledged in fifteen minutes, page the secondary" is out of scope.

- Filtering is limited to what the source event contains. It cannot enrich, correlate across events, or suppress based on external context such as a known maintenance window.

- Aggregation trades completeness for readability. A collapsed summary can hide the specific finding that mattered inside a burst of similar ones, and the aggregation window is not tuned per rule.

- Email delivery depends on recipient confirmation and is subject to the usual mail filtering and deliverability problems, which is precisely the failure mode alerting is meant to avoid.

- Chat delivery inherits the security posture of the chat workspace. A Slack channel with broad membership means finding details including resource names and IP addresses are visible to everyone in it.

- Notification content is stored in the selected hub Regions, which is an additional copy of security finding data outside the Region where the finding originated.

- Source coverage is limited to services that publish suitable events, and event schemas differ per source, so rule authoring is not uniform across services.

- It adds a layer between detection and response without adding reliability guarantees. If the requirement is that an alert must not be lost, a durable queue or a ticket creation step belongs in the path rather than a notification alone.