# Amazon Q Business

Amazon Q Business is a managed retrieval-augmented generation assistant that indexes an organization's internal content through prebuilt connectors and answers natural language questions against it with citations. Architecturally it sits above Bedrock, adding the parts that make an LLM usable on enterprise data: connectors, a managed index, an identity layer, and a permission model. The defining security property, and the reason it exists as a separate service rather than a Bedrock pattern, is that it propagates source-system permissions into retrieval. When a connector crawls SharePoint or Confluence, it ingests each document's access control list alongside the content, and at query time it filters candidate documents against the asking user's identity before anything reaches the model. That inversion matters because the default failure mode of every enterprise RAG deployment is an index that flattens permissions and cheerfully summarizes the compensation spreadsheet for whoever asks. The counterweight is that Q Business creates a single searchable index spanning systems that were previously separated by friction, so a permission mistake in one connector is now reachable through a natural language query rather than requiring someone to know where to look. The thing to hold onto is that Q Business enforces access at retrieval time using permissions copied from the source system, so its security depends entirely on those ACLs being correctly mapped and kept current.

## How it works

- **Applications and indexes.** An application is the top-level resource holding the index, the data sources, the retriever configuration, the identity provider association, and the guardrails. The index stores parsed, chunked, and embedded document content along with each document's access control metadata.

- **Identity model.** Users authenticate through IAM Identity Center, which federates to an external IdP such as Okta or Entra ID. Q Business resolves the signed-in user to an identity it can match against the ACLs captured during crawling. Getting this mapping right, particularly matching email addresses or group identifiers across systems, is the practical hard part of a deployment and the most common source of both over-exposure and false negatives.

- **Connectors and permission ingestion.** Prebuilt connectors for SharePoint, Confluence, Slack, Google Drive, Salesforce, Jira, GitHub, S3, and dozens more. Each connector authenticates with credentials stored in Secrets Manager, crawls content on a schedule, and captures document-level ACLs including user and group grants. Custom sources can be pushed through the API with ACLs supplied explicitly.

- **Retrieval-time filtering.** At query time the retriever filters candidate documents to those the asking user is authorized to see in the source system, then passes only those to the model. Documents the user cannot access are never in the prompt, which means the model cannot leak them regardless of how the question is phrased.

- **Attribute filtering and metadata boosting.** Document attributes captured at ingestion can constrain retrieval further or bias ranking, which is how you scope an assistant to a subset of content or prioritize authoritative sources over stale ones.

- **Guardrails and topic controls.** Global controls and per-topic rules define blocked words, blocked topics, and whether the model may answer from its own general knowledge when the index returns nothing. That last setting is significant: allowing fallback to model knowledge turns a grounded internal assistant into a general chatbot that will confidently answer questions about your policies from training data.

- **Plugins and actions.** Q Business can invoke actions in external systems such as creating a Jira ticket or a ServiceNow record, either through prebuilt plugins or a custom plugin defined by an OpenAPI schema. Each action executes with credentials configured for the plugin, which makes plugins a write path from a chat interface into production systems and a distinct authorization concern from retrieval.

- **Encryption.** Content at rest in the index is encrypted with an AWS owned key by default or a customer managed KMS key specified at application creation. TLS everywhere in transit. Connector credentials live in Secrets Manager under your key policy.

- **Network.** Connectors reaching private data sources use a VPC configuration with subnets and security groups. The Q Business API is reachable through interface VPC endpoints, and end user access to the web experience can be restricted by IdP policy and network conditions.

- **Data handling.** Customer content is not used to train the underlying foundation models, and inference runs within the AWS boundary. This is the contractual and architectural answer to the most common objection in regulated environments.

- **Logging.** CloudTrail records control plane operations including application, index, data source, plugin, and guardrail configuration, plus API activity. Conversation content is retrievable through the conversation history APIs, and chat interaction logging to CloudWatch Logs can be configured. Assuming that every prompt and response is captured in CloudTrail by default is a mistake worth correcting explicitly.

## Q Business versus adjacent AI and search options

