# AWS CloudFormation: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 05 Infrastructure as Code |
| **Role** | Bridge (anchor is [Terraform](../terraform/README.md)) |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | AWS CLI v2, cfn-lint 1.x, cfn-guard 3.x, Rain 1.x, SAM CLI 1.x, AWS CDK v2 |
| **Capstone phase** | Phase 2 (Months 5-8) |

> Task-oriented reference for the `aws cloudformation` CLI and the template toolchain. Every command has a one-line explanation. Examples assume `AWS_PROFILE` and `AWS_REGION` are exported and use `$STACK` for the stack name. The AWS CLI has no short flags, so all options are shown in long form.

---

## 1. Terraform-to-CLI Quick Map

| You want to... | Terraform | CloudFormation CLI |
|---|---|---|
| Check syntax | `terraform validate` | `cfn-lint template.yaml` (thorough), `aws cloudformation validate-template` (basic) |
| Preview | `terraform plan -out=tfplan` | `aws cloudformation create-change-set ...` then `describe-change-set` |
| Apply the preview | `terraform apply tfplan` | `aws cloudformation execute-change-set` |
| Preview and apply in one step | `terraform apply` | `aws cloudformation deploy` |
| List managed resources | `terraform state list` | `aws cloudformation list-stack-resources` |
| Show outputs | `terraform output` | `aws cloudformation describe-stacks --query "Stacks[0].Outputs"` |
| Detect drift | `terraform plan -refresh-only` | `aws cloudformation detect-stack-drift` |
| Adopt a resource | `import {}` block | `create-change-set --change-set-type IMPORT` |
| Destroy | `terraform destroy` | `aws cloudformation delete-stack` |

---

## 2. Validate and Lint

```bash
cfn-lint template.yaml                                   # schema, types, enums, best-practice rules
cfn-lint "templates/**/*.yaml"                           # glob across a tree (quote it)
cfn-lint --regions eu-west-1 us-east-1 -- template.yaml  # validate against several regions' resource availability
cfn-lint --ignore-checks W3005 -- template.yaml          # suppress a rule for this run
cfn-lint --include-checks I -- template.yaml             # also show informational findings
cfn-lint --format sarif template.yaml > cfn-lint.sarif   # SARIF for GitHub code scanning
cfn-lint --build-graph template.yaml                     # write template.dot dependency graph

aws cloudformation validate-template --template-body file://template.yaml          # syntax + required capabilities
aws cloudformation validate-template --template-url https://<BUCKET>.s3.amazonaws.com/t.yaml
aws cloudformation get-template-summary --template-body file://template.yaml       # parameters, resource types, capabilities
```

---

## 3. Policy as Code

```bash
cfn-guard validate --data template.yaml --rules rules/                      # evaluate all rule files against a template
cfn-guard validate --data "templates/" --rules rules/ --show-summary fail   # only print failures
cfn-guard validate --data template.yaml --rules rules/ --output-format sarif # SARIF output for code scanning
cfn-guard test --rules-file rules/s3.guard --test-data rules/tests/s3_tests.yaml  # unit-test rules
cfn-guard rulegen --template template.yaml                                  # draft rules from an existing template
checkov --directory infra/cloudformation --framework cloudformation        # Checkov's CloudFormation checks
```

Minimal Guard rule:

```text
let buckets = Resources.*[ Type == 'AWS::S3::Bucket' ]

rule s3_encrypted_and_private when %buckets !empty {
    %buckets.Properties.BucketEncryption exists
    %buckets.Properties.PublicAccessBlockConfiguration.BlockPublicAcls == true
    %buckets.Properties.PublicAccessBlockConfiguration.RestrictPublicBuckets == true
}
```

---

## 4. Deploy (High-Level)

