# AWS CloudFormation: Interview Questions

| | |
|---|---|
| **Cluster** | 05 Infrastructure as Code |
| **Role** | Bridge (anchor is [Terraform](../terraform/README.md)) |
| **Prerequisites** | All previous CloudFormation files |
| **Target version** | CloudFormation service as of 2025-2026 |
| **Capstone phase** | Phase 2 (Months 5-8) |

> Twenty-eight questions from junior to principal level, each with a model answer. Many interviewers ask CloudFormation questions as "compare it with Terraform", so several answers make that comparison explicit. The senior and principal answers show whether a candidate has operated stacks in production or has only written templates.

---

## Junior

### Q1. What is CloudFormation and what is a stack? (Junior)

**Answer.** CloudFormation is AWS's managed infrastructure-as-code service. You submit a YAML or JSON template describing resources; CloudFormation builds a dependency graph and creates, updates or deletes the resources through AWS APIs. A **stack** is one deployed instance of a template in one account and region. It owns its resources, stores the template and parameter values, records events, and is the unit of update, rollback and deletion.

### Q2. Which template sections exist, and which is required? (Junior)

**Answer.** `AWSTemplateFormatVersion`, `Description`, `Metadata`, `Parameters`, `Rules`, `Mappings`, `Conditions`, `Transform`, `Resources` and `Outputs`. Only `Resources` is required.

### Q3. What is the difference between `Ref` and `Fn::GetAtt`? (Junior)

**Answer.** `Ref` returns a parameter's value or a resource's primary identifier, which depends on the type: a bucket name, a VPC ID, or an ARN for some types. `Fn::GetAtt` returns a named attribute, such as `Bucket.Arn` or `LoadBalancer.DNSName`. Both create an implicit dependency, so the referenced resource is created first.

### Q4. What are pseudo parameters? Give examples. (Junior)

**Answer.** Built-in values CloudFormation provides without declaring them: `AWS::AccountId`, `AWS::Region`, `AWS::Partition`, `AWS::StackName`, `AWS::StackId`, `AWS::URLSuffix`, `AWS::NoValue` and `AWS::NotificationARNs`. They keep templates portable across accounts, regions and partitions; ARNs should use `${AWS::Partition}` rather than a hard-coded `aws`.

### Q5. What happens if a resource fails during stack creation? (Junior)

**Answer.** By default CloudFormation rolls back: it deletes every resource it created in that operation, and the stack ends in `ROLLBACK_COMPLETE`. That stack cannot be updated; you delete it and create it again. You can change this with `--on-failure DO_NOTHING` or `--disable-rollback`, which keeps the successful resources for debugging.

### Q6. What are capabilities and why do they exist? (Junior)

**Answer.** They are explicit acknowledgements that a template does something sensitive. `CAPABILITY_IAM` and `CAPABILITY_NAMED_IAM` acknowledge that IAM resources will be created (named ones need the latter), and `CAPABILITY_AUTO_EXPAND` acknowledges that macros will transform the template. Without them the API returns `InsufficientCapabilitiesException`, so nobody grants IAM permissions without noticing.

### Q7. How do you pass values between stacks? (Junior)

**Answer.** Either through `Outputs` with `Export` consumed by `Fn::ImportValue`, or by writing values to SSM Parameter Store and reading them with SSM parameter types or dynamic references. Nested stacks pass values through child `Outputs` and `!GetAtt Child.Outputs.Name`.

---

## Mid-Level

### Q8. What is a change set and how does it compare with `terraform plan`? (Mid-level)

**Answer.** A change set is a server-side preview of what an update will do per resource: `Add`, `Modify`, `Remove`, `Import`, `Dynamic`, and for modifications whether `Replacement` is `True`, `False` or `Conditional`. Like a saved Terraform plan, it can be reviewed and then executed exactly as reviewed. Differences: it is stored in AWS rather than in a file, it does not account for drift (CloudFormation diffs templates, not reality), and it cannot predict runtime failures. `aws cloudformation deploy` creates and executes one in a single step.

