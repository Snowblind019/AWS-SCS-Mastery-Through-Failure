# Amazon Q Developer

Amazon Q Developer is AWS's AI coding assistant, embedded in IDEs, the AWS Console, the CLI, and services like the Lambda and CloudWatch consoles, and specialized for AWS development: it is grounded in AWS documentation, SDKs, the CDK, CloudFormation, and IAM behavior, and it can run multi-step agentic tasks such as implementing a feature across files, reviewing code, or transforming a Java version. For a security engineer its relevance is dual. It is a productivity tool that reduces IAM mistakes by generating least-privilege policies and flagging wildcards, and it is a tool that reads your source code, executes actions with your credentials in agentic modes, and can be configured to draw on your private repositories, which makes its data handling and permission model a real governance question rather than a vendor assurance to take on faith. The most important administrative fact is the Pro tier's IP indemnification and reference tracking, which flags generated code resembling training data and is the answer to the intellectual property objection that blocks many enterprise coding assistants. The thing to hold onto is that Q Developer runs with the developer's own AWS permissions in its agentic and console modes, so what it can do to your account is bounded by that identity, and what it does with your code is bounded by the tier and its data settings.

## How it works

- **Tiers.** A Free tier with monthly limits and no administrative controls, and a Pro tier subscribed per user through IAM Identity Center. Pro adds higher limits, administrative policy management, IP indemnification, and the ability to control whether content is used for service improvement. The tier distinction is a security decision, not just a quota one, because the controls that matter to an enterprise exist only in Pro.

- **Surfaces.** IDE plugins for VS Code, JetBrains, Visual Studio, and Eclipse; the command line including an agentic CLI; the AWS Console with contextual help and error diagnosis; and inline assistance in the Lambda, CloudWatch, and other service consoles. Each surface has a different data and permission profile.

- **Inline suggestions versus agentic tasks.** Inline code completion sends surrounding code context to generate suggestions. Agentic capabilities such as `/dev` for feature implementation, `/review` for code review, `/transform` for language and framework upgrades, and `/test` for test generation operate across multiple files and can propose and apply changes, which is a materially larger surface than autocomplete.

- **IAM and credentials.** In the Console, CLI, and agentic modes that call AWS, Q Developer operates under the signed-in principal's IAM permissions. It does not grant new access. When it performs an AWS action or reasons about a policy, it is bounded by, and can be scoped through, that identity. This is why an over-permissioned developer identity is the real risk multiplier, not Q Developer itself.

- **Reference tracking.** Suggestions that resemble open-source training data are flagged with the source repository and its license, so a developer can decline code that would import a license obligation. Administrators can configure whether such suggestions are shown or suppressed entirely.

- **Data handling.** On the Pro tier, prompts, code context, and completions are not used to improve the service, and content stays within the AWS boundary. On the Free tier, content may be used for service improvement unless opted out where that option exists. This difference is the crux of most enterprise adoption decisions.

- **Administrative controls.** Through IAM Identity Center and the Q Developer console, administrators assign subscriptions, set whether telemetry and content sharing are on, manage customization access, and enable or disable specific features org-wide.

- **Customization.** Pro supports customizing suggestions on your own private code, so completions reflect internal libraries and patterns. The customization is built from repositories you connect, is isolated to your organization, is encryptable with a customer managed KMS key, and is access-controlled so only designated users receive it. A customization is a distilled representation of proprietary code and is therefore a sensitive artifact in its own right.

- **Network.** IDE and CLI usage authenticate through Identity Center or Builder ID. Enterprise deployments use Identity Center federation, and console usage inherits the session's network and identity controls.

- **Logging.** Console interactions with Q Developer are recorded in CloudTrail like other console activity. IDE and CLI telemetry is governed by the administrative settings rather than appearing in your CloudTrail, so the audit trail for developer-surface usage is thinner than for console usage, which is worth knowing when a requirement calls for auditing assistant use.

## Q Developer versus adjacent assistants and tools