```bash
aws cloudformation deploy \
  --stack-name "$STACK" \
  --template-file template.yaml \
  --parameter-overrides Environment=dev DesiredCount=2 \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn "$CFN_ROLE" \
  --tags project=capstone env=dev                     # create or update via an auto-generated change set

aws cloudformation deploy ... --no-execute-changeset  # create the change set only; prints the describe command
aws cloudformation deploy ... --no-fail-on-empty-changeset  # exit 0 when nothing changed (CI-friendly)
aws cloudformation deploy ... --s3-bucket "$CFN_BUCKET" --s3-prefix templates  # templates over 51,200 bytes
aws cloudformation deploy ... --disable-rollback      # keep failed resources for debugging (dev only)

aws cloudformation package \
  --template-file root.yaml \
  --s3-bucket "$CFN_BUCKET" --s3-prefix packaged \
  --output-template-file packaged.yaml                # upload local nested templates / Lambda code, rewrite paths to S3 URLs
```

| Capability | Required when the template... |
|---|---|
| `CAPABILITY_IAM` | Creates IAM resources without custom names |
| `CAPABILITY_NAMED_IAM` | Creates IAM resources with custom names (`RoleName`, `PolicyName` on managed policies) |
| `CAPABILITY_AUTO_EXPAND` | Contains macros or nested stacks that contain macros (SAM, `AWS::Include`, custom macros) |

---

## 5. Stacks (Low-Level)

```bash
aws cloudformation create-stack \
  --stack-name "$STACK" \
  --template-body file://template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=dev \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn "$CFN_ROLE" \
  --on-failure ROLLBACK \
  --enable-termination-protection \
  --tags Key=project,Value=capstone                   # create directly without a change set

aws cloudformation update-stack \
  --stack-name "$STACK" \
  --use-previous-template \
  --parameters ParameterKey=Environment,UsePreviousValue=true ParameterKey=DesiredCount,ParameterValue=3
                                                      # change only a parameter, keep the template

aws cloudformation delete-stack --stack-name "$STACK" --role-arn "$CFN_ROLE"   # delete stack and its resources
aws cloudformation delete-stack --stack-name "$STACK" --retain-resources LogicalIdA LogicalIdB
                                                      # only valid for a stack in DELETE_FAILED; keeps those resources
aws cloudformation delete-stack --stack-name "$STACK" --deletion-mode FORCE_DELETE_STACK
                                                      # DELETE_FAILED stacks: remove the stack, leave failing resources
aws cloudformation cancel-update-stack --stack-name "$STACK"                  # abort an UPDATE_IN_PROGRESS (triggers rollback)
aws cloudformation update-stack ... --rollback-configuration \
  'RollbackTriggers=[{Arn=arn:aws:cloudwatch:eu-west-1:123456789012:alarm:api-5xx,Type=AWS::CloudWatch::Alarm}],MonitoringTimeInMinutes=10'
                                                      # roll back automatically if the alarm fires during or after the update
```

---

## 6. Change Sets

```bash
aws cloudformation create-change-set \
  --stack-name "$STACK" \
  --change-set-name "pr-42-$(date +%s)" \
  --change-set-type UPDATE \
  --template-body file://template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=dev \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn "$CFN_ROLE" \
  --include-nested-stacks                             # CREATE | UPDATE | IMPORT; nested change sets for child stacks

aws cloudformation wait change-set-create-complete --stack-name "$STACK" --change-set-name "$CS"

aws cloudformation describe-change-set --stack-name "$STACK" --change-set-name "$CS" \
  --query "Changes[].ResourceChange.[Action,LogicalResourceId,ResourceType,Replacement]" --output table
                                                      # the review table: look at the Replacement column first

aws cloudformation describe-change-set --stack-name "$STACK" --change-set-name "$CS" \
  --include-property-values                           # before/after values per property

aws cloudformation list-change-sets --stack-name "$STACK"                         # pending change sets
aws cloudformation execute-change-set --stack-name "$STACK" --change-set-name "$CS"  # apply exactly what was reviewed
aws cloudformation delete-change-set --stack-name "$STACK" --change-set-name "$CS"   # discard
```

A change set that has no changes ends in `FAILED` with the reason `The submitted information didn't contain changes.` Treat that as success in CI.

---

## 7. Inspect Stacks