### Q9. Explain update behaviours. Why does changing some properties replace the resource? (Mid-level)

**Answer.** Each property in the resource reference declares an update behaviour: no interruption, some interruption, or replacement. Replacement happens when the underlying API cannot change that attribute in place, for example a bucket name, an RDS identifier, or an EC2 AMI. CloudFormation creates the new resource, repoints dependents, and deletes the old one in the cleanup phase. Replacement means a new physical ID, possible data loss and dependents updating, so you check the `Replacement` column before executing.

### Q10. `DeletionPolicy` versus `UpdateReplacePolicy`? (Mid-level)

**Answer.** `DeletionPolicy` controls what happens to the resource when it is removed from the template or the stack is deleted: `Delete`, `Retain`, `RetainExceptOnCreate`, or `Snapshot` for supported types. `UpdateReplacePolicy` controls what happens to the **old** physical resource when an update replaces it. A database needs both: without `UpdateReplacePolicy`, an accidental replacement deletes the old instance even if `DeletionPolicy` is `Retain`. RDS clusters and standalone instances default to `Snapshot` for deletion.

### Q11. How do conditions work, and what happens to a resource whose condition becomes false? (Mid-level)

**Answer.** Conditions are booleans built from parameters with `Fn::Equals`, `Fn::And`, `Fn::Or` and `Fn::Not`. A resource or output with `Condition: X` exists only when X is true, and `Fn::If` chooses property values, with `AWS::NoValue` removing a property entirely. If a condition flips to false on update, CloudFormation **deletes** that resource, which is why conditions on stateful resources deserve extra care.

### Q12. What are dynamic references and when do you use them? (Mid-level)

**Answer.** Placeholders such as `{{resolve:ssm:/path}}`, `{{resolve:ssm-secure:/path:1}}` and `{{resolve:secretsmanager:id:SecretString:key}}` that CloudFormation resolves at execution time. Use them for values that should not live in the template or parameters, especially secrets. The resolved secret is not shown in the template or `describe-stacks`. `ssm-secure` works only with specific properties, and Secrets Manager references are the general solution.

### Q13. Exports versus SSM parameters for cross-stack references: trade-offs? (Mid-level)

**Answer.** Exports are enforced: a consumer cannot reference a missing export, and the producer cannot change or delete an export while it is imported. That is safe but couples stacks, so producer refactors and deletions become multi-step migrations. SSM parameters are loose: producers change freely, they work across tools (Terraform writes, CloudFormation reads), and values resolve on every consumer update. The cost is that nothing prevents a breaking change. Most platform teams prefer SSM with naming conventions and reserve exports for small, stable values.

### Q14. How does drift detection work, and what are its limits? (Mid-level)

**Answer.** `detect-stack-drift` compares the current configuration of each supported resource with the properties **explicitly declared** in the template and reports `IN_SYNC`, `MODIFIED`, `DELETED` or `NOT_CHECKED`. Limits: undeclared properties are not checked, not all types support it, it never fixes anything, and redeploying an unchanged template does not correct drift because CloudFormation diffs templates. You remediate by changing reality back or by codifying the change.

### Q15. Nested stacks versus Registry modules versus separate stacks? (Mid-level)

**Answer.** Nested stacks are child stacks created from S3 templates. They are good for reuse and for getting past the 500-resource limit, but the whole tree updates and rolls back together. Registry modules are fragments expanded into the parent template at deployment, which gives reuse without child stacks. Separate root stacks connected by contracts are what you use for different lifecycles and owners. Nested stacks do not reduce blast radius; separate stacks do.

---

## Senior

### Q16. A stack is in `UPDATE_ROLLBACK_FAILED`. Walk through recovery. (Senior)

