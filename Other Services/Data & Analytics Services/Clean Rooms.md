# AWS Clean Rooms

AWS Clean Rooms is a managed service for running SQL analysis across datasets owned by two or more parties without any party gaining access to the other's raw rows. Each member keeps their data in their own account, in their own S3 bucket, cataloged in their own Glue Data Catalog. Nothing is copied into a shared bucket and nothing moves between accounts. What is shared is a configured table: a reference to the underlying data plus an analysis rule that declares which columns can be joined on, which can be aggregated, what minimum output threshold applies, and which query templates are permitted to run at all. The security value is that the restriction is enforced by the service at query planning time rather than by contract, masking pipelines, or reviewer discipline, and every query that runs is recorded with its full text and the identity that submitted it. The thing to hold onto is that Clean Rooms is not a data store or a sharing mechanism, it is a query-authorization boundary layered over data that never leaves its owner's account.

## How it works

- **Collaboration and members.** One account creates the collaboration and becomes the creator. It invites other AWS accounts by account ID and assigns abilities: which members can run queries, which member receives results, and which member pays for query compute. A member can contribute data, query, receive results, or any combination. Members join by creating a membership resource in their own account.

- **Configured tables.** Each contributor registers a Glue table as a configured table in their account, then associates it with the collaboration. The underlying S3 objects and Glue catalog entries never leave the owner's account, and the owner can disassociate at any time to revoke access immediately.

- **Analysis rules.** The enforcement primitive, attached per configured table by its owner. Three types: aggregation rules require every output to be an aggregate function and support minimum row constraints so small cohorts are suppressed; list rules allow row-level overlap output but restrict which columns can appear; custom rules permit specific approved analysis templates or specific accounts to author arbitrary SQL. The rule declares allowed join columns, allowed aggregate and dimension columns, and any output constraints.

- **Analysis templates.** Parameterized, pre-reviewed SQL stored in the collaboration. Under custom analysis rules a data owner approves a specific template rather than trusting the query author, so the only SQL that can touch their table is SQL they have read.

- **Query execution and results.** Queries run on Clean Rooms managed compute, or on Spark for larger workloads. Results are written only to the S3 location owned by the designated receiver member, and only that member sees them. Contributors see that a query ran, not the output.

- **Cryptographic Computing for Clean Rooms.** An optional client-side encryption path. Contributors encrypt data with a shared collaboration key using the C3R client before uploading, so the service joins on ciphertext and never handles plaintext. Columns are typed as sealed (retrievable but not joinable), fingerprint (joinable but not readable), or cleartext. The trade is a restricted set of supported operations.

- **Differential privacy.** An optional policy that injects calibrated noise into aggregate results and enforces a per-collaboration privacy budget, so repeated queries cannot be composed to isolate an individual. Available on a subset of Regions and query types.

- **Clean Rooms ML.** Lookalike modeling on a partner's data without exposing the seed audience or the training data, producing a similarity segment rather than raw records.

- **Encryption and network posture.** Data at rest stays under the owner's S3 and KMS configuration. Results are encrypted with the receiver's KMS key. Service access is via IAM, and the service role each member creates scopes what Clean Rooms may read on their behalf. Access can be reached over an interface VPC endpoint to keep control plane traffic off the public internet.

- **Logging.** CloudTrail records the control plane, including collaboration creation, member invitation, analysis rule changes, and query submission. Query logs, including submitted SQL and the member who ran it, are delivered to CloudWatch Logs in each member's own account, which is what makes a contributor able to audit what was asked of their data.

## Clean Rooms versus adjacent sharing mechanisms

| Mechanism | Data movement | Who enforces the restriction | Granularity of control | Raw row exposure to counterparty | Query-level audit |
|---|---|---|---|---|---|
| AWS Clean Rooms | None, data stays in owner's account | Clean Rooms analysis rules at query plan time | Column, join key, aggregate, minimum output threshold | None by default | Full SQL plus identity, per member |
| Lake Formation cross-account sharing | None, but consumer queries the shared table directly | Lake Formation permissions and data filters | Table, column, row filter | Yes, consumer reads rows within the filter | CloudTrail plus Lake Formation access logs |
| S3 cross-account bucket policy | None, but consumer reads objects | S3 and IAM policy | Object or prefix | Full raw object access | S3 server access logs or CloudTrail data events |
| Redshift data sharing | None, consumer queries producer cluster | Redshift grants | Schema, table, column, row-level security | Yes, within granted scope | Redshift audit logs |
| AWS Data Exchange | Yes, licensed copy delivered to subscriber | Subscription entitlement | Whole dataset or revision | Full, that is the point | CloudTrail on entitlement and export |
| Manual hashing plus S3 handoff | Yes, copy sent to counterparty | Nothing technical, contract only | Whatever was applied before sending | Depends entirely on the sender | None after handoff |

## What gets tested

- **Pick Clean Rooms when the requirement is joint analysis with no party seeing the other's records.** If the requirement is that a partner can query rows within a filter, that is Lake Formation. If the requirement is delivering a dataset to a customer, that is Data Exchange.

- **Analysis rules are set by the data owner, not the collaboration creator.** A common distractor gives the creator the ability to loosen another member's restrictions. Only the account that owns the configured table can change its analysis rule.

- **Aggregation rules with a minimum output constraint are the answer to re-identification of small cohorts.** Differential privacy is the answer when the concern is an adversary composing many legitimate aggregate queries over time to isolate an individual.

- **Cryptographic Computing is the answer when the requirement says AWS must not process plaintext**, or when a compliance control forbids the service handling readable identifiers. Standard Clean Rooms encrypts at rest and in transit but does process plaintext during query execution.

- **Only the designated receiver gets results.** Expect scenarios asking how to let one partner run the analysis while a different partner receives the output, which is configured through member abilities at collaboration creation.

- **Revocation is disassociating the configured table**, which takes effect immediately, because no copy of the data was ever made. Contrast with any answer that involves deleting a copy from a partner account.

- **Query logs land in each member's own CloudWatch Logs.** The audit trail belongs to the data contributor, which is what satisfies "the data owner must be able to prove what was asked of their data."

- **Data must be in S3 and cataloged in Glue**, or in a supported source such as Redshift or Athena-queryable formats. Clean Rooms does not ingest from operational databases directly.

## Limitations

- Analysis is SQL over cataloged data. There is no interactive access, no notebook against raw rows, and no arbitrary code execution against another member's data outside of Clean Rooms ML.

- Restrictions apply only to what the analysis rule declares. A permissive custom rule combined with an unreviewed template is an unrestricted query path, so the control is only as strong as the rule the owner writes.

- Aggregation thresholds reduce but do not eliminate inference risk. Without differential privacy, an analyst who can vary query predicates freely can still narrow a population across successive queries.

- Differential privacy and Cryptographic Computing are both Region-limited and constrain which SQL operations are available. Cryptographic Computing in particular restricts joins to fingerprint columns and blocks most functions on sealed columns.

- Members must all be AWS accounts. There is no path for a partner outside AWS to contribute data to a collaboration.

- Query compute is billed to the designated payer per Clean Rooms Processing Unit hour, and a poorly bounded custom query can be expensive without producing a usable result.

- Results delivery is one receiver per query configuration, so a symmetric exchange where both sides get output requires separate runs.

- Clean Rooms does not classify or discover sensitive data. Knowing that a column contains PII before deciding whether it is joinable is the owner's job, typically with Macie or Glue sensitive data detection upstream.