```bash
aws cloudformation describe-stacks --stack-name "$STACK" --query "Stacks[0].StackStatus" --output text
aws cloudformation describe-stacks --stack-name "$STACK" --query "Stacks[0].Outputs" --output table
aws cloudformation describe-stacks --stack-name "$STACK" \
  --query "Stacks[0].Outputs[?OutputKey=='ServiceUrl'].OutputValue" --output text   # one output, script-friendly
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE UPDATE_ROLLBACK_COMPLETE
aws cloudformation list-stack-resources --stack-name "$STACK" \
  --query "StackResourceSummaries[].[LogicalResourceId,PhysicalResourceId,ResourceStatus]" --output table
aws cloudformation describe-stack-resource --stack-name "$STACK" --logical-resource-id AppBucket
aws cloudformation get-template --stack-name "$STACK" --template-stage Original    # template as submitted
aws cloudformation get-template --stack-name "$STACK" --template-stage Processed   # after transforms/macros
aws cloudformation list-exports --query "Exports[].[Name,Value]" --output table    # all exports in the region
aws cloudformation list-imports --export-name network-dev-VpcId                    # which stacks consume an export
aws cloudformation estimate-template-cost --template-body file://template.yaml     # returns a Pricing Calculator URL
```

---

## 8. Events: The Debugging Command

```bash
aws cloudformation describe-stack-events --stack-name "$STACK" \
  --query "StackEvents[?contains(ResourceStatus,'FAILED')].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]" \
  --output table                                      # every failure with its reason, newest first

aws cloudformation describe-stack-events --stack-name "$STACK" --max-items 20 \
  --query "StackEvents[].[Timestamp,LogicalResourceId,ResourceStatus]" --output text  # recent timeline

rain logs "$STACK"                                    # same, formatted and filtered to failures
rain logs "$STACK" --chart > timeline.html            # Gantt chart of resource durations
```

The **first** `CREATE_FAILED` or `UPDATE_FAILED` event, the oldest one, is the root cause. Later failures usually say `Resource creation cancelled`.

---

## 9. Waiters

```bash
aws cloudformation wait stack-create-complete --stack-name "$STACK"
aws cloudformation wait stack-update-complete --stack-name "$STACK"
aws cloudformation wait stack-delete-complete --stack-name "$STACK"
aws cloudformation wait stack-import-complete --stack-name "$STACK"
aws cloudformation wait stack-rollback-complete --stack-name "$STACK"
aws cloudformation wait change-set-create-complete --stack-name "$STACK" --change-set-name "$CS"
aws cloudformation wait stack-exists --stack-name "$STACK"
```

Waiters poll every 30 seconds for up to 120 attempts (one hour) and exit non-zero on a failure state, which makes them usable in CI.

---

## 10. Rollback and Recovery

```bash
aws cloudformation continue-update-rollback --stack-name "$STACK"                 # resume a rollback in UPDATE_ROLLBACK_FAILED
aws cloudformation continue-update-rollback --stack-name "$STACK" \
  --resources-to-skip AppDatabase NestedStack.InnerResource                      # skip resources that cannot roll back (marks them complete)
aws cloudformation rollback-stack --stack-name "$STACK"                          # roll back a CREATE_FAILED / UPDATE_FAILED stack
aws cloudformation signal-resource --stack-name "$STACK" \
  --logical-resource-id WebAsg --unique-id i-0123456789abcdef0 --status SUCCESS  # manual CreationPolicy signal
```

After `--resources-to-skip`, the template and the real resource no longer agree. Run drift detection and reconcile before the next update.

---

## 11. Protection

```bash
aws cloudformation update-termination-protection --stack-name "$STACK" --enable-termination-protection
aws cloudformation update-termination-protection --stack-name "$STACK" --no-enable-termination-protection
aws cloudformation set-stack-policy --stack-name "$STACK" --stack-policy-body file://stack-policy.json
aws cloudformation get-stack-policy --stack-name "$STACK"
aws cloudformation update-stack ... --stack-policy-during-update-body file://allow-db-replace.json   # temporary override for one update
```

`stack-policy.json` that blocks replacing or deleting the database but allows everything else:

```json
{
  "Statement": [
    { "Effect": "Allow", "Action": "Update:*", "Principal": "*", "Resource": "*" },
    {
      "Effect": "Deny",
      "Action": ["Update:Replace", "Update:Delete"],
      "Principal": "*",
      "Resource": "LogicalResourceId/AppDatabase"
    }
  ]
}
```

---

## 12. Drift

