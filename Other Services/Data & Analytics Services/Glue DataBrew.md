# AWS Glue DataBrew

AWS Glue DataBrew is a visual data preparation service that applies a versioned sequence of transformations to a dataset without writing Spark code. You point a project at data in S3, the Glue Data Catalog, Redshift, or a JDBC source, sample it, build a recipe in the UI, and run a serverless job that applies the recipe to the full dataset and writes output to S3. Its security relevance is that it is a governed redaction and normalization stage: it sits between raw ingestion and anything downstream that should not see raw sensitive values. Recipes support hashing, masking, replacement, and cryptographic operations on specific columns, and because a recipe is a named, versioned artifact rather than ad hoc code, the transformation applied to a dataset is itself auditable evidence. The counterweight is that DataBrew is a service that reads your most sensitive raw data by design, so the IAM role it assumes, the KMS keys it can use, and the output location it writes to are the three things that matter. The thing to hold onto is that DataBrew is a versioned transformation layer whose security value comes from the recipe being reviewable and reproducible, and whose security risk is concentrated in the service role that reads the pre-redaction data.

## How it works

- **Datasets.** A dataset is a pointer to source data plus format metadata. Supported sources include S3, Glue Data Catalog tables, Redshift, RDS through the catalog, and Data Exchange datasets. The data is not copied into DataBrew at registration, the service reads it when a project or job runs.

- **Projects and sampling.** A project loads a sample, by default 500 rows, into an interactive session for recipe authoring. That sample is real production data rendered in a console, which makes console access to a project an effective read grant on sensitive columns even before any job runs.

- **Recipes.** An ordered, versioned list of transformation steps. Recipes are publishable, so a specific version can be pinned to a job, and the version history is the record of what transformation was applied when. Recipes can be exported as JSON and managed in source control alongside infrastructure code.

- **Jobs.** Two types. A recipe job applies a published recipe to the full dataset and writes output. A profile job analyzes the dataset and produces a data quality profile including null counts, distinct values, distributions, and correlations, written to S3 as JSON.

- **PII detection in profile jobs.** Profile jobs can be configured to detect personally identifiable information, flagging columns that appear to contain identifiers such as email addresses, phone numbers, government IDs, or credit card numbers. That output is the input to deciding which columns need masking, and it overlaps with what Macie does at the object level.

- **Redaction and obfuscation transforms.** The security-relevant transform families are masking (replace characters with a mask symbol, optionally preserving a prefix or suffix), hashing (one-way, non-reversible), replacement with a fixed or random value, deterministic encryption and decryption using a KMS key so the same input maps to the same ciphertext and can be reversed by an authorized job, and column deletion. Deterministic encryption is the one that preserves joinability without preserving readability.

- **Data lineage.** DataBrew tracks the path from source dataset through recipe versions to output, which is the artifact you produce when asked how a redacted dataset was derived.

- **IAM and the service role.** DataBrew assumes a role you create to read sources, write outputs, and use KMS keys. That role is the highest-privilege element in the design because it reads pre-redaction data. `databrew:*` actions govern who can create datasets, open projects, publish recipes, and start jobs, and opening a project is the action that exposes sample data.

- **Encryption.** Job output is encrypted with SSE-S3 or SSE-KMS at the output location. Jobs can be configured with a KMS key for the data DataBrew handles, and the service role needs corresponding key policy grants. Deterministic encryption transforms require an explicitly specified KMS key.

- **Network.** Recipe jobs can be run inside a VPC by attaching a Glue connection with subnet and security group configuration, which is required when the source is a private RDS or Redshift instance and is how you keep job traffic off the public network.

- **Scheduling and logging.** Jobs run on demand, on a cron schedule, or triggered externally through EventBridge or Step Functions. CloudTrail records dataset, project, recipe, and job API calls. Job run logs go to CloudWatch Logs, and job history records which recipe version was applied to which dataset.

## DataBrew versus adjacent data preparation and protection services