**Answer.** Find the resource whose rollback failed and why, usually because it was modified or deleted outside CloudFormation, or a dependency such as a KMS key or role disappeared. Fix the cause if possible, then `continue-update-rollback`. If the resource cannot be restored, continue with `--resources-to-skip` (nested resources as `Nested.Logical`), which marks it complete without touching it. Afterwards the template and reality disagree, so run drift detection and reconcile before the next update. To prevent it: no out-of-band changes, run drift detection before risky updates, never delete service roles or keys that stacks depend on.

### Q17. How do you prevent a database from being destroyed by a CloudFormation change? (Senior)

**Answer.** Use layers. In the template: `DeletionPolicy` and `UpdateReplacePolicy` of `Snapshot` or `Retain`, no physical name, and `DeletionProtection: true` on the instance. On the stack: a stack policy denying `Update:Replace` and `Update:Delete` on that logical ID, termination protection, and a separate long-lived data stack. In the pipeline: change set review that fails on `Replacement: True` for protected types, and a Guard rule requiring the policies. Native deletion protection is the only control that also covers console and API calls.

### Q18. Design a least-privilege CI/CD flow for CloudFormation. (Senior)

**Answer.** GitHub Actions authenticates with OIDC; no static keys. A plan role, trusted for pull requests and main, may only create and describe change sets, read the SSM contract, and pass one service role. A deploy role, trusted only for an approved GitHub environment, may only execute change sets and describe stacks. The CloudFormation service role, scoped to the stack family's resource types and name prefixes and with a permissions boundary for any roles it creates, is the only principal that touches resources. Lint, Guard and Checkov run before the change set; the change set summary is posted to the PR; rollback triggers watch SLO alarms after execution.

### Q19. How do custom resources work and what can go wrong? (Senior)

**Answer.** A `Custom::` resource sends Create, Update and Delete requests to a Lambda function or SNS topic with properties and a pre-signed S3 `ResponseURL`. The handler must PUT `SUCCESS` or `FAILED` with a `PhysicalResourceId`. Things that go wrong: not responding on an exception, so the stack hangs until `ServiceTimeout` or one hour; no network path to S3 for the response; returning a new physical ID on update, which makes CloudFormation send a Delete for the old one, sometimes destroying data unexpectedly; Delete handlers that fail and block stack deletion; responses over 4 KB; and secrets in `Data` without `NoEcho`. Prefer a Registry resource type for anything reusable.

### Q20. How does resource import work? (Senior)

**Answer.** Create a change set of type `IMPORT` with a template that adds only the resources being imported, each with a `DeletionPolicy`, and a `--resources-to-import` list giving each type, logical ID and identifier (for example `BucketName`). Execute it, and the resources are adopted without changes. Then align their properties with a normal update. For estates built by hand, the IaC generator scans the account and writes a template to import. `--import-existing-resources` adopts named resources during a normal deploy.

### Q21. Why is `DesiredCount` on an ECS service, or capacity on an ASG, a classic CloudFormation trap? (Senior)

**Answer.** Autoscaling changes the live value, so the template and reality diverge. CloudFormation sends the template value only when the template value changes. When someone later edits that property, or a replacement occurs, the count is reset to the template value, possibly scaling a busy service down. Mitigations: set the count once and leave it unchanged, give autoscaling the minimum and maximum, and treat the template value as the initial value. In ASGs, use `UpdatePolicy` with `AutoScalingRollingUpdate` and avoid editing `DesiredCapacity`.

### Q22. What are CloudFormation Hooks and how do they differ from cfn-guard in CI? (Senior)

**Answer.** Hooks run inside CloudFormation before resources, stacks or change sets are provisioned, for every stack in the account and region, regardless of which pipeline or person deployed it. They can be Guard hooks (rules in S3), Lambda hooks, or Registry hooks, with `FailureMode` `FAIL` or `WARN`. cfn-guard in CI gives fast feedback but can be bypassed by deploying outside the pipeline. Run the same rules in both places: CI for developer experience, Hooks for enforcement. Control Tower proactive controls are implemented as Hooks.

### Q23. How would you roll out a security baseline to 200 accounts? (Senior)