```bash
DRIFT_ID=$(aws cloudformation detect-stack-drift --stack-name "$STACK" --query StackDriftDetectionId --output text)
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id "$DRIFT_ID" \
  --query "[DetectionStatus,StackDriftStatus,DriftedStackResourceCount]"         # poll until DETECTION_COMPLETE
aws cloudformation describe-stack-resource-drifts --stack-name "$STACK" \
  --stack-resource-drift-status-filters MODIFIED DELETED \
  --query "StackResourceDrifts[].[LogicalResourceId,StackResourceDriftStatus,PropertyDifferences[].PropertyPath]"
aws cloudformation detect-stack-resource-drift --stack-name "$STACK" --logical-resource-id AppBucket   # one resource, synchronous
aws cloudformation detect-stack-set-drift --stack-set-name org-baseline                               # all instances of a StackSet
```

---

## 13. Import, IaC Generator, Refactoring

```bash
# Import existing resources into a stack (template must add only these resources, each with DeletionPolicy)
aws cloudformation create-change-set --stack-name "$STACK" --change-set-name import-1 \
  --change-set-type IMPORT --template-body file://template-with-imports.yaml \
  --resources-to-import file://resources-to-import.json
aws cloudformation execute-change-set --stack-name "$STACK" --change-set-name import-1
aws cloudformation wait stack-import-complete --stack-name "$STACK"

# Adopt resources with custom names during a normal create/update
aws cloudformation create-change-set ... --import-existing-resources

# IaC generator: scan the account and write a template
SCAN_ID=$(aws cloudformation start-resource-scan --query ResourceScanId --output text)
aws cloudformation describe-resource-scan --resource-scan-id "$SCAN_ID" --query Status
aws cloudformation list-resource-scan-resources --resource-scan-id "$SCAN_ID" --resource-type-prefix AWS::S3
aws cloudformation create-generated-template --generated-template-name legacy-buckets --resources file://selected.json
aws cloudformation get-generated-template --generated-template-name legacy-buckets --format YAML --query TemplateBody --output text

# Stack refactoring: move resources between stacks / rename logical IDs without replacement
aws cloudformation create-stack-refactor --stack-definitions file://stack-definitions.json \
  --resource-mappings file://resource-mappings.json
aws cloudformation describe-stack-refactor --stack-refactor-id "$REFACTOR_ID"
aws cloudformation execute-stack-refactor --stack-refactor-id "$REFACTOR_ID"
```

`resources-to-import.json`:

```json
[
  {
    "ResourceType": "AWS::S3::Bucket",
    "LogicalResourceId": "LegacyBucket",
    "ResourceIdentifier": { "BucketName": "legacy-reports-123456789012" }
  }
]
```

Stack refactoring is a recent addition. Check `aws cloudformation create-stack-refactor help` for the exact JSON shapes your CLI version expects.

---

## 14. StackSets

```bash
aws cloudformation activate-organizations-access                                 # once, from the management account
aws cloudformation create-stack-set --stack-set-name org-baseline \
  --template-body file://baseline.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false \
  --capabilities CAPABILITY_NAMED_IAM
aws cloudformation create-stack-instances --stack-set-name org-baseline \
  --deployment-targets OrganizationalUnitIds=ou-abcd-11111111 \
  --regions eu-west-1 eu-central-1 \
  --operation-preferences MaxConcurrentPercentage=25,FailureTolerancePercentage=0,RegionConcurrencyType=PARALLEL
aws cloudformation update-stack-set --stack-set-name org-baseline --template-body file://baseline.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --operation-preferences MaxConcurrentPercentage=25,FailureTolerancePercentage=0
aws cloudformation list-stack-instances --stack-set-name org-baseline \
  --query "Summaries[].[Account,Region,Status,StackInstanceStatus.DetailedStatus]" --output table
aws cloudformation describe-stack-set-operation --stack-set-name org-baseline --operation-id "$OP_ID"
aws cloudformation list-stack-set-operation-results --stack-set-name org-baseline --operation-id "$OP_ID"   # per-instance failure reasons
aws cloudformation delete-stack-instances --stack-set-name org-baseline \
  --deployment-targets OrganizationalUnitIds=ou-abcd-11111111 --regions eu-west-1 --no-retain-stacks
aws cloudformation delete-stack-set --stack-set-name org-baseline                # only after all instances are gone
```

