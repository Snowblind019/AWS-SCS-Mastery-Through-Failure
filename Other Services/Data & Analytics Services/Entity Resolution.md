# AWS Entity Resolution

AWS Entity Resolution is a managed matching service that links records referring to the same real-world entity across datasets that share no common unique identifier. You point it at data in S3, describe the schema, choose a matching technique, and it emits a Match ID grouping records it believes represent one entity. The security relevance is twofold and cuts both ways. Constructively, it is the mechanism for reconciling fragmented identity data during a privacy request, so a GDPR or CCPA deletion or access request can be satisfied across systems that spell the same person three different ways. Defensively, it is a service whose entire purpose is joining personal data, which makes it a high-consequence resource: the input is PII by definition, the output is a more complete profile than any single source held, and the service role it assumes has read access to whatever S3 locations you registered. The thing to hold onto is that Entity Resolution creates linkage that did not previously exist, so the control emphasis is on which data can be fed in, which principals can run workflows, and where the enriched output lands.

## How it works

- **Schema mappings.** You define a schema mapping describing each input dataset's columns and their semantic type, such as name, email address, phone number, address, or a unique ID. The mapping is what lets the service compare a `cust_email` column in one table against `contact_addr` in another.

- **Input sources.** Data is read from S3 through the Glue Data Catalog, or from a supported SaaS source through AppFlow. The data stays in your account and is read by an Entity Resolution service role that you create and scope.

- **Rule-based matching.** Deterministic rules you author, evaluated in priority order, for example match when hashed email matches, else match when phone plus postal code matches. Output includes which rule fired, which makes the result explainable and auditable. This is the technique to pick when a regulator or an auditor needs to know why two records were joined.

- **Machine learning matching.** An AWS-provided model performing probabilistic matching across multiple attributes with fuzzy comparison. No customer-specific training and no model hosting. Output includes a confidence score. Faster to stand up, less explainable.

- **Provider service matching.** Matching against a third-party identity provider's reference data through AWS Data Exchange, for example resolving to a provider's persistent identifier. This sends your data into a subscribed provider workflow, which is a distinct data-sharing decision from the other two techniques.

- **ID mapping workflows.** Rather than deduplicating within your data, these translate identifiers from one namespace to another, which is the pattern for joining a first-party ID to a partner's ID without merging the underlying records.

- **Hashed and normalized input.** Entity Resolution supports matching on SHA-256 hashed attributes, so a workflow can resolve identity without the service or a downstream consumer handling raw email addresses or phone numbers. Normalization can be applied before hashing so formatting differences do not defeat the hash comparison.

- **Encryption.** Input and output S3 objects use your bucket's SSE-S3 or SSE-KMS configuration. Workflows support a customer managed KMS key for data the service processes and stores, and the service role needs `kms:Decrypt` and `kms:GenerateDataKey` on that key, which means the key policy is a real access control point.

- **IAM surface.** `entityresolution:*` actions govern creating schema mappings, creating and starting workflows, and reading match results through `GetMatchId`. The service role is assumed by Entity Resolution with a confused-deputy guard on `aws:SourceAccount` and `aws:SourceArn`, and it carries whatever S3 and Glue read permissions you grant.

- **Output.** Results write to an S3 location you specify, containing the original records plus a Match ID and, for ML matching, a confidence score. Output can be restricted to a subset of columns so the enriched dataset does not carry every source attribute forward.

- **Logging and network.** CloudTrail records workflow creation, configuration changes, and job starts, and records `GetMatchId` lookups. CloudWatch captures job metrics and status. The API is reachable through an interface VPC endpoint to keep control plane calls off the public internet.

## Entity Resolution versus adjacent services

| Service | Purpose | Data movement | Sees raw PII | Explainability of the join | Primary security concern |
|---|---|---|---|---|---|
| Entity Resolution | Link and dedupe records for the same entity | None, reads S3 in place | Yes unless hashed input is used | High for rule-based, low for ML | Creates new linkage across previously separate PII |
| AWS Clean Rooms | Joint analysis across accounts with no raw exposure | None, each party keeps data | No, restricted by analysis rules | Query text is logged | Query authorization and re-identification via aggregates |
| Amazon Macie | Discover and classify sensitive data in S3 | None, samples objects | Yes, reports findings with location | Not applicable | Findings themselves reveal where PII lives |
| Glue FindMatches | ML dedupe inside an ETL job | Data flows through the Glue job | Yes | Low, model-driven | Requires you to label training data and manage the model |
| Amazon Neptune | Graph store for explicit relationships | Yes, data is loaded | Yes | Relationships are asserted, not inferred | Standard database access control |
| Fraud Detector | Score events for fraud risk | Events sent to the service | Yes, event attributes | Model explanations available | Model and event data handling |

## What gets tested

- **Pick Entity Resolution when the requirement is unifying records for the same person across systems with no shared key.** Pick Clean Rooms when the requirement is collaborating with another party without exposing rows. The two are frequently offered as distractors for each other because both involve matching identity data.

- **Rule-based over ML matching when the requirement mentions auditability, explainability, or a regulator asking why records were merged.** ML matching when the requirement emphasizes messy data, typos, and minimal setup.

- **Hashed input is the answer when the requirement says raw identifiers must not be processed.** Normalize before hashing, otherwise formatting variance breaks the match.

- **The service role is the confused-deputy surface.** Expect a scenario where a role can be assumed by Entity Resolution on behalf of another account, and the correct hardening is the `aws:SourceAccount` and `aws:SourceArn` condition in the trust policy.

- **KMS key policy governs whether a workflow can run at all.** If a workflow fails on encrypted input, the answer is granting the service role `kms:Decrypt` in the key policy, not loosening the bucket policy.

- **Output location is a data classification event.** The result set is more sensitive than any single input because it is a joined profile. Expect answers involving a separate bucket with its own KMS key, Lake Formation permissions, or a restricted output column list.

- **Provider service matching sends data outside your control boundary.** Any question with a data residency or third-party sharing constraint should exclude it in favor of rule-based or ML matching on your own data.

- **CloudTrail is the audit answer.** There is no separate Entity Resolution audit log, so workflow starts, configuration changes, and `GetMatchId` calls are visible only through CloudTrail.

## Limitations

- No customer-specific model training for ML matching. You cannot tune the model on your own labeled pairs, and you cannot inspect the features driving a match beyond the confidence score. If you need a trained, tunable matcher, Glue FindMatches is the alternative.

- Rule-based matching is only as good as the rules you write, and a permissive rule silently over-merges distinct people, which is a data integrity and privacy failure rather than a visible error.

- ML matching produces probabilistic output. False merges combine two individuals' records into one profile, and false splits defeat a deletion request. Neither surfaces as a job failure.

- Input must be in supported S3 formats through Glue, or an AppFlow-supported SaaS source. There is no direct connector to an operational database, so getting data in is a separate pipeline with its own security controls.

- Matching is a batch or on-demand workflow, not a synchronous lookup for arbitrary records. `GetMatchId` returns an ID for a record against an existing workflow output rather than performing live resolution.

- Regional availability is limited compared to core services, which matters for data residency requirements.

- Cost scales with records processed per run, so an incremental strategy matters. Reprocessing a full dataset repeatedly is both expensive and an unnecessary repeated exposure of PII to the service.

- The service does not classify or discover sensitive fields. Deciding that a column contains PII and should be hashed before ingestion is upstream work, typically Macie or Glue sensitive data detection.