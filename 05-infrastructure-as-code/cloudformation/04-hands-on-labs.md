# AWS CloudFormation: Hands-On Labs

| | |
|---|---|
| **Cluster** | 05 Infrastructure as Code |
| **Role** | Bridge (anchor is [Terraform](../terraform/README.md)) |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md) complete, including the `cfn-lab-bootstrap` stack and the `CFN_BUCKET` / `CFN_ROLE` variables; Terraform Labs 2 and 7 for Labs 8-9 |
| **Target version** | AWS CLI v2, cfn-lint 1.x, cfn-guard 3.x, AWS CDK v2 (Lab 7b only) |
| **Capstone phase** | Phase 2 (Months 5-8) |

> Nine progressive labs: a first stack through a change set; parameters, mappings and conditions in a VPC; replacement and protection controls; failure and rollback states; exports versus SSM contracts; drift and import; a Lambda-backed custom resource (plus an optional CDK comparison); policy-as-code and an OIDC pipeline; and the capstone deliverable, the FastAPI incident API on ECS Fargate fed by Terraform's SSM parameters. Every lab has verification and cleanup steps.

**Cost and safety notes.** Use a sandbox account in `eu-west-1`. Labs 1-7 cost cents. Lab 2 creates a NAT gateway only if you set `CreateNatGateway=true` (about USD 0.05/hour plus data). Lab 9 runs an ALB and two Fargate tasks (about USD 0.05/hour) on top of the Terraform network layer; tear it down the same day. Every stack except the bootstrap is deployed with `--role-arn "$CFN_ROLE"`, so CloudFormation acts with the service role, not with your SSO session.

**Layout the labs build:**

```text
cfn-labs/
├── bootstrap/cfn-lab-bootstrap.yaml          # from 01-installation-and-setup.md
├── lab01-bucket.yaml                         # Labs 1, 3, 6
├── lab02-vpc.yaml                            # Labs 2, 5
├── lab04-failure.yaml                        # Lab 4
├── lab05-app-sg.yaml                         # Lab 5
├── lab07-custom-resource.yaml                # Lab 7
├── cdk-bucket/                               # Lab 7b (optional)
├── rules/secure-defaults.guard               # Lab 8
├── pipeline-roles.yaml                       # Lab 8
└── capstone/
    ├── incident-api.yaml                     # Lab 9
    └── params/dev.json                       # Lab 9
```

```bash
mkdir -p cfn-labs && cd cfn-labs
export AWS_REGION=eu-west-1
aws cloudformation describe-stacks --stack-name cfn-lab-bootstrap --query "Stacks[0].StackStatus" --output text
echo "$CFN_BUCKET $CFN_ROLE"     # both must be set; see 01-installation-and-setup.md section 7
```

---

## Lab 1: First Stack Through a Change Set

**Objective.** Deploy a secure S3 bucket by explicitly creating, reviewing and executing a change set: the CloudFormation version of `plan -out` then `apply`.

**Prerequisites.** Bootstrap stack deployed.

**Time.** 20 minutes.

### Steps

1. Create `lab01-bucket.yaml`:

   ```yaml
   AWSTemplateFormatVersion: "2010-09-09"
   Description: Lab 1 - secure application bucket

   Resources:
     AppBucket:
       Type: AWS::S3::Bucket
       Properties:
         BucketEncryption:
           ServerSideEncryptionConfiguration:
             - ServerSideEncryptionByDefault:
                 SSEAlgorithm: AES256
         PublicAccessBlockConfiguration:
           BlockPublicAcls: true
           BlockPublicPolicy: true
           IgnorePublicAcls: true
           RestrictPublicBuckets: true
         OwnershipControls:
           Rules:
             - ObjectOwnership: BucketOwnerEnforced
         VersioningConfiguration:
           Status: Enabled

   Outputs:
     BucketName:
       Value: !Ref AppBucket
     BucketArn:
       Value: !GetAtt AppBucket.Arn
   ```

2. Lint, then create a change set for a stack that does not exist yet.

   ```bash
   cfn-lint lab01-bucket.yaml
   aws cloudformation create-change-set \
     --stack-name cfn-lab-bucket \
     --change-set-name initial \
     --change-set-type CREATE \
     --template-body file://lab01-bucket.yaml \
     --role-arn "$CFN_ROLE" \
     --tags Key=project,Value=cfn-lab
   aws cloudformation wait change-set-create-complete --stack-name cfn-lab-bucket --change-set-name initial
   ```

3. Review it, and notice that the stack already exists in `REVIEW_IN_PROGRESS` with no resources.

   ```bash
   aws cloudformation describe-change-set --stack-name cfn-lab-bucket --change-set-name initial \
     --query "Changes[].ResourceChange.[Action,LogicalResourceId,ResourceType]" --output table
   aws cloudformation describe-stacks --stack-name cfn-lab-bucket --query "Stacks[0].StackStatus" --output text
   ```

   Expected output:

   ```text
   -------------------------------------------
   |           DescribeChangeSet             |
   +------+------------+---------------------+
   |  Add |  AppBucket |  AWS::S3::Bucket    |
   +------+------------+---------------------+
   REVIEW_IN_PROGRESS
   ```

4. Execute and watch the events.

   ```bash
   aws cloudformation execute-change-set --stack-name cfn-lab-bucket --change-set-name initial
   aws cloudformation wait stack-create-complete --stack-name cfn-lab-bucket
   aws cloudformation describe-stack-events --stack-name cfn-lab-bucket \
     --query "reverse(StackEvents)[].[LogicalResourceId,ResourceStatus]" --output text
   ```

   Expected output:

   ```text
   cfn-lab-bucket  REVIEW_IN_PROGRESS
   cfn-lab-bucket  CREATE_IN_PROGRESS
   AppBucket       CREATE_IN_PROGRESS
   AppBucket       CREATE_IN_PROGRESS
   AppBucket       CREATE_COMPLETE
   cfn-lab-bucket  CREATE_COMPLETE
   ```

   The second `AppBucket CREATE_IN_PROGRESS` event carries `Resource creation Initiated` and the physical ID.

### Verify

```bash
export LAB_BUCKET=$(aws cloudformation describe-stacks --stack-name cfn-lab-bucket \
  --query "Stacks[0].Outputs[?OutputKey=='BucketName'].OutputValue" --output text)
aws s3api get-bucket-versioning --bucket "$LAB_BUCKET"            # "Status": "Enabled"
aws s3api get-public-access-block --bucket "$LAB_BUCKET"          # all four true
aws cloudformation list-stack-resources --stack-name cfn-lab-bucket --output table
```

The bucket name is generated (`cfn-lab-bucket-appbucket-<random>`) because the template does not set `BucketName`. That keeps replacements possible (Lab 3).

### Cleanup

Keep the stack for Labs 3 and 6.

### What you learned

A change set for a new stack creates the stack shell in `REVIEW_IN_PROGRESS`. Executing it is the apply step. Events are the audit log, and omitting physical names is a deliberate choice.

---

## Lab 2: Parameters, Mappings and Conditions in a VPC

**Objective.** Build a two-AZ VPC whose NAT gateway is optional, using `Fn::Cidr`, `Fn::GetAZs`, a mapping and conditions, and export its IDs.

**Prerequisites.** Lab 1.

**Time.** 40 minutes.

### Steps