Add `--call-as DELEGATED_ADMIN` when running from a delegated administrator account.

---

## 15. Registry and Hooks

```bash
aws cloudformation list-types --visibility PUBLIC --type RESOURCE --filters Category=THIRD_PARTY --max-items 20
aws cloudformation describe-type --type RESOURCE --type-name AWS::S3::Bucket --query Schema --output text | jq '.properties | keys'
                                                      # the authoritative property schema for a type
aws cloudformation activate-type --type RESOURCE --type-name-alias MongoDB::Atlas::Project \
  --public-type-arn "$PUBLIC_TYPE_ARN" --execution-role-arn "$EXEC_ROLE_ARN"   # enable a third-party type
aws cloudformation set-type-configuration --type HOOK --type-name MyOrg::Guard::Hook \
  --configuration file://hook-config.json             # target resource types, failure mode
aws cloudformation list-types --visibility PRIVATE --type HOOK
```

---

## 16. Rain

```bash
rain deploy template.yaml "$STACK" --params Environment=dev,DesiredCount=2   # change set + live progress view
rain fmt --write template.yaml                       # canonical formatting (also converts JSON to YAML)
rain diff template.yaml "$STACK"                     # local template versus deployed template
rain ls                                              # stacks in the region with status
rain cat "$STACK"                                    # print the deployed template
rain build AWS::ECS::Service                         # skeleton with every property for a type
rain rm "$STACK"                                     # delete with confirmation
```

---

## 17. SAM

```bash
sam init --runtime python3.12 --name hello-api       # scaffold a serverless app
sam validate --lint                                   # SAM validation + cfn-lint
sam build                                             # build functions and dependencies into .aws-sam/
sam local invoke HelloFunction --event events/event.json   # run a function locally in Docker
sam deploy --guided                                   # first deploy; writes samconfig.toml
sam deploy                                            # subsequent deploys via change set
sam sync --watch --stack-name "$STACK"                # dev-only fast sync (skips change sets)
sam delete --stack-name "$STACK"
```

---

## 18. AWS CDK v2

```bash
cdk init app --language python                        # new app
cdk bootstrap aws://123456789012/eu-west-1            # one-off CDKToolkit stack per account/region
cdk ls                                                # stacks in the app
cdk synth                                             # write CloudFormation to cdk.out/
cdk diff                                              # compare synthesised template with deployed stack
cdk deploy --require-approval broadening              # prompt only when IAM/security-group permissions widen
cdk deploy --method=prepare-change-set                # create the change set without executing it
cdk destroy
cdk import                                            # adopt existing resources into a CDK stack
cdk migrate --stack-name legacy --from-stack --language python   # generate a CDK app from a deployed stack
cdk doctor                                            # environment diagnostics
```

---

## 19. Template Snippet Reference

```yaml
VpcId: !Ref Vpc                                          # resource identifier or parameter value
BucketArn: !GetAtt AppBucket.Arn                         # attribute
Name: !Sub "${AWS::StackName}-api"                       # interpolation
PolicyArn: !Sub "arn:${AWS::Partition}:iam::aws:policy/ReadOnlyAccess"
Subnets: !Split [",", !Ref PrivateSubnetIdsCsv]          # SSM comma string to list
Az0: !Select [0, !GetAZs ""]                             # first AZ in region
Cidrs: !Cidr [!GetAtt Vpc.CidrBlock, 6, 8]               # six /24s from the VPC CIDR
Size: !FindInMap [EnvSizing, !Ref Environment, Cpu]
Optional: !If [IsProd, "value", !Ref AWS::NoValue]       # omit property when false
Imported: !ImportValue network-dev-VpcId
UserData:
  Fn::Base64: !Sub |
    #!/bin/bash
    echo "${AWS::Region}" > /etc/region
Secret: "{{resolve:secretsmanager:capstone/dev/api:SecretString:token}}"
Param: "{{resolve:ssm:/capstone/dev/api/image_tag}}"
```

Next: [04-hands-on-labs.md](04-hands-on-labs.md).