| Option | Domain specialization | Runs with your AWS credentials | Private code customization | IP indemnification | Data used for training | Primary surface |
|---|---|---|---|---|---|---|
| Amazon Q Developer | AWS development, IAM, IaC | Yes, in console, CLI, and agentic modes | Yes, on Pro with your repos | Yes, on Pro | No on Pro | IDE, CLI, and AWS consoles |
| Amazon Q Business | Organizational knowledge | No, enforces source ACLs | Not applicable | Not for code | No | Business web experience |
| GitHub Copilot | General coding | No | Enterprise fine-tuning | Enterprise tier terms | Per tier settings | IDE |
| Bedrock with a coding model | Whatever you build | Only if you build it in | Whatever you build | Depends on the model terms | Depends | Applications you build |
| CodeWhisperer | Predecessor, now part of Q Developer | Legacy | Legacy | Legacy | Legacy | Legacy |
| IAM Access Analyzer policy generation | IAM policy from activity | Reads CloudTrail, generates policy | Not applicable | Not applicable | No | Console and API |

## What gets tested

- **Q Developer operates with the developer's IAM permissions** in AWS-acting modes, so it cannot exceed what that identity allows. The security answer to limiting its reach is scoping the developer's role, not restricting Q Developer separately.

- **Pro tier is required for enterprise controls.** IP indemnification, opting content out of service improvement, private customization, and administrative management all live in Pro. A question about deploying a coding assistant under compliance constraints points at Pro, not Free.

- **Reference tracking is the intellectual property answer.** When a scenario raises the concern that generated code might carry open-source license obligations, reference tracking with source and license attribution, plus the option to suppress flagged suggestions, is the control.

- **Customization is built from proprietary code and is sensitive.** It should be encrypted with a customer managed key and access-controlled to specific users, because it is a distilled form of the source repositories it was built from.

- **IAM policy generation and review** is a legitimate use, and the answer to "reduce over-permissive policies during development" alongside IAM Access Analyzer, which generates policies from actual CloudTrail activity rather than from a model.

- **Data residency and training.** Pro content is not used for training and stays in the AWS boundary, which is the answer to the objection that a coding assistant exfiltrates source code to a third-party model provider.

- **Agentic modes are a larger surface than autocomplete.** `/dev`, `/transform`, and CLI agents read and modify multiple files and can execute actions, so the review-before-apply discipline and the developer's permission scope both matter more than for inline suggestions.

- **CloudTrail covers console interactions**, but IDE and CLI usage auditing depends on administrative telemetry settings rather than CloudTrail, which is the honest answer when a requirement asks for a complete audit trail of assistant usage.

- **Q Developer versus Q Business.** Code, IAM, and AWS operations point at Q Developer. Organizational documents and end-user knowledge point at Q Business. The two are routinely offered as distractors for each other.

## Limitations

- Generated code and policies can be wrong or insecure. The assistant reduces certain classes of mistake but does not remove the need for review, and a confidently generated IAM policy still needs validation before it ships.

- Its AWS actions are bounded by the developer's identity, which cuts both ways: a developer with broad permissions gets an assistant that can exercise them, so the tool amplifies whatever the identity already allows.

- Free tier lacks the controls an enterprise needs, and using it in an organization means content may be used for service improvement and no administrative governance applies.

- Customization exposes proprietary code to the customization build process and produces an artifact derived from that code. Without KMS encryption and access control, the customization itself becomes an uncontrolled copy of intellectual property.

- Audit coverage of developer-surface usage is limited. There is no CloudTrail record of every IDE interaction, so proving what the assistant was asked and what it produced across a team is not straightforward.

- Reference tracking flags resemblance to known training data but is not a guarantee of license cleanliness, and it does not catch every case, so it reduces rather than eliminates IP risk.

- Agentic changes across many files can introduce subtle errors that are harder to catch in review than a single suggestion, and applying them without careful diff review is a real risk.

- Effectiveness depends on AWS-specific context, so it is strongest inside the AWS ecosystem and less differentiated for general-purpose code where a general coding assistant may perform comparably.

- Feature availability differs across IDEs and surfaces, so a capability present in VS Code may be absent or in preview elsewhere, which complicates standardizing a team on it.