**Answer.** Use a service-managed StackSet from a delegated administrator, targeting OUs with auto-deployment so new accounts receive it. Roll out in waves with `MaxConcurrentPercentage` of 10-25%, zero failure tolerance and a low-risk region first; enable managed execution to queue operations. Keep the template small (roles, Config, CloudTrail, SSM defaults), retain stacks on account removal for security resources, run StackSet drift detection, and use `list-stack-set-operation-results` to triage per-account failures.

---

## Principal

### Q24. When would you choose CloudFormation or CDK over Terraform for an organisation? (Principal)

**Answer.** Choose CloudFormation or CDK when the estate is AWS-only and you value no state to operate, transactional rollbacks, day-one support for some AWS features, Hooks enforcement in the account, StackSets and Control Tower integration, and AWS support covering the tool. Choose Terraform when you manage several clouds or SaaS providers (Kubernetes, GitHub, Vault, Grafana, Datadog) in one workflow, want a wide module ecosystem, or need features such as `ignore_changes` and flexible targeting. Many organisations use both: Control Tower and StackSets for the account foundation, Terraform or CDK for workloads, connected by SSM contracts. In this capstone, [ADR-0001](../../capstone/adrs/0001-terraform-as-iac-anchor.md) records Terraform as anchor for multi-provider coverage and CloudFormation as a bridge.

### Q25. Compare CloudFormation's state and rollback model with Terraform's. What are the operational consequences? (Principal)

**Answer.** CloudFormation keeps state server-side, serialises operations per stack and rolls back by default: there is no state file to secure, lock or corrupt, and a failed change returns to a known configuration. The costs are slow operations on large stacks, rollbacks that can themselves fail and need manual recovery, and less control (no `-target`, no `ignore_changes`, refactoring tools only since 2025). Terraform's client-side state is a liability you must protect (encryption, locking, secrets in state), and failures leave partial applies you fix forward, but you get fine-grained state manipulation and multi-provider graphs. Operationally, CloudFormation teams build runbooks around stack states; Terraform teams build them around state files.

### Q26. A team's single stack has 450 resources and takes 40 minutes to update. What do you do? (Principal)

**Answer.** Split it by lifecycle and ownership: network, data and per-service stacks connected by SSM contracts. Migrate without replacing resources by using stack refactoring to move resources between stacks, or, where refactoring is not available, set `DeletionPolicy: Retain`, remove the resources from the old stack, and import them into the new one. Do it in small batches, running drift detection before each. Add policy so no new stack can approach the limit, measure update duration, and prefer CDK or modules to keep templates DRY. Nested stacks alone would not reduce the 40-minute coupled updates.

### Q27. How do you make CDK safe at enterprise scale? (Principal)

**Answer.** Bootstrap with a custom qualifier and least-privilege `--cloudformation-execution-policies` instead of the default admin policy, and trust only pipeline accounts. Publish internal L2 and L3 constructs with secure defaults, run cdk-nag and assertion or snapshot tests in CI, commit `cdk.context.json`, pin library and CLI versions, and review the CloudFormation change set rather than only `cdk diff`. Watch for logical ID changes on refactors. Back all of it with Hooks so non-compliant synthesised templates are rejected anyway. CDK is a generator, and everything you know about CloudFormation operations still applies.

### Q28. How does CloudFormation fit into a DevSecOps gate model? (Principal)

**Answer.** At Gate 5 (IaC policy as code) templates are scanned by cfn-lint, cfn-guard and Checkov in pull requests, with the same Guard rules enforced server-side by Hooks. Secrets never enter templates (dynamic references, service-managed secrets), and Gate 1 secret scanning covers parameter files. Pipelines use OIDC with separated plan, deploy and service roles. After deployment, Config rules, Security Hub and scheduled drift detection provide detective controls, and findings flow to the ASPM hub. The change set is the auditable artefact connecting a pull request to the API calls in CloudTrail.

---

Back to [README.md](README.md).