1. Create `lab02-vpc.yaml`:

   ```yaml
   AWSTemplateFormatVersion: "2010-09-09"
   Description: Lab 2 - two-AZ VPC with optional NAT

   Parameters:
     Environment:
       Type: String
       AllowedValues: [dev, prod]
       Default: dev
     VpcCidr:
       Type: String
       Default: 10.20.0.0/16
       AllowedPattern: "^10\\.[0-9]{1,3}\\.0\\.0/16$"
     CreateNatGateway:
       Type: String
       AllowedValues: ["true", "false"]
       Default: "false"

   Mappings:
     EnvSettings:
       dev:  { FlowLogRetention: 7 }
       prod: { FlowLogRetention: 90 }

   Conditions:
     WithNat: !Equals [!Ref CreateNatGateway, "true"]

   Resources:
     Vpc:
       Type: AWS::EC2::VPC
       Properties:
         CidrBlock: !Ref VpcCidr
         EnableDnsHostnames: true
         EnableDnsSupport: true
         Tags: [{ Key: Name, Value: !Sub "${AWS::StackName}" }]

     InternetGateway:
       Type: AWS::EC2::InternetGateway
     GatewayAttachment:
       Type: AWS::EC2::VPCGatewayAttachment
       Properties:
         VpcId: !Ref Vpc
         InternetGatewayId: !Ref InternetGateway

     PublicSubnetA:
       Type: AWS::EC2::Subnet
       Properties:
         VpcId: !Ref Vpc
         AvailabilityZone: !Select [0, !GetAZs ""]
         CidrBlock: !Select [0, !Cidr [!Ref VpcCidr, 4, 8]]
         Tags: [{ Key: Name, Value: public-a }]
     PublicSubnetB:
       Type: AWS::EC2::Subnet
       Properties:
         VpcId: !Ref Vpc
         AvailabilityZone: !Select [1, !GetAZs ""]
         CidrBlock: !Select [1, !Cidr [!Ref VpcCidr, 4, 8]]
         Tags: [{ Key: Name, Value: public-b }]
     PrivateSubnetA:
       Type: AWS::EC2::Subnet
       Properties:
         VpcId: !Ref Vpc
         AvailabilityZone: !Select [0, !GetAZs ""]
         CidrBlock: !Select [2, !Cidr [!Ref VpcCidr, 4, 8]]
         Tags: [{ Key: Name, Value: private-a }]
     PrivateSubnetB:
       Type: AWS::EC2::Subnet
       Properties:
         VpcId: !Ref Vpc
         AvailabilityZone: !Select [1, !GetAZs ""]
         CidrBlock: !Select [3, !Cidr [!Ref VpcCidr, 4, 8]]
         Tags: [{ Key: Name, Value: private-b }]

     PublicRouteTable:
       Type: AWS::EC2::RouteTable
       Properties: { VpcId: !Ref Vpc }
     PublicDefaultRoute:
       Type: AWS::EC2::Route
       DependsOn: GatewayAttachment
       Properties:
         RouteTableId: !Ref PublicRouteTable
         DestinationCidrBlock: 0.0.0.0/0
         GatewayId: !Ref InternetGateway
     PublicAssocA:
       Type: AWS::EC2::SubnetRouteTableAssociation
       Properties: { SubnetId: !Ref PublicSubnetA, RouteTableId: !Ref PublicRouteTable }
     PublicAssocB:
       Type: AWS::EC2::SubnetRouteTableAssociation
       Properties: { SubnetId: !Ref PublicSubnetB, RouteTableId: !Ref PublicRouteTable }

     PrivateRouteTable:
       Type: AWS::EC2::RouteTable
       Properties: { VpcId: !Ref Vpc }
     PrivateAssocA:
       Type: AWS::EC2::SubnetRouteTableAssociation
       Properties: { SubnetId: !Ref PrivateSubnetA, RouteTableId: !Ref PrivateRouteTable }
     PrivateAssocB:
       Type: AWS::EC2::SubnetRouteTableAssociation
       Properties: { SubnetId: !Ref PrivateSubnetB, RouteTableId: !Ref PrivateRouteTable }

     NatEip:
       Type: AWS::EC2::EIP
       Condition: WithNat
       DependsOn: GatewayAttachment
       Properties: { Domain: vpc }
     NatGateway:
       Type: AWS::EC2::NatGateway
       Condition: WithNat
       Properties:
         AllocationId: !GetAtt NatEip.AllocationId
         SubnetId: !Ref PublicSubnetA
     PrivateDefaultRoute:
       Type: AWS::EC2::Route
       Condition: WithNat
       Properties:
         RouteTableId: !Ref PrivateRouteTable
         DestinationCidrBlock: 0.0.0.0/0
         NatGatewayId: !Ref NatGateway

     S3GatewayEndpoint:
       Type: AWS::EC2::VPCEndpoint
       Properties:
         VpcId: !Ref Vpc
         ServiceName: !Sub "com.amazonaws.${AWS::Region}.s3"
         VpcEndpointType: Gateway
         RouteTableIds: [!Ref PrivateRouteTable]

     FlowLogGroup:
       Type: AWS::Logs::LogGroup
       Properties:
         RetentionInDays: !FindInMap [EnvSettings, !Ref Environment, FlowLogRetention]

   Outputs:
     VpcId:
       Value: !Ref Vpc
       Export: { Name: !Sub "${AWS::StackName}-VpcId" }
     PrivateSubnetIds:
       Value: !Join [",", [!Ref PrivateSubnetA, !Ref PrivateSubnetB]]
       Export: { Name: !Sub "${AWS::StackName}-PrivateSubnetIds" }
     NatGatewayId:
       Condition: WithNat
       Value: !Ref NatGateway
   ```

2. Lint and deploy without NAT.

   ```bash
   cfn-lint lab02-vpc.yaml
   aws cloudformation deploy --stack-name cfn-lab-vpc --template-file lab02-vpc.yaml \
     --role-arn "$CFN_ROLE" --tags project=cfn-lab
   ```

3. Preview turning NAT on, without executing it.

   ```bash
   aws cloudformation deploy --stack-name cfn-lab-vpc --template-file lab02-vpc.yaml \
     --parameter-overrides CreateNatGateway=true --role-arn "$CFN_ROLE" --no-execute-changeset
   ```

   The command prints a `describe-change-set` command. Run it. You should see `Add` for `NatEip`, `NatGateway` and `PrivateDefaultRoute`, and no other changes. Delete the change set with `aws cloudformation delete-change-set --stack-name cfn-lab-vpc --change-set-name <NAME>`.

### Verify

```bash
aws cloudformation describe-stacks --stack-name cfn-lab-vpc --query "Stacks[0].Outputs" --output table
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$(aws cloudformation describe-stacks --stack-name cfn-lab-vpc \
  --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue" --output text)" \
  --query "Subnets[].[Tags[?Key=='Name']|[0].Value,CidrBlock,AvailabilityZone]" --output table
aws cloudformation get-template --stack-name cfn-lab-vpc --template-stage Processed --query TemplateBody | head -c 300
```

Expected subnets: `10.20.0.0/24` and `10.20.1.0/24` public, `10.20.2.0/24` and `10.20.3.0/24` private, across two AZs.

### Cleanup

Keep `cfn-lab-vpc` for Lab 5.

### What you learned

`Fn::Cidr` and `Fn::GetAZs` replace Terraform's `cidrsubnet` and AZ data source. A condition removes resources and outputs entirely. `DependsOn` fixes the missing edge between the route and the gateway attachment. Toggling a condition shows up as clean adds and removes in a change set.

---

## Lab 3: Replacement, Retention and Protection

**Objective.** Learn to read the `Replacement` column, and layer `DeletionPolicy`, `UpdateReplacePolicy`, a stack policy and termination protection over a stateful resource.

**Prerequisites.** Lab 1 stack `cfn-lab-bucket`.

**Time.** 30 minutes.

### Steps

1. Add a tag to `AppBucket` (under `Properties`) and preview it:

   ```yaml
         Tags:
           - Key: data-classification
             Value: internal
   ```

   ```bash
   aws cloudformation create-change-set --stack-name cfn-lab-bucket --change-set-name add-tag \
     --template-body file://lab01-bucket.yaml --role-arn "$CFN_ROLE"
   aws cloudformation wait change-set-create-complete --stack-name cfn-lab-bucket --change-set-name add-tag
   aws cloudformation describe-change-set --stack-name cfn-lab-bucket --change-set-name add-tag \
     --query "Changes[].ResourceChange.[Action,LogicalResourceId,Replacement]" --output text
   ```

   Expected: `Modify  AppBucket  False`. Execute it with `execute-change-set` and wait for `stack-update-complete`.