| Option | Data grounding | Permission enforcement | Who operates the retrieval layer | Action execution | Typical consumer |
|---|---|---|---|---|---|
| Amazon Q Business | Managed index over connectors | Source-system ACLs enforced at retrieval | AWS | Plugins to external systems | End users across the business |
| Amazon Q Developer | Code, AWS docs, and your repositories | Repository and IAM permissions | AWS | Code changes and AWS operations | Developers and cloud engineers |
| Bedrock Knowledge Bases | Vector store you configure over S3 or a database | Metadata filtering you implement | You | Bedrock Agents | Applications you build |
| Amazon Kendra | Managed enterprise search index | Source ACLs, token-based user context | AWS | None | Search applications |
| Bedrock with a custom RAG pipeline | Whatever you build | Entirely your responsibility | You | Whatever you build | Applications you build |
| OpenSearch with vector search | Indices you manage | Fine-grained access control on documents and fields | You | None | Search and analytics applications |

## What gets tested

- **Retrieval-time ACL enforcement is the differentiator.** If a requirement says users must only receive answers grounded in content they are already permitted to see, the answer is Q Business or Kendra with user context, not a Bedrock Knowledge Base where filtering is your responsibility to implement.

- **Identity mapping is the weak point.** Expect scenarios where a user sees content they should not, and the root cause is a connector whose group or user identifiers did not map cleanly to the IdP identity, or an index whose ACLs are stale relative to a permission revocation in the source.

- **Source permission changes are not instant.** Revoking a user's access in SharePoint takes effect in Q Business only after the next crawl, so the answer to "how quickly does a revocation propagate" is the sync schedule, and shortening it is the mitigation.

- **Allowing the model to answer from general knowledge** is a configuration choice with a real risk profile. When a question describes the assistant fabricating policy answers, the answer includes disabling fallback so it responds only from indexed content or declines.

- **Plugins are a write path.** An action that creates a ticket or updates a record executes with the plugin's configured credentials, not the user's, so plugin authorization is separate from retrieval authorization and needs its own scoping and review.

- **Connector credentials live in Secrets Manager**, and the Q Business service role needs access to them and to the KMS key encrypting them. A connector failing to sync after encryption is enabled is a key policy problem.

- **Customer managed KMS key on the application** is the answer for control over index encryption and for the ability to render indexed content unreadable by disabling the key.

- **Customer content is not used for model training**, which is the answer to the data governance objection, alongside the fact that inference stays within the AWS boundary.

- **Guardrails handle topic and content restriction**, not access control. Confusing the two is a distractor: blocking a topic does not prevent retrieval of a document the user should not have, and ACL enforcement does not stop the model discussing a forbidden subject.

- **Q Business versus Q Developer.** Q Business is for organizational knowledge and end users. Q Developer is for code and AWS operations. Questions naming one and offering the other are testing that distinction.

## Limitations

- Security is inherited, not created. Q Business faithfully reproduces the source system's permissions, including its mistakes. An overshared SharePoint folder becomes an overshared answer, and the assistant makes previously obscure content trivially discoverable.

- Permission freshness is bounded by the crawl schedule. Between a revocation in the source and the next sync, the index still authorizes the removed user, and there is no immediate propagation mechanism.

- Identity mapping across heterogeneous systems is genuinely difficult. Systems that identify users by email, by SAM account name, by internal ID, or by group nesting all have to reconcile to one IdP identity, and mismatches fail in both directions.

- Not every connector captures ACLs with the same fidelity. Some sources have permission models the connector cannot fully represent, and custom sources rely on you supplying correct ACLs at ingestion.

- Generated answers can be wrong even when grounded, and a citation to a real document does not guarantee the summary of it is accurate. In regulated contexts an answer is a starting point, not an authoritative statement.

- Chat content auditing is not comprehensive by default. Reconstructing exactly what a user asked and what they were told requires configuring conversation logging in advance.

- Plugins execute with their own credentials, so a chat interface becomes an execution path into external systems, and prompt-driven action invocation is an injection surface distinct from anything in the retrieval path.

- Connector coverage, feature parity, and Region availability vary, and connectors to systems outside AWS require network reachability and stored credentials for each one, each of which is a standing access grant into that system.

- Cost scales with indexed document volume, connector count, and user subscriptions rather than with query volume, so a broad initial index is expensive whether or not anyone uses it.