| Service | Interface | Redaction and masking capability | Where the transformation is recorded | Detects sensitive data | Best fit |
|---|---|---|---|---|---|
| Glue DataBrew | Visual, no code | Mask, hash, replace, deterministic KMS encryption, drop column | Versioned recipe plus job history | Yes, in profile jobs | Analyst-driven redaction and normalization with an auditable artifact |
| Glue ETL | PySpark or Scala | Anything you code, plus Glue sensitive data detection transform | Job script in source control | Yes, detection transform | Engineered pipelines and complex logic |
| Amazon Macie | Managed scanning | None, discovery only | Findings, not transformations | Yes, that is its purpose | Knowing where sensitive data is in S3 |
| Lake Formation | Policy engine | Column, row, and cell filtering at query time | Permission grants | No | Restricting what a querying principal sees without changing the data |
| Comprehend PII | API and Lambda | Redaction of PII in unstructured text | Whatever calls it | Yes, in free text | Documents and text bodies rather than tabular columns |
| Kinesis Firehose with Lambda | Code in a transform function | Anything you code, applied in stream | Function version | No | Redaction in the ingestion stream before landing |

## What gets tested

- **Hashing versus deterministic encryption versus masking.** Hashing is irreversible and joinable. Deterministic encryption is reversible by a principal with the KMS key and joinable. Masking is irreversible, not joinable, and preserves format for display. Pick by whether the value must be recoverable and whether records must still be linked.

- **Redaction before analysis versus restriction at query time.** DataBrew produces a physically redacted copy, which is the answer when the data will be shared, exported, or handed to a third party. Lake Formation filters at query time and leaves the data intact, which is the answer when different principals need different views of the same table.

- **The service role reads unredacted data.** Expect hardening questions answered by scoping the role to specific source prefixes, adding `aws:SourceAccount` and `aws:SourceArn` conditions on the trust policy, and separating the role that reads raw input from any role with access to the redacted output bucket.

- **Opening a project displays live sample data.** Restricting `databrew:CreateProject` and `databrew:DescribeProject` is a data access control, not just a management control. A common design puts recipe authoring on a de-identified sample and job execution on production data under a separate role.

- **Profile job PII detection versus Macie.** Profile jobs flag sensitive columns in a specific dataset you are already preparing. Macie discovers sensitive data across buckets you have not curated. If the question is about finding unknown exposure, the answer is Macie.

- **Private source connectivity is a Glue connection with VPC configuration**, not a public endpoint or a NAT-only answer. The job runs in your subnets with your security groups.

- **Recipe versions are the audit artifact.** When a question asks how to prove which transformation was applied to a released dataset, the answer is the published recipe version pinned to the job run, plus CloudTrail and job history.

- **KMS key policy grants to the DataBrew service role** are required for both encrypted sources and deterministic encryption transforms. A job failing on encrypted input is a key policy problem, not a bucket policy problem.

## Limitations

- No code, which is also the ceiling. Anything requiring conditional logic across rows, custom parsing, or an external lookup exceeds the recipe model and belongs in Glue ETL or a Lambda transform.

- Interactive sessions are billed per minute and jobs per node-hour, so an analyst leaving a project open is a live cost and a live exposure of sample data.

- Sampling is a correctness risk. A recipe built on 500 rows may not handle formats, nulls, or edge cases present in the full dataset, and a masking rule that misses a variant silently passes raw values through.

- PII detection in profile jobs is heuristic. It will miss custom identifiers, internal account numbers, and free-text fields containing identifiers, so it is a starting point rather than a compliance guarantee.

- Output is a new copy in S3. The original unredacted data still exists and still needs its own lifecycle, access control, and deletion policy. DataBrew does not reduce the sensitivity of the source.

- Hashing without a salt is vulnerable to precomputation on low-cardinality fields such as phone numbers or postal codes. The recipe transform alone does not make a value unrecoverable in practice.

- Deterministic encryption preserves the ability to link records, which means it preserves re-identification risk when combined with other datasets. It is pseudonymization, not anonymization, and most privacy regimes treat it as still-personal data.

- Regional and source support is narrower than Glue ETL, and very large datasets are more economically processed with a Spark job than with a recipe job.