2. Now add a physical name, which forces replacement:

   ```yaml
         BucketName: !Sub "cfn-lab-${AWS::AccountId}-${AWS::Region}-renamed"
   ```

   Create a change set named `rename` the same way and describe it with `--include-property-values`. Expected: `Modify  AppBucket  True`, and the details list `BucketName` with `RequiresRecreation: Always`. **Do not execute.** Executing would create a new empty bucket, repoint outputs, and delete the old bucket in cleanup. The delete would fail because the bucket is not empty, and the stack would end in `UPDATE_COMPLETE` with an orphaned old bucket.

3. Protect the bucket. Add these at the same level as `Type`:

   ```yaml
       DeletionPolicy: Retain
       UpdateReplacePolicy: Retain
   ```

   Remove the `BucketName` line again. In the same edit, add a second tag (`- Key: retention`, `Value: protected`) so the update also changes a property; an update that changes only resource attributes may not be detected as a change. Deploy:

   ```bash
   aws cloudformation delete-change-set --stack-name cfn-lab-bucket --change-set-name rename
   aws cloudformation deploy --stack-name cfn-lab-bucket --template-file lab01-bucket.yaml --role-arn "$CFN_ROLE"
   ```

4. Add a stack policy that denies replacement and deletion of `AppBucket`, and enable termination protection:

   ```bash
   cat > stack-policy.json <<'EOF'
   {
     "Statement": [
       { "Effect": "Allow", "Action": "Update:*", "Principal": "*", "Resource": "*" },
       { "Effect": "Deny", "Action": ["Update:Replace", "Update:Delete"], "Principal": "*",
         "Resource": "LogicalResourceId/AppBucket" }
     ]
   }
   EOF
   aws cloudformation set-stack-policy --stack-name cfn-lab-bucket --stack-policy-body file://stack-policy.json
   aws cloudformation update-termination-protection --stack-name cfn-lab-bucket --enable-termination-protection
   ```

5. Prove both controls. Put `BucketName` back, deploy, and expect a rolled-back update. Then try to delete the stack.

   ```bash
   aws cloudformation deploy --stack-name cfn-lab-bucket --template-file lab01-bucket.yaml --role-arn "$CFN_ROLE" || true
   aws cloudformation describe-stack-events --stack-name cfn-lab-bucket \
     --query "StackEvents[?ResourceStatus=='UPDATE_FAILED'].ResourceStatusReason | [0]" --output text
   aws cloudformation delete-stack --stack-name cfn-lab-bucket
   ```

   Expected output (wording may vary slightly):

   ```text
   Action denied by stack policy: Statement [#1] does not allow [Update:Replace] for resource [LogicalResourceId/AppBucket];

   An error occurred (ValidationError) when calling the DeleteStack operation: Stack [cfn-lab-bucket] cannot be deleted while TerminationProtection is enabled
   ```

   Remove the `BucketName` line again before continuing.

### Verify

```bash
aws cloudformation describe-stacks --stack-name cfn-lab-bucket \
  --query "Stacks[0].[StackStatus,EnableTerminationProtection]" --output text   # UPDATE_ROLLBACK_COMPLETE  True
aws cloudformation get-stack-policy --stack-name cfn-lab-bucket --query StackPolicyBody --output text
```

### Cleanup

Leave the protections on; Lab 6 uses this stack and the final teardown removes them.

### What you learned

Replacement is a property-level fact you read from the change set, not something you guess. `UpdateReplacePolicy` protects the old resource during a replacement, `DeletionPolicy` protects it when the stack or resource is deleted, a stack policy blocks the action itself, and termination protection blocks stack deletion. `UPDATE_ROLLBACK_COMPLETE` is a healthy state.

---

## Lab 4: Failure, Rollback and the States That Trap You

**Objective.** Cause a create failure and an update failure on purpose, and recover from `ROLLBACK_COMPLETE`, `CREATE_FAILED` and `UPDATE_ROLLBACK_COMPLETE`.

**Prerequisites.** Bootstrap.

**Time.** 30 minutes.

### Steps

1. Create a resource outside CloudFormation that will collide:

   ```bash
   aws ssm put-parameter --name /cfn-lab/collision --type String --value manual
   ```

2. Create `lab04-failure.yaml`:

   ```yaml
   AWSTemplateFormatVersion: "2010-09-09"
   Description: Lab 4 - deliberate failure
   Parameters:
     SecondName:
       Type: String
       Default: /cfn-lab/collision
   Resources:
     GoodParam:
       Type: AWS::SSM::Parameter
       Properties: { Name: /cfn-lab/good, Type: String, Value: "ok" }
     CollidingParam:
       Type: AWS::SSM::Parameter
       Properties: { Name: !Ref SecondName, Type: String, Value: "from-cfn" }
   ```

3. Deploy and watch the first create fail.

   ```bash
   aws cloudformation deploy --stack-name cfn-lab-failure --template-file lab04-failure.yaml --role-arn "$CFN_ROLE" || true
   aws cloudformation describe-stack-events --stack-name cfn-lab-failure \
     --query "StackEvents[?ResourceStatus=='CREATE_FAILED'].[LogicalResourceId,ResourceStatusReason]" --output text
   aws cloudformation describe-stacks --stack-name cfn-lab-failure --query "Stacks[0].StackStatus" --output text
   ```

   Expected output:

   ```text
   CollidingParam  Resource handler returned message: "Parameter /cfn-lab/collision already exists." ...
   ROLLBACK_COMPLETE
   ```

   `GoodParam` was created and then deleted by the rollback.

4. Try to fix it with an update. It fails, because a stack whose first create rolled back can only be deleted.

   ```bash
   aws cloudformation deploy --stack-name cfn-lab-failure --template-file lab04-failure.yaml \
     --parameter-overrides SecondName=/cfn-lab/second --role-arn "$CFN_ROLE"
   ```

   Expected: `... is in ROLLBACK_COMPLETE state and can not be updated.`

   ```bash
   aws cloudformation delete-stack --stack-name cfn-lab-failure
   aws cloudformation wait stack-delete-complete --stack-name cfn-lab-failure
   ```

5. Repeat the failing create with rollback disabled, then fix forward.

   ```bash
   aws cloudformation deploy --stack-name cfn-lab-failure --template-file lab04-failure.yaml \
     --role-arn "$CFN_ROLE" --disable-rollback || true
   aws cloudformation describe-stacks --stack-name cfn-lab-failure --query "Stacks[0].StackStatus" --output text   # CREATE_FAILED
   aws ssm get-parameter --name /cfn-lab/good --query Parameter.Value --output text                               # ok (kept)
   aws cloudformation deploy --stack-name cfn-lab-failure --template-file lab04-failure.yaml \
     --parameter-overrides SecondName=/cfn-lab/second --role-arn "$CFN_ROLE"
   ```

   The stack moves to `UPDATE_COMPLETE`; only the failed resource was retried.

6. Cause an update failure with rollback enabled.

   ```bash
   aws cloudformation deploy --stack-name cfn-lab-failure --template-file lab04-failure.yaml \
     --parameter-overrides SecondName=/cfn-lab/collision --role-arn "$CFN_ROLE" || true
   aws cloudformation describe-stacks --stack-name cfn-lab-failure --query "Stacks[0].StackStatus" --output text
   ```

   Expected: `UPDATE_ROLLBACK_COMPLETE`. `/cfn-lab/second` still exists, because the replacement was rolled back.

### Verify

```bash
aws cloudformation describe-stack-events --stack-name cfn-lab-failure \
  --query "StackEvents[?LogicalResourceId=='cfn-lab-failure'].ResourceStatus" --output text | tr '\t' '\n' | head -n 8
```

Read the stack-level statuses from newest to oldest and match them to the state diagram in [02-core-concepts.md](02-core-concepts.md#7-the-stack-lifecycle).

### Cleanup

```bash
aws cloudformation delete-stack --stack-name cfn-lab-failure
aws ssm delete-parameter --name /cfn-lab/collision
rm lab04-failure.yaml
```

### What you learned

The first `CREATE_FAILED` event is the root cause. `ROLLBACK_COMPLETE` means delete and recreate. `--disable-rollback` speeds up debugging by keeping what worked. A failed update returns the stack to its previous good state.

---

## Lab 5: Cross-Stack References: Exports Versus SSM Contracts

**Objective.** Consume the Lab 2 VPC through `Fn::ImportValue`, experience the export lock, then migrate to an SSM parameter contract without downtime.

**Prerequisites.** Lab 2 stack `cfn-lab-vpc`.

**Time.** 30 minutes.

### Steps

1. Create `lab05-app-sg.yaml`:

   ```yaml
   AWSTemplateFormatVersion: "2010-09-09"
   Description: Lab 5 - consumer of the VPC stack
   Resources:
     AppSecurityGroup:
       Type: AWS::EC2::SecurityGroup
       Properties:
         GroupDescription: App tier
         VpcId: !ImportValue cfn-lab-vpc-VpcId
         SecurityGroupEgress:
           - { IpProtocol: tcp, FromPort: 443, ToPort: 443, CidrIp: 0.0.0.0/0, Description: HTTPS out }
   ```

   ```bash
   aws cloudformation deploy --stack-name cfn-lab-app-sg --template-file lab05-app-sg.yaml --role-arn "$CFN_ROLE"
   aws cloudformation list-imports --export-name cfn-lab-vpc-VpcId     # cfn-lab-app-sg
   ```

2. In `lab02-vpc.yaml`, change the export name to `!Sub "${AWS::StackName}-Vpc"` and deploy.

   ```bash
   aws cloudformation deploy --stack-name cfn-lab-vpc --template-file lab02-vpc.yaml --role-arn "$CFN_ROLE" || true
   aws cloudformation describe-stack-events --stack-name cfn-lab-vpc \
     --query "StackEvents[?contains(ResourceStatus,'FAILED')].ResourceStatusReason | [0]" --output text
   ```

   Expected: `Export cfn-lab-vpc-VpcId cannot be updated as it is in use by cfn-lab-app-sg`. Revert the export name.

3. Migrate to SSM, producer first. Add to `lab02-vpc.yaml` `Resources`:

   ```yaml
     VpcIdParameter:
       Type: AWS::SSM::Parameter
       Properties:
         Name: /cfn-lab/network/vpc_id
         Type: String
         Value: !Ref Vpc
   ```

   Deploy `cfn-lab-vpc`. Then change the consumer to read the parameter:

   ```yaml
   Parameters:
     VpcId:
       Type: AWS::SSM::Parameter::Value<AWS::EC2::VPC::Id>
       Default: /cfn-lab/network/vpc_id
   ```

   and set `VpcId: !Ref VpcId` on the security group. Deploy `cfn-lab-app-sg` and confirm the change set reports **no replacement** (the value is identical).

4. Now the export is unused. Remove the `Export` from the `VpcId` output and deploy `cfn-lab-vpc`; it succeeds.

### Verify

```bash
aws cloudformation list-exports --query "Exports[?starts_with(Name,'cfn-lab-vpc')].Name" --output text   # only PrivateSubnetIds
aws cloudformation describe-stacks --stack-name cfn-lab-app-sg --query "Stacks[0].Parameters" --output table
```

The `ResolvedValue` column shows the VPC ID that CloudFormation read from Parameter Store.

### Cleanup

```bash
aws cloudformation delete-stack --stack-name cfn-lab-app-sg
aws cloudformation wait stack-delete-complete --stack-name cfn-lab-app-sg
```

Keep `cfn-lab-vpc` until the final teardown.

### What you learned

Exports guarantee consistency by blocking the producer. SSM contracts decouple producer and consumer and work across tools (this is how Lab 9 consumes Terraform's VPC). To migrate, add the new contract, switch consumers, then remove the export.

---

## Lab 6: Drift Detection and Resource Import

**Objective.** Detect an out-of-band change and remediate it, then bring a bucket created by hand under CloudFormation management.

**Prerequisites.** Lab 3 (bucket stack with protections).

**Time.** 30 minutes.

### Steps

1. Make an out-of-band change and detect it.

   ```bash
   aws s3api put-bucket-versioning --bucket "$LAB_BUCKET" --versioning-configuration Status=Suspended
   DRIFT_ID=$(aws cloudformation detect-stack-drift --stack-name cfn-lab-bucket --query StackDriftDetectionId --output text)
   aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id "$DRIFT_ID" \
     --query "[DetectionStatus,StackDriftStatus]" --output text
   aws cloudformation describe-stack-resource-drifts --stack-name cfn-lab-bucket \
     --stack-resource-drift-status-filters MODIFIED \
     --query "StackResourceDrifts[].PropertyDifferences[].[PropertyPath,ExpectedValue,ActualValue]" --output text
   ```

   Expected output:

   ```text
   DETECTION_COMPLETE  DRIFTED
   /VersioningConfiguration/Status  Enabled  Suspended
   ```

2. Try to remediate by redeploying the unchanged template:

   ```bash
   aws cloudformation deploy --stack-name cfn-lab-bucket --template-file lab01-bucket.yaml --role-arn "$CFN_ROLE"
   ```

   Expected: `No changes to deploy. Stack cfn-lab-bucket is up to date`. CloudFormation compares templates, not reality, so it does nothing. Remediate directly (`put-bucket-versioning ... Status=Enabled`) or change the property in the template, deploy, and change it back. Re-run detection until it reports `IN_SYNC`.

3. Create a bucket outside CloudFormation, then import it.

   ```bash
   export LEGACY_BUCKET="cfn-lab-legacy-$(aws sts get-caller-identity --query Account --output text)"
   aws s3api create-bucket --bucket "$LEGACY_BUCKET" --create-bucket-configuration LocationConstraint=eu-west-1
   ```

   Add to `lab01-bucket.yaml` `Resources` (and make no other change):

   ```yaml
     LegacyBucket:
       Type: AWS::S3::Bucket
       DeletionPolicy: Retain
       UpdateReplacePolicy: Retain
       Properties:
         BucketName: !Sub "cfn-lab-legacy-${AWS::AccountId}"
   ```

   ```bash
   cat > resources-to-import.json <<EOF
   [{"ResourceType":"AWS::S3::Bucket","LogicalResourceId":"LegacyBucket","ResourceIdentifier":{"BucketName":"$LEGACY_BUCKET"}}]
   EOF
   aws cloudformation create-change-set --stack-name cfn-lab-bucket --change-set-name import-legacy \
     --change-set-type IMPORT --template-body file://lab01-bucket.yaml \
     --resources-to-import file://resources-to-import.json --role-arn "$CFN_ROLE"
   aws cloudformation wait change-set-create-complete --stack-name cfn-lab-bucket --change-set-name import-legacy
   aws cloudformation execute-change-set --stack-name cfn-lab-bucket --change-set-name import-legacy
   aws cloudformation wait stack-import-complete --stack-name cfn-lab-bucket
   ```

4. Bring the imported bucket up to standard with a normal update: copy the encryption, public access block, ownership and versioning properties from `AppBucket` into `LegacyBucket` and deploy. The change set shows `Modify LegacyBucket Replacement False`.

### Verify

```bash
aws cloudformation describe-stack-resource --stack-name cfn-lab-bucket --logical-resource-id LegacyBucket \
  --query "StackResourceDetail.[PhysicalResourceId,ResourceStatus]" --output text
aws s3api get-public-access-block --bucket "$LEGACY_BUCKET"
```

### Cleanup

Keep both; the final teardown handles retained buckets.

### What you learned

Drift detection reports and does not fix, and only checks properties you declared. A redeploy of an unchanged template is a no-op. Import needs a `DeletionPolicy` and a template that only adds the imported resources; you harden the resource afterwards in a normal update.

---

## Lab 7: A Lambda-Backed Custom Resource

**Objective.** Write a custom resource that empties a versioned bucket on stack deletion, and see the request and response protocol in the logs.

**Prerequisites.** Bootstrap (the service role can create roles only under `/cfn-lab/`).

**Time.** 40 minutes.

### Steps

1. Create `lab07-custom-resource.yaml`:

   ```yaml
   AWSTemplateFormatVersion: "2010-09-09"
   Description: Lab 7 - bucket that empties itself on delete

   Resources:
     ScratchBucket:
       Type: AWS::S3::Bucket
       Properties:
         BucketEncryption:
           ServerSideEncryptionConfiguration:
             - ServerSideEncryptionByDefault: { SSEAlgorithm: AES256 }
         PublicAccessBlockConfiguration:
           BlockPublicAcls: true
           BlockPublicPolicy: true
           IgnorePublicAcls: true
           RestrictPublicBuckets: true
         VersioningConfiguration: { Status: Enabled }

     EmptierRole:
       Type: AWS::IAM::Role
       Properties:
         Path: /cfn-lab/
         AssumeRolePolicyDocument:
           Version: "2012-10-17"
           Statement:
             - Effect: Allow
               Principal: { Service: lambda.amazonaws.com }
               Action: sts:AssumeRole
         ManagedPolicyArns:
           - !Sub "arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
         Policies:
           - PolicyName: empty-one-bucket
             PolicyDocument:
               Version: "2012-10-17"
               Statement:
                 - Effect: Allow
                   Action: [s3:ListBucket, s3:ListBucketVersions]
                   Resource: !GetAtt ScratchBucket.Arn
                 - Effect: Allow
                   Action: [s3:DeleteObject, s3:DeleteObjectVersion]
                   Resource: !Sub "${ScratchBucket.Arn}/*"

     EmptierFunction:
       Type: AWS::Lambda::Function
       Properties:
         Runtime: python3.12
         Handler: index.handler
         Timeout: 120
         Role: !GetAtt EmptierRole.Arn
         Code:
           ZipFile: |
             import boto3
             import cfnresponse

             s3 = boto3.resource("s3")

             def handler(event, context):
                 print({k: v for k, v in event.items() if k != "ResponseURL"})
                 bucket = event["ResourceProperties"]["BucketName"]
                 physical_id = event.get("PhysicalResourceId", f"emptier-{bucket}")
                 try:
                     if event["RequestType"] == "Delete":
                         s3.Bucket(bucket).object_versions.delete()
                     cfnresponse.send(event, context, cfnresponse.SUCCESS, {}, physical_id)
                 except Exception as exc:
                     cfnresponse.send(event, context, cfnresponse.FAILED, {}, physical_id, reason=str(exc))

     BucketEmptier:
       Type: Custom::BucketEmptier
       Properties:
         ServiceToken: !GetAtt EmptierFunction.Arn
         ServiceTimeout: 300
         BucketName: !Ref ScratchBucket

   Outputs:
     BucketName:
       Value: !Ref ScratchBucket
   ```

   The `cfnresponse` module is only available automatically for inline `ZipFile` code. Packaged functions must vendor it or call the response URL themselves.

2. Deploy and put some objects in the bucket.

   ```bash
   cfn-lint lab07-custom-resource.yaml
   aws cloudformation deploy --stack-name cfn-lab-custom --template-file lab07-custom-resource.yaml \
     --capabilities CAPABILITY_IAM --role-arn "$CFN_ROLE"
   SCRATCH=$(aws cloudformation describe-stacks --stack-name cfn-lab-custom \
     --query "Stacks[0].Outputs[0].OutputValue" --output text)
   for i in 1 2 3; do echo "v$i" | aws s3 cp - "s3://$SCRATCH/file.txt"; done
   aws s3api list-object-versions --bucket "$SCRATCH" --query "length(Versions)"     # 3
   ```

3. Delete the stack. Because `BucketEmptier` depends on the bucket, it is deleted first and its `Delete` request empties the bucket.

   ```bash
   aws cloudformation delete-stack --stack-name cfn-lab-custom
   aws cloudformation wait stack-delete-complete --stack-name cfn-lab-custom && echo deleted
   ```

### Verify

Lambda log groups outlive their functions, so the request log is still there after the stack is gone:

```bash
LOG_GROUP=$(aws logs describe-log-groups --log-group-name-prefix /aws/lambda/cfn-lab-custom-EmptierFunction \
  --query "logGroups[0].logGroupName" --output text)
aws logs tail "$LOG_GROUP" --since 30m
```

You should see one `Create` and one `Delete` request.

Break it on purpose: deploy again with `raise RuntimeError("boom")` as the first line of the handler but **without** the `try` block. The stack sits in `CREATE_IN_PROGRESS` until `ServiceTimeout` expires. This is why every code path must send a response.

### Cleanup

Delete the stack if it still exists, and delete leftover `/aws/lambda/cfn-lab-custom-*` log groups.

### What you learned

Custom resources are a request and response protocol over a pre-signed URL. The dependency graph controls the order of delete requests. `ServiceTimeout` limits the damage from a handler that never answers.

### Lab 7b (optional): What CDK Generates

```bash
mkdir cdk-bucket && cd cdk-bucket
cdk init app --language python
source .venv/bin/activate && pip install -r requirements.txt
```

Replace the stack body in `cdk_bucket/cdk_bucket_stack.py`:

```python
from aws_cdk import Stack, RemovalPolicy, aws_s3 as s3
from constructs import Construct


class CdkBucketStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)
        s3.Bucket(
            self, "AppBucket",
            encryption=s3.BucketEncryption.S3_MANAGED,
            block_public_access=s3.BlockPublicAccess.BLOCK_ALL,
            enforce_ssl=True,
            versioned=True,
            removal_policy=RemovalPolicy.RETAIN,
        )
```

```bash
cdk synth > synthesized.yaml
grep -E "Type: AWS::|DeletionPolicy|aws:SecureTransport" synthesized.yaml
cd ..
```

Compare with `lab01-bucket.yaml`. The L2 construct added a bucket policy that denies non-TLS access, set `DeletionPolicy: Retain`, and generated a hashed logical ID such as `AppBucket` followed by eight characters. Do not deploy unless you also run `cdk bootstrap`.

---

## Lab 8: Policy as Code and an OIDC Pipeline

**Objective.** Enforce secure defaults with cfn-guard, and create GitHub Actions roles where pull requests can only create change sets and only the approved `dev` environment can execute them.

**Prerequisites.** Terraform Lab 7 (the GitHub OIDC provider exists in the account); a GitHub repository for the capstone.

**Time.** 45 minutes.

### Steps

1. Create `rules/secure-defaults.guard`:

   ```text
   let s3_buckets = Resources.*[ Type == 'AWS::S3::Bucket' ]
   rule s3_secure when %s3_buckets !empty {
       %s3_buckets.Properties.BucketEncryption exists
       %s3_buckets.Properties.PublicAccessBlockConfiguration {
           BlockPublicAcls == true
           BlockPublicPolicy == true
           IgnorePublicAcls == true
           RestrictPublicBuckets == true
       }
   }

   let log_groups = Resources.*[ Type == 'AWS::Logs::LogGroup' ]
   rule logs_have_retention when %log_groups !empty {
       %log_groups.Properties.RetentionInDays exists
   }

   let task_defs = Resources.*[ Type == 'AWS::ECS::TaskDefinition' ]
   rule containers_read_only when %task_defs !empty {
       %task_defs.Properties.ContainerDefinitions[*].ReadonlyRootFilesystem == true
   }
   ```

   ```bash
   for t in bootstrap/*.yaml lab0*.yaml; do
     cfn-guard validate --data "$t" --rules rules/ --show-summary fail
   done
   ```

   `lab01-bucket.yaml` fails `s3_secure` if you skipped step 4 of Lab 6 (the imported `LegacyBucket` has no encryption block). Fix the template, not the rule.

2. Create `pipeline-roles.yaml`. Deploy it with your SSO session (no `--role-arn`), because it creates roles outside the `/cfn-lab/` path.

   ```yaml
   AWSTemplateFormatVersion: "2010-09-09"
   Description: GitHub Actions roles for CloudFormation change sets
   Parameters:
     GitHubOrg: { Type: String }
     GitHubRepo: { Type: String }
     CfnServiceRoleArn: { Type: String }
   Resources:
     PlanRole:
       Type: AWS::IAM::Role
       Properties:
         RoleName: gha-cfn-plan
         MaxSessionDuration: 3600
         AssumeRolePolicyDocument:
           Version: "2012-10-17"
           Statement:
             - Effect: Allow
               Principal:
                 Federated: !Sub "arn:${AWS::Partition}:iam::${AWS::AccountId}:oidc-provider/token.actions.githubusercontent.com"
               Action: sts:AssumeRoleWithWebIdentity
               Condition:
                 StringEquals:
                   token.actions.githubusercontent.com:aud: sts.amazonaws.com
                 StringLike:
                   token.actions.githubusercontent.com:sub:
                     - !Sub "repo:${GitHubOrg}/${GitHubRepo}:pull_request"
                     - !Sub "repo:${GitHubOrg}/${GitHubRepo}:ref:refs/heads/main"
         Policies:
           - PolicyName: create-change-sets
             PolicyDocument:
               Version: "2012-10-17"
               Statement:
                 - Effect: Allow
                   Action:
                     - cloudformation:CreateChangeSet
                     - cloudformation:DescribeChangeSet
                     - cloudformation:DeleteChangeSet
                     - cloudformation:ListChangeSets
                     - cloudformation:DescribeStacks
                     - cloudformation:DescribeStackEvents
                     - cloudformation:GetTemplateSummary
                   Resource: !Sub "arn:${AWS::Partition}:cloudformation:${AWS::Region}:${AWS::AccountId}:stack/capstone-*/*"
                 - Effect: Allow
                   Action: cloudformation:ValidateTemplate
                   Resource: "*"
                 - Effect: Allow
                   Action: ssm:GetParameters
                   Resource: !Sub "arn:${AWS::Partition}:ssm:${AWS::Region}:${AWS::AccountId}:parameter/capstone/*"
                 - Effect: Allow
                   Action: iam:PassRole
                   Resource: !Ref CfnServiceRoleArn
                   Condition:
                     StringEquals: { iam:PassedToService: cloudformation.amazonaws.com }
     DeployRole:
       Type: AWS::IAM::Role
       Properties:
         RoleName: gha-cfn-deploy
         MaxSessionDuration: 3600
         AssumeRolePolicyDocument:
           Version: "2012-10-17"
           Statement:
             - Effect: Allow
               Principal:
                 Federated: !Sub "arn:${AWS::Partition}:iam::${AWS::AccountId}:oidc-provider/token.actions.githubusercontent.com"
               Action: sts:AssumeRoleWithWebIdentity
               Condition:
                 StringEquals:
                   token.actions.githubusercontent.com:aud: sts.amazonaws.com
                   token.actions.githubusercontent.com:sub: !Sub "repo:${GitHubOrg}/${GitHubRepo}:environment:dev"
         Policies:
           - PolicyName: execute-change-sets
             PolicyDocument:
               Version: "2012-10-17"
               Statement:
                 - Effect: Allow
                   Action:
                     - cloudformation:ExecuteChangeSet
                     - cloudformation:DescribeChangeSet
                     - cloudformation:DescribeStacks
                     - cloudformation:DescribeStackEvents
                   Resource: !Sub "arn:${AWS::Partition}:cloudformation:${AWS::Region}:${AWS::AccountId}:stack/capstone-*/*"
                 - Effect: Allow
                   Action: iam:PassRole
                   Resource: !Ref CfnServiceRoleArn
                   Condition:
                     StringEquals: { iam:PassedToService: cloudformation.amazonaws.com }
   ```

   ```bash
   aws cloudformation deploy --stack-name cfn-lab-pipeline-roles --template-file pipeline-roles.yaml \
     --capabilities CAPABILITY_NAMED_IAM \
     --parameter-overrides GitHubOrg=<ORG> GitHubRepo=<REPO> CfnServiceRoleArn="$CFN_ROLE"
   ```

   Neither role can create an S3 bucket or an ECS service. Only the CloudFormation service role can, and only through a stack.

3. Add `.github/workflows/cloudformation.yml` to the capstone repository:

   ```yaml
   name: cloudformation
   on:
     pull_request:
       paths: ["infra/cloudformation/**"]
     push:
       branches: [main]
       paths: ["infra/cloudformation/**"]

   permissions:
     id-token: write
     contents: read

   env:
     AWS_REGION: eu-west-1
     STACK: capstone-dev-incident-api
     TEMPLATE: infra/cloudformation/incident-api.yaml
     PARAMS: infra/cloudformation/params/dev.json
     CFN_ROLE: arn:aws:iam::123456789012:role/cfn-lab-service-role

   jobs:
     validate:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: actions/setup-python@v5
           with: { python-version: "3.12" }
         - run: pip install "cfn-lint>=1,<2"
         - run: cfn-lint "$TEMPLATE"
         - name: cfn-guard
           run: |
             curl --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/aws-cloudformation/cloudformation-guard/main/install-guard.sh | sh
             ~/.guard/bin/cfn-guard validate --data "$TEMPLATE" --rules infra/policies/guard/ --show-summary fail

     change-set:
       needs: validate
       runs-on: ubuntu-latest
       outputs:
         name: ${{ steps.cs.outputs.name }}
         empty: ${{ steps.cs.outputs.empty }}
       steps:
         - uses: actions/checkout@v4
         - uses: aws-actions/configure-aws-credentials@v4
           with:
             role-to-assume: arn:aws:iam::123456789012:role/gha-cfn-plan
             aws-region: ${{ env.AWS_REGION }}
         - id: cs
           run: |
             CS="gha-${GITHUB_SHA::7}-${GITHUB_RUN_ATTEMPT}"
             aws cloudformation create-change-set --stack-name "$STACK" --change-set-name "$CS" \
               --change-set-type UPDATE --template-body "file://$TEMPLATE" --parameters "file://$PARAMS" \
               --capabilities CAPABILITY_IAM --role-arn "$CFN_ROLE"
             if ! aws cloudformation wait change-set-create-complete --stack-name "$STACK" --change-set-name "$CS"; then
               REASON=$(aws cloudformation describe-change-set --stack-name "$STACK" --change-set-name "$CS" --query StatusReason --output text)
               if [[ "$REASON" == *"didn't contain changes"* ]]; then
                 echo "empty=true" >> "$GITHUB_OUTPUT"; echo "No changes." >> "$GITHUB_STEP_SUMMARY"; exit 0
               fi
               echo "::error::$REASON"; exit 1
             fi
             {
               echo "### Change set \`$CS\`"
               echo '| Action | Resource | Type | Replacement |'
               echo '|---|---|---|---|'
               aws cloudformation describe-change-set --stack-name "$STACK" --change-set-name "$CS" \
                 --query "Changes[].ResourceChange.[Action,LogicalResourceId,ResourceType,Replacement]" --output text \
                 | awk -F'\t' '{print "| "$1" | "$2" | "$3" | "$4" |"}'
             } >> "$GITHUB_STEP_SUMMARY"
             echo "name=$CS" >> "$GITHUB_OUTPUT"

     execute:
       if: github.event_name == 'push' && needs.change-set.outputs.empty != 'true'
       needs: change-set
       runs-on: ubuntu-latest
       environment: dev
       steps:
         - uses: aws-actions/configure-aws-credentials@v4
           with:
             role-to-assume: arn:aws:iam::123456789012:role/gha-cfn-deploy
             aws-region: ${{ env.AWS_REGION }}
         - run: |
             aws cloudformation execute-change-set --stack-name "$STACK" --change-set-name "${{ needs.change-set.outputs.name }}"
             aws cloudformation wait stack-update-complete --stack-name "$STACK"
   ```

   In GitHub, create the `dev` environment with required reviewers. Copy `rules/secure-defaults.guard` to `infra/policies/guard/`.

### Verify

- On a pull request, the job summary shows the change set table and `aws sts get-caller-identity` (if added) shows `assumed-role/gha-cfn-plan/...`.
- On merge, the `execute` job waits for approval, then runs as `gha-cfn-deploy`.
- CloudTrail shows `CreateChangeSet` by `gha-cfn-plan` and resource API calls by `cfn-lab-service-role`.

The workflow uses `--change-set-type UPDATE`, so the stack must exist first (Lab 9 step 3).

### Cleanup

Keep the roles while you work on the capstone. Delete `cfn-lab-pipeline-roles` in the final teardown.

### What you learned

Separating "who can propose" from "who can execute" from "who can create resources" gives you three narrow roles. The change set is the reviewed artefact. Policy runs before AWS sees the template.

---

## Lab 9: Capstone: The Incident API on ECS Fargate

**Objective.** Deploy the FastAPI incident API as an ECS Fargate service behind an ALB, reading the VPC and subnets from the SSM parameters written by Terraform's network layer.

**Prerequisites.** Terraform Lab 2 applied (`/capstone/dev/network/*` exists, one NAT gateway); the FastAPI image pushed to ECR in Phase 1; Lab 8 for the pipeline.

**Time.** 60 minutes.

### Steps

1. Confirm the contract exists.

   ```bash
   aws ssm get-parameters-by-path --path /capstone/dev/network --query "Parameters[].Name" --output text
   ```

   Expected: `.../private_subnet_ids .../public_subnet_ids .../vpc_cidr .../vpc_id`.

2. Create `capstone/incident-api.yaml`:

   ```yaml
   AWSTemplateFormatVersion: "2010-09-09"
   Description: Capstone incident API on ECS Fargate (CloudFormation bridge)

   Parameters:
     Environment:
       Type: String
       AllowedValues: [dev, staging, prod]
       Default: dev
     VpcId:
       Type: AWS::SSM::Parameter::Value<String>
       Default: /capstone/dev/network/vpc_id
     VpcCidr:
       Type: AWS::SSM::Parameter::Value<String>
       Default: /capstone/dev/network/vpc_cidr
     PrivateSubnetIds:
       Type: AWS::SSM::Parameter::Value<String>
       Default: /capstone/dev/network/private_subnet_ids
     PublicSubnetIds:
       Type: AWS::SSM::Parameter::Value<String>
       Default: /capstone/dev/network/public_subnet_ids
     ContainerImage:
       Type: String
       AllowedPattern: "^.+@sha256:[a-f0-9]{64}$"
       ConstraintDescription: pin the image by digest
     ContainerPort:
       Type: Number
       Default: 8000
     HealthCheckPath:
       Type: String
       Default: /healthz
     AllowedIngressCidr:
       Type: String
       AllowedPattern: "^([0-9]{1,3}\\.){3}[0-9]{1,3}/[0-9]{1,2}$"
     DesiredCount:
       Type: Number
       Default: 2
       MinValue: 1
       MaxValue: 10

   Conditions:
     IsProd: !Equals [!Ref Environment, prod]

   Resources:
     LogGroup:
       Type: AWS::Logs::LogGroup
       Properties:
         LogGroupName: !Sub "/capstone/${Environment}/incident-api"
         RetentionInDays: !If [IsProd, 90, 14]

     Cluster:
       Type: AWS::ECS::Cluster
       Properties:
         ClusterSettings: [{ Name: containerInsights, Value: enabled }]

     ExecutionRole:
       Type: AWS::IAM::Role
       Properties:
         Path: /cfn-lab/
         AssumeRolePolicyDocument:
           Version: "2012-10-17"
           Statement:
             - Effect: Allow
               Principal: { Service: ecs-tasks.amazonaws.com }
               Action: sts:AssumeRole
               Condition: { StringEquals: { aws:SourceAccount: !Ref AWS::AccountId } }
         ManagedPolicyArns:
           - !Sub "arn:${AWS::Partition}:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"

     TaskRole:
       Type: AWS::IAM::Role
       Properties:
         Path: /cfn-lab/
         AssumeRolePolicyDocument:
           Version: "2012-10-17"
           Statement:
             - Effect: Allow
               Principal: { Service: ecs-tasks.amazonaws.com }
               Action: sts:AssumeRole
               Condition: { StringEquals: { aws:SourceAccount: !Ref AWS::AccountId } }

     TaskDefinition:
       Type: AWS::ECS::TaskDefinition
       Properties:
         Family: !Sub "capstone-${Environment}-incident-api"
         RequiresCompatibilities: [FARGATE]
         NetworkMode: awsvpc
         Cpu: "256"
         Memory: "512"
         RuntimePlatform: { CpuArchitecture: X86_64, OperatingSystemFamily: LINUX }
         ExecutionRoleArn: !GetAtt ExecutionRole.Arn
         TaskRoleArn: !GetAtt TaskRole.Arn
         ContainerDefinitions:
           - Name: api
             Image: !Ref ContainerImage
             Essential: true
             ReadonlyRootFilesystem: true
             PortMappings: [{ ContainerPort: !Ref ContainerPort, Protocol: tcp }]
             Environment: [{ Name: APP_ENV, Value: !Ref Environment }]
             LogConfiguration:
               LogDriver: awslogs
               Options:
                 awslogs-group: !Ref LogGroup
                 awslogs-region: !Ref AWS::Region
                 awslogs-stream-prefix: api

     AlbSecurityGroup:
       Type: AWS::EC2::SecurityGroup
       Properties:
         GroupDescription: Incident API load balancer
         VpcId: !Ref VpcId
         SecurityGroupIngress:
           - { IpProtocol: tcp, FromPort: 80, ToPort: 80, CidrIp: !Ref AllowedIngressCidr }
         SecurityGroupEgress:
           - { IpProtocol: tcp, FromPort: !Ref ContainerPort, ToPort: !Ref ContainerPort, CidrIp: !Ref VpcCidr }

     ServiceSecurityGroup:
       Type: AWS::EC2::SecurityGroup
       Properties:
         GroupDescription: Incident API tasks
         VpcId: !Ref VpcId
         SecurityGroupIngress:
           - IpProtocol: tcp
             FromPort: !Ref ContainerPort
             ToPort: !Ref ContainerPort
             SourceSecurityGroupId: !Ref AlbSecurityGroup
         SecurityGroupEgress:
           - { IpProtocol: tcp, FromPort: 443, ToPort: 443, CidrIp: 0.0.0.0/0, Description: ECR, logs, AWS APIs }

     LoadBalancer:
       Type: AWS::ElasticLoadBalancingV2::LoadBalancer
       Properties:
         Scheme: internet-facing
         Subnets: !Split [",", !Ref PublicSubnetIds]
         SecurityGroups: [!Ref AlbSecurityGroup]
         LoadBalancerAttributes:
           - { Key: routing.http.drop_invalid_header_fields.enabled, Value: "true" }
           - { Key: deletion_protection.enabled, Value: !If [IsProd, "true", "false"] }

     TargetGroup:
       Type: AWS::ElasticLoadBalancingV2::TargetGroup
       Properties:
         TargetType: ip
         Protocol: HTTP
         Port: !Ref ContainerPort
         VpcId: !Ref VpcId
         HealthCheckPath: !Ref HealthCheckPath
         Matcher: { HttpCode: "200" }
         TargetGroupAttributes:
           - { Key: deregistration_delay.timeout_seconds, Value: "30" }

     Listener:
       Type: AWS::ElasticLoadBalancingV2::Listener
       Properties:
         LoadBalancerArn: !Ref LoadBalancer
         Port: 80
         Protocol: HTTP
         DefaultActions: [{ Type: forward, TargetGroupArn: !Ref TargetGroup }]

     Service:
       Type: AWS::ECS::Service
       DependsOn: Listener
       Properties:
         Cluster: !Ref Cluster
         LaunchType: FARGATE
         TaskDefinition: !Ref TaskDefinition
         DesiredCount: !Ref DesiredCount
         HealthCheckGracePeriodSeconds: 30
         PropagateTags: SERVICE
         DeploymentConfiguration:
           MinimumHealthyPercent: 100
           MaximumPercent: 200
           DeploymentCircuitBreaker: { Enable: true, Rollback: true }
         NetworkConfiguration:
           AwsvpcConfiguration:
             AssignPublicIp: DISABLED
             Subnets: !Split [",", !Ref PrivateSubnetIds]
             SecurityGroups: [!Ref ServiceSecurityGroup]
         LoadBalancers:
           - { ContainerName: api, ContainerPort: !Ref ContainerPort, TargetGroupArn: !Ref TargetGroup }

     ScalableTarget:
       Type: AWS::ApplicationAutoScaling::ScalableTarget
       Properties:
         ServiceNamespace: ecs
         ScalableDimension: ecs:service:DesiredCount
         ResourceId: !Sub "service/${Cluster}/${Service.Name}"
         MinCapacity: !Ref DesiredCount
         MaxCapacity: !If [IsProd, 10, 4]

     CpuScalingPolicy:
       Type: AWS::ApplicationAutoScaling::ScalingPolicy
       Properties:
         PolicyName: cpu-target-60
         PolicyType: TargetTrackingScaling
         ScalingTargetId: !Ref ScalableTarget
         TargetTrackingScalingPolicyConfiguration:
           TargetValue: 60
           PredefinedMetricSpecification: { PredefinedMetricType: ECSServiceAverageCPUUtilization }

   Outputs:
     ServiceUrl:
       Value: !Sub "http://${LoadBalancer.DNSName}"
     ClusterName:
       Value: !Ref Cluster
     ServiceName:
       Value: !GetAtt Service.Name
   ```

   > **Lab shortcut.** The listener is plain HTTP on port 80, restricted to `AllowedIngressCidr`. In production, add an ACM certificate and an HTTPS listener with `ELBSecurityPolicy-TLS13-1-2-2021-06`, redirect 80 to 443, and put AWS WAF in front.

3. Create `capstone/params/dev.json` and deploy the stack for the first time from your workstation.

   ```json
   [
     { "ParameterKey": "Environment", "ParameterValue": "dev" },
     { "ParameterKey": "ContainerImage", "ParameterValue": "123456789012.dkr.ecr.eu-west-1.amazonaws.com/incident-api-python@sha256:<DIGEST>" },
     { "ParameterKey": "AllowedIngressCidr", "ParameterValue": "<YOUR_PUBLIC_IP>/32" },
     { "ParameterKey": "DesiredCount", "ParameterValue": "2" }
   ]
   ```

   ```bash
   cfn-lint capstone/incident-api.yaml
   cfn-guard validate --data capstone/incident-api.yaml --rules rules/ --show-summary fail
   aws cloudformation create-change-set --stack-name capstone-dev-incident-api --change-set-name initial \
     --change-set-type CREATE --template-body file://capstone/incident-api.yaml \
     --parameters file://capstone/params/dev.json --capabilities CAPABILITY_IAM --role-arn "$CFN_ROLE" \
     --tags Key=project,Value=capstone Key=env,Value=dev
   aws cloudformation wait change-set-create-complete --stack-name capstone-dev-incident-api --change-set-name initial
   aws cloudformation execute-change-set --stack-name capstone-dev-incident-api --change-set-name initial
   aws cloudformation wait stack-create-complete --stack-name capstone-dev-incident-api
   ```

   Creation takes 5 to 8 minutes; the ECS service waits for tasks to pass the ALB health check. If the image digest is wrong, the deployment circuit breaker fails the service and the stack rolls back. Read the `Service` event and the ECS service events.

4. Move the files into the capstone repository at `infra/cloudformation/` and change `DesiredCount` to `3` in a pull request. The Lab 8 workflow posts the change set (`Modify Service False`, `Modify ScalableTarget False`); approve the `dev` environment after merge.

### Verify

```bash
URL=$(aws cloudformation describe-stacks --stack-name capstone-dev-incident-api \
  --query "Stacks[0].Outputs[?OutputKey=='ServiceUrl'].OutputValue" --output text)
curl --silent --fail "$URL/healthz"
aws ecs describe-services --cluster "$(aws cloudformation describe-stacks --stack-name capstone-dev-incident-api \
  --query "Stacks[0].Outputs[?OutputKey=='ClusterName'].OutputValue" --output text)" \
  --services "$(aws cloudformation describe-stacks --stack-name capstone-dev-incident-api \
  --query "Stacks[0].Outputs[?OutputKey=='ServiceName'].OutputValue" --output text)" \
  --query "services[0].[runningCount,deployments[0].rolloutState]" --output text     # 2  COMPLETED
aws cloudformation describe-stacks --stack-name capstone-dev-incident-api --query "Stacks[0].Parameters[?ParameterKey=='VpcId'].ResolvedValue" --output text
```

The resolved `VpcId` must equal `terraform output -raw vpc_id` in `live/dev/network`. That confirms the cross-tool contract works.

### Cleanup

```bash
aws cloudformation delete-stack --stack-name capstone-dev-incident-api --role-arn "$CFN_ROLE"
aws cloudformation wait stack-delete-complete --stack-name capstone-dev-incident-api
```

Tear down the Terraform network layer afterwards if you are not continuing with the Terraform labs.

### What you learned

CloudFormation and Terraform can share one estate through SSM parameters. A single service stack contains roles, network rules, load balancing, compute and autoscaling, and the pipeline from Lab 8 deploys it through reviewed change sets. This completes the "(bridge) CloudFormation template for the same service deployed via change set" item in [phase-checklists.md](../../capstone/phase-checklists.md).

---

## Final Teardown

```bash
aws cloudformation delete-stack --stack-name cfn-lab-pipeline-roles
aws cloudformation delete-stack --stack-name cfn-lab-vpc --role-arn "$CFN_ROLE"

aws cloudformation update-termination-protection --stack-name cfn-lab-bucket --no-enable-termination-protection
aws cloudformation delete-stack --stack-name cfn-lab-bucket --role-arn "$CFN_ROLE"
aws cloudformation wait stack-delete-complete --stack-name cfn-lab-bucket
# Both buckets had DeletionPolicy: Retain and still exist. The labs put no objects in them:
aws s3api delete-bucket --bucket "$LAB_BUCKET"
aws s3api delete-bucket --bucket "$LEGACY_BUCKET"

aws ssm delete-parameters --names /cfn-lab/good /cfn-lab/second /cfn-lab/network/vpc_id 2>/dev/null || true
```

Delete the bootstrap stack last. If you uploaded anything to the artefacts bucket (for example with `package`), empty it first, including old versions (S3 console **Empty**, or `aws s3api delete-objects` over `list-object-versions`). The bucket has `DeletionPolicy: Delete`, but CloudFormation cannot delete a bucket that still contains objects.
