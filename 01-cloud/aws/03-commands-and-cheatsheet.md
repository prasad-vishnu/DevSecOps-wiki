# AWS: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 01 Hyperscale Cloud |
| **Role** | Anchor |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md); `AWS_PROFILE` exported |
| **Target version** | AWS CLI v2.15+ |
| **Capstone phase** | Phase 2 (Months 5-8) |

> A task-organised reference for AWS CLI v2. Every command is real syntax, uses account `123456789012` and region `us-east-1`, and has a one-line explanation. Long-form flags are used; common short forms are noted in the CLI section. Commands that create billable resources are marked **($)**. Resource names used here match the labs so you can paste between the two files.

---

## 1. CLI Mechanics

### 1.1 Global options

| Option | Purpose |
|---|---|
| `--profile lab-admin` | Use a named profile from `~/.aws/config` |
| `--region us-east-1` | Override region for one call |
| `--output json\|yaml\|yaml-stream\|text\|table` | Output format; `text` for scripting, `table` for humans |
| `--query '<JMESPath>'` | Server-returned JSON filtered client-side |
| `--no-cli-pager` | Disable pager for one call (or set `cli_pager=` in config) |
| `--no-paginate` | Return only the first page |
| `--max-items 50 --starting-token <t>` | Client-side pagination control |
| `--page-size 100` | Server-side page size (useful for S3 `ls` on huge buckets) |
| `--cli-input-json file://req.json` / `--generate-cli-skeleton` | Drive a command from a JSON document; generate the skeleton first |
| `--dry-run` | EC2 only: check permissions without acting |
| `--debug` | Full HTTP request/response trace; the credential chain resolution is in the first 50 lines |
| `--endpoint-url http://localhost:4566` | Point at LocalStack or a VPC endpoint |
| `--cli-connect-timeout 5 --cli-read-timeout 30` | Fail fast in scripts |

### 1.2 Profiles, SSO, and identity

```bash
aws configure sso                                   # interactive: create sso-session + profile
aws sso login --sso-session capstone                # browser login; refreshes token cache
aws sso login --sso-session capstone --use-device-code   # headless host
aws sso logout                                      # clear all SSO tokens
aws configure list                                  # show which profile/region/creds source is active
aws configure list-profiles                         # all profile names
aws configure get region --profile lab-dev          # read one config value
aws configure set region us-east-2 --profile lab-dev   # write one config value
aws configure export-credentials --format env       # print AWS_* vars for the current profile
aws configure export-credentials --format process   # credential_process-compatible JSON
aws sts get-caller-identity                         # who am I (account, ARN, user id)
aws sts assume-role --role-arn arn:aws:iam::123456789012:role/breakglass \
  --role-session-name "$(whoami)-$(date +%s)" --duration-seconds 3600 \
  --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' --output text   # manual assume
aws sts get-session-token --serial-number arn:aws:iam::123456789012:mfa/you --token-code 123456   # MFA session (IAM users only; avoid)
aws sts decode-authorization-message --encoded-message "$MSG" --query DecodedMessage --output text | jq .   # decode EC2 encoded AccessDenied
```

A profile that assumes a role from the SSO profile (role chaining, 1-hour max):

```ini
[profile lab-breakglass]
source_profile = lab-admin
role_arn = arn:aws:iam::123456789012:role/breakglass
role_session_name = breakglass
mfa_serial = arn:aws:iam::123456789012:mfa/you
duration_seconds = 3600
```

### 1.3 JMESPath `--query` patterns

```bash
# Project fields into a new object with chosen names
aws ec2 describe-instances --query 'Reservations[].Instances[].{id:InstanceId,type:InstanceType,az:Placement.AvailabilityZone,state:State.Name}' --output table

# Filter with a predicate
aws ec2 describe-instances --query 'Reservations[].Instances[?State.Name==`running`].InstanceId' --output text

# Pull a tag value (Name) out of a Tags list
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId, Tags[?Key==`Name`].Value | [0]]' --output text

# Sort and take the newest AMI
aws ec2 describe-images --owners amazon --filters Name=name,Values='al2023-ami-2023*-arm64' \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text

# Flatten nested lists and count
aws iam list-roles --query 'length(Roles[?starts_with(RoleName, `AWSReservedSSO`)])'

# Contains / string functions
aws s3api list-buckets --query 'Buckets[?contains(Name, `capstone`)].Name' --output text

# Pipe expression and multiselect list
aws rds describe-db-instances --query 'DBInstances[].[DBInstanceIdentifier,Engine,DBInstanceClass,MultiAZ,Endpoint.Address]' --output table

# Join into a single string (for passing to another command)
aws ec2 describe-subnets --filters Name=tag:Tier,Values=private --query 'join(`,`, Subnets[].SubnetId)' --output text
```

Backticks denote literals inside JMESPath. In Bash, wrap the whole expression in single quotes so backticks are not interpreted as command substitution.

---

## 2. Organizations and Account

```bash
aws organizations describe-organization --query 'Organization.{id:Id,features:FeatureSet,mgmt:MasterAccountId}'
aws organizations list-roots --query 'Roots[0].Id' --output text                 # r-xxxx
aws organizations list-organizational-units-for-parent --parent-id r-xxxx       # OUs under root
aws organizations create-organizational-unit --parent-id r-xxxx --name Sandbox
aws organizations list-accounts --query 'Accounts[].{id:Id,name:Name,status:Status}' --output table
aws organizations create-account --email aws-lab@example.com --account-name lab --role-name OrganizationAccountAccessRole
aws organizations describe-create-account-status --create-account-request-id car-xxxx
aws organizations move-account --account-id 210987654321 --source-parent-id r-xxxx --destination-parent-id ou-xxxx-sandbox
aws organizations enable-policy-type --root-id r-xxxx --policy-type SERVICE_CONTROL_POLICY
aws organizations create-policy --name DenyOutsideRegions --type SERVICE_CONTROL_POLICY \
  --description "Lab region guard" --content file://scp-regions.json
aws organizations attach-policy --policy-id p-xxxx --target-id ou-xxxx-sandbox
aws organizations list-policies-for-target --target-id ou-xxxx-sandbox --filter SERVICE_CONTROL_POLICY
aws organizations enable-aws-service-access --service-principal guardduty.amazonaws.com   # trusted access
aws organizations register-delegated-administrator --account-id 210987654321 --service-principal securityhub.amazonaws.com
aws account get-contact-information                                              # primary contact
aws account put-alternate-contact --alternate-contact-type SECURITY --email-address security@example.com \
  --name "Security Team" --phone-number "+1-555-0100" --title "SecOps"
aws account list-regions --query 'Regions[?RegionOptStatus==`ENABLED` || RegionOptStatus==`ENABLED_BY_DEFAULT`].RegionName'
aws iam create-account-alias --account-alias capstone-lab
aws service-quotas get-service-quota --service-code eks --quota-code L-1194D53C   # EKS clusters per region
aws service-quotas request-service-quota-increase --service-code ec2 --quota-code L-34B43A08 --desired-value 64   # Spot vCPUs
```

---

## 3. IAM

### 3.1 Inspect

```bash
aws iam get-account-summary --query SummaryMap                         # counts, root MFA, root keys
aws iam list-users --query 'Users[].UserName'                          # should be empty
aws iam list-roles --query 'Roles[?!starts_with(Path, `/aws-service-role/`)].RoleName' --output text
aws iam get-role --role-name gha-capstone-deploy --query 'Role.AssumeRolePolicyDocument'   # trust policy (URL-decoded in v2)
aws iam list-attached-role-policies --role-name gha-capstone-deploy
aws iam list-role-policies --role-name gha-capstone-deploy             # inline policy names
aws iam get-role-policy --role-name gha-capstone-deploy --policy-name inline-ecr
aws iam get-policy --policy-arn arn:aws:iam::123456789012:policy/CapstoneDeploy --query 'Policy.DefaultVersionId' --output text
aws iam get-policy-version --policy-arn arn:aws:iam::123456789012:policy/CapstoneDeploy --version-id v3 --query 'PolicyVersion.Document'
aws iam list-policy-versions --policy-arn arn:aws:iam::123456789012:policy/CapstoneDeploy   # max 5; delete old before new
aws iam get-account-authorization-details --filter Role --query 'RoleDetailList[].RoleName'   # everything, big
aws iam generate-credential-report && sleep 5 && aws iam get-credential-report --query Content --output text | base64 -d
aws iam list-open-id-connect-providers
aws iam get-open-id-connect-provider --open-id-connect-provider-arn arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com
aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::123456789012:role/gha-capstone-deploy \
  --action-names s3:PutObject --resource-arns arn:aws:s3:::capstone-tfstate-123456789012/env/lab.tfstate \
  --query 'EvaluationResults[].{action:EvalActionName,decision:EvalDecision}'       # dry-run evaluation
```

### 3.2 Create roles and policies

```bash
# OIDC provider for GitHub Actions (thumbprint is ignored for GitHub since 2023 but the flag is required)
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list ffffffffffffffffffffffffffffffffffffffff

# Role with trust policy from file
aws iam create-role --role-name gha-capstone-deploy \
  --assume-role-policy-document file://trust-github.json \
  --max-session-duration 3600 \
  --tags Key=Project,Value=capstone Key=Environment,Value=lab

# Customer-managed policy and attach
aws iam create-policy --policy-name CapstoneDeploy --policy-document file://policy-deploy.json
aws iam attach-role-policy --role-name gha-capstone-deploy --policy-arn arn:aws:iam::123456789012:policy/CapstoneDeploy

# Update a managed policy (creates new default version; prune first if 5 exist)
aws iam create-policy-version --policy-arn arn:aws:iam::123456789012:policy/CapstoneDeploy \
  --policy-document file://policy-deploy.json --set-as-default
aws iam delete-policy-version --policy-arn arn:aws:iam::123456789012:policy/CapstoneDeploy --version-id v1

# Inline policy (lives and dies with the role)
aws iam put-role-policy --role-name gha-capstone-deploy --policy-name inline-ecr --policy-document file://ecr.json

# Permission boundary
aws iam put-role-permissions-boundary --role-name app-orders --permissions-boundary arn:aws:iam::123456789012:policy/CapstoneWorkloadBoundary

# Update trust policy in place
aws iam update-assume-role-policy --role-name gha-capstone-deploy --policy-document file://trust-github.json

# Service-linked role (most are auto-created; this one is needed before Spot)
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com

# Instance profile wrapping a role for EC2
aws iam create-instance-profile --instance-profile-name lab-ssm-instance
aws iam add-role-to-instance-profile --instance-profile-name lab-ssm-instance --role-name lab-ssm-instance

# Delete a role (detach everything first, in this order)
aws iam detach-role-policy --role-name gha-capstone-deploy --policy-arn arn:aws:iam::123456789012:policy/CapstoneDeploy
aws iam delete-role-policy --role-name gha-capstone-deploy --policy-name inline-ecr
aws iam delete-role --role-name gha-capstone-deploy
```

### 3.3 Access Analyzer

```bash
aws accessanalyzer create-analyzer --analyzer-name account-external --type ACCOUNT
aws accessanalyzer list-findings-v2 --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/account-external \
  --filter '{"status":{"eq":["ACTIVE"]}}' --query 'findings[].{res:resource,type:resourceType,public:isPublic}' --output table
aws accessanalyzer validate-policy --policy-type IDENTITY_POLICY --policy-document file://policy.json
aws accessanalyzer validate-policy --policy-type RESOURCE_POLICY --policy-document file://bucket-policy.json
aws accessanalyzer check-no-public-access --policy-document file://bucket-policy.json --resource-type AWS::S3::Bucket
aws accessanalyzer check-access-not-granted --policy-document file://policy.json --policy-type IDENTITY_POLICY \
  --access '[{"actions":["iam:CreateUser","iam:CreateAccessKey"]}]'
```

### 3.4 IAM Identity Center

```bash
aws sso-admin list-instances --query 'Instances[0].{arn:InstanceArn,store:IdentityStoreId}'
aws sso-admin list-permission-sets --instance-arn arn:aws:sso:::instance/ssoins-xxxx
aws sso-admin create-permission-set --instance-arn arn:aws:sso:::instance/ssoins-xxxx --name CapstoneDeveloper --session-duration PT8H
aws sso-admin attach-managed-policy-to-permission-set --instance-arn arn:aws:sso:::instance/ssoins-xxxx \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-xxxx/ps-xxxx --managed-policy-arn arn:aws:iam::aws:policy/PowerUserAccess
aws sso-admin put-permissions-boundary-to-permission-set --instance-arn arn:aws:sso:::instance/ssoins-xxxx \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-xxxx/ps-xxxx \
  --permissions-boundary '{"ManagedPolicyArn":"arn:aws:iam::aws:policy/PowerUserAccess"}'
aws identitystore list-users --identity-store-id d-xxxx --query 'Users[].UserName'
aws identitystore list-groups --identity-store-id d-xxxx --query 'Groups[].{id:GroupId,name:DisplayName}'
aws sso-admin create-account-assignment --instance-arn arn:aws:sso:::instance/ssoins-xxxx --target-id 123456789012 --target-type AWS_ACCOUNT \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-xxxx/ps-xxxx --principal-type GROUP --principal-id <GROUP_ID>
aws sso-admin provision-permission-set --instance-arn arn:aws:sso:::instance/ssoins-xxxx \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-xxxx/ps-xxxx --target-type ALL_PROVISIONED_ACCOUNTS
```

---

## 4. VPC and Networking

### 4.1 Build

```bash
# VPC with DNS enabled
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=capstone-lab},{Key=Project,Value=capstone}]' \
  --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames '{"Value":true}'
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-support '{"Value":true}'

# Subnets (one per tier per AZ)
aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.0.0.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-a},{Key=Tier,Value=public},{Key=kubernetes.io/role/elb,Value=1}]'
aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.0.10.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-a},{Key=Tier,Value=private},{Key=kubernetes.io/role/internal-elb,Value=1}]'
aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.0.20.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=isolated-a},{Key=Tier,Value=isolated}]'
aws ec2 modify-subnet-attribute --subnet-id subnet-pub-a --map-public-ip-on-launch     # only on public subnets

# Internet gateway
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"

# NAT gateway ($) with an Elastic IP
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
NAT_ID=$(aws ec2 create-nat-gateway --subnet-id subnet-pub-a --allocation-id "$EIP_ALLOC" \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-a}]' --query 'NatGateway.NatGatewayId' --output text)
aws ec2 wait nat-gateway-available --nat-gateway-ids "$NAT_ID"

# Route tables
RT_PUB=$(aws ec2 create-route-table --vpc-id "$VPC_ID" --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-public}]' --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id "$RT_PUB" --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID"
aws ec2 associate-route-table --route-table-id "$RT_PUB" --subnet-id subnet-pub-a
RT_PRIV_A=$(aws ec2 create-route-table --vpc-id "$VPC_ID" --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id "$RT_PRIV_A" --destination-cidr-block 0.0.0.0/0 --nat-gateway-id "$NAT_ID"
aws ec2 associate-route-table --route-table-id "$RT_PRIV_A" --subnet-id subnet-priv-a

# Gateway endpoint for S3 (free) attached to private route tables
aws ec2 create-vpc-endpoint --vpc-id "$VPC_ID" --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.us-east-1.s3 --route-table-ids "$RT_PRIV_A" \
  --policy-document '{"Statement":[{"Effect":"Allow","Principal":"*","Action":"s3:*","Resource":"*","Condition":{"StringEquals":{"aws:ResourceAccount":"123456789012"}}}]}'

# Interface endpoints ($) for ECR, STS, Logs, Secrets Manager
SG_VPCE=$(aws ec2 create-security-group --group-name sg-vpce --description "443 from VPC" --vpc-id "$VPC_ID" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id "$SG_VPCE" --protocol tcp --port 443 --cidr 10.0.0.0/16
for SVC in ecr.api ecr.dkr sts logs secretsmanager; do
  aws ec2 create-vpc-endpoint --vpc-id "$VPC_ID" --vpc-endpoint-type Interface \
    --service-name "com.amazonaws.us-east-1.${SVC}" --subnet-ids subnet-priv-a subnet-priv-b \
    --security-group-ids "$SG_VPCE" --private-dns-enabled
done

# Flow logs to CloudWatch Logs (needs a role trusted by vpc-flow-logs.amazonaws.com)
aws logs create-log-group --log-group-name /capstone/vpc/flow-logs
aws logs put-retention-policy --log-group-name /capstone/vpc/flow-logs --retention-in-days 14
aws ec2 create-flow-logs --resource-type VPC --resource-ids "$VPC_ID" --traffic-type ALL \
  --log-destination-type cloud-watch-logs --log-group-name /capstone/vpc/flow-logs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/vpc-flow-logs --max-aggregation-interval 60
```

### 4.2 Security groups

```bash
aws ec2 create-security-group --group-name sg-app --description "app tier" --vpc-id "$VPC_ID"
aws ec2 authorize-security-group-ingress --group-id sg-app --protocol tcp --port 8080 --source-group sg-alb   # SG chaining
aws ec2 authorize-security-group-ingress --group-id sg-db --ip-permissions \
  'IpProtocol=tcp,FromPort=5432,ToPort=5432,UserIdGroupPairs=[{GroupId=sg-app,Description="from app"}]'
aws ec2 revoke-security-group-ingress --group-id sg-app --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 describe-security-group-rules --filters Name=group-id,Values=sg-app \
  --query 'SecurityGroupRules[].{dir:IsEgress,proto:IpProtocol,from:FromPort,to:ToPort,cidr:CidrIpv4,sg:ReferencedGroupInfo.GroupId}' --output table
aws ec2 describe-security-groups --filters Name=ip-permission.cidr,Values=0.0.0.0/0 Name=ip-permission.from-port,Values=22 \
  --query 'SecurityGroups[].GroupId'                                                          # find SSH-open-to-world
aws ec2 describe-network-interfaces --filters Name=group-id,Values=sg-app --query 'NetworkInterfaces[].{eni:NetworkInterfaceId,desc:Description}'   # what uses this SG (blocks deletion)
```

### 4.3 Inspect and debug

```bash
aws ec2 describe-vpcs --query 'Vpcs[].{id:VpcId,cidr:CidrBlock,default:IsDefault,name:Tags[?Key==`Name`].Value|[0]}' --output table
aws ec2 describe-subnets --filters Name=vpc-id,Values="$VPC_ID" \
  --query 'Subnets[].{id:SubnetId,az:AvailabilityZone,cidr:CidrBlock,free:AvailableIpAddressCount,tier:Tags[?Key==`Tier`].Value|[0]}' --output table
aws ec2 describe-route-tables --filters Name=vpc-id,Values="$VPC_ID" \
  --query 'RouteTables[].{id:RouteTableId,subnets:Associations[].SubnetId,routes:Routes[].[DestinationCidrBlock,GatewayId,NatGatewayId,VpcPeeringConnectionId,TransitGatewayId]}'
aws ec2 describe-nat-gateways --filter Name=vpc-id,Values="$VPC_ID" --query 'NatGateways[].{id:NatGatewayId,state:State,subnet:SubnetId}'
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values="$VPC_ID" --query 'VpcEndpoints[].{id:VpcEndpointId,svc:ServiceName,type:VpcEndpointType,state:State}' --output table
aws ec2 describe-prefix-lists --filters Name=prefix-list-name,Values=com.amazonaws.us-east-1.s3 --query 'PrefixLists[0].PrefixListId'
aws ec2 describe-network-acls --filters Name=vpc-id,Values="$VPC_ID" --query 'NetworkAcls[].{id:NetworkAclId,entries:Entries[?Egress==`false`].[RuleNumber,Protocol,RuleAction,CidrBlock,PortRange.From,PortRange.To]}'
# Reachability Analyzer: does the path exist from ENI A to RDS ENI B on 5432?
aws ec2 create-network-insights-path --source eni-aaaa --destination eni-bbbb --protocol tcp --destination-port 5432 --query 'NetworkInsightsPath.NetworkInsightsPathId' --output text
aws ec2 start-network-insights-analysis --network-insights-path-id nip-xxxx --query 'NetworkInsightsAnalysis.NetworkInsightsAnalysisId' --output text
aws ec2 describe-network-insights-analyses --network-insights-analysis-ids nia-xxxx --query 'NetworkInsightsAnalyses[0].{reachable:NetworkPathFound,blocker:Explanations[0]}'
# Flow logs: rejected traffic to port 5432 in the last hour
aws logs start-query --log-group-name /capstone/vpc/flow-logs --start-time $(date -d '-1 hour' +%s) --end-time $(date +%s) \
  --query-string 'fields @timestamp, srcAddr, dstAddr, dstPort, action | filter action = "REJECT" and dstPort = 5432 | sort @timestamp desc | limit 50'
aws logs get-query-results --query-id <QUERY_ID>
```

### 4.4 Transit Gateway and peering

```bash
aws ec2 create-transit-gateway --description capstone-hub --options AmazonSideAsn=64512,DefaultRouteTableAssociation=disable,DefaultRouteTablePropagation=disable   # ($)
aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id tgw-xxxx --vpc-id "$VPC_ID" --subnet-ids subnet-priv-a subnet-priv-b
aws ec2 create-transit-gateway-route-table --transit-gateway-id tgw-xxxx
aws ec2 associate-transit-gateway-route-table --transit-gateway-route-table-id tgw-rtb-xxxx --transit-gateway-attachment-id tgw-attach-xxxx
aws ec2 create-transit-gateway-route --transit-gateway-route-table-id tgw-rtb-xxxx --destination-cidr-block 10.1.0.0/16 --transit-gateway-attachment-id tgw-attach-yyyy
aws ec2 create-route --route-table-id "$RT_PRIV_A" --destination-cidr-block 10.1.0.0/16 --transit-gateway-id tgw-xxxx
aws ec2 create-vpc-peering-connection --vpc-id "$VPC_ID" --peer-vpc-id vpc-peer --peer-owner-id 210987654321 --peer-region us-east-2
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-xxxx --region us-east-2
```

---

## 5. EC2 and Auto Scaling

```bash
# Latest AL2023 ARM64 AMI via public SSM parameter (no describe-images sorting needed)
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 --query 'Parameter.Value' --output text)

# Launch a Graviton instance in a private subnet with SSM access and IMDSv2 ($)
aws ec2 run-instances --image-id "$AMI" --instance-type t4g.micro --subnet-id subnet-priv-a \
  --security-group-ids sg-app --iam-instance-profile Name=lab-ssm-instance \
  --metadata-options HttpTokens=required,HttpEndpoint=enabled,HttpPutResponseHopLimit=2 \
  --block-device-mappings 'DeviceName=/dev/xvda,Ebs={VolumeSize=8,VolumeType=gp3,Encrypted=true,DeleteOnTermination=true}' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-probe},{Key=Project,Value=capstone},{Key=TTL,Value=24h}]' \
  --user-data file://cloud-init.sh --query 'Instances[0].InstanceId' --output text

aws ec2 wait instance-running --instance-ids i-xxxx
aws ec2 describe-instances --instance-ids i-xxxx --query 'Reservations[0].Instances[0].{state:State.Name,ip:PrivateIpAddress,az:Placement.AvailabilityZone}'
aws ec2 describe-instance-status --instance-ids i-xxxx --query 'InstanceStatuses[0].{sys:SystemStatus.Status,inst:InstanceStatus.Status}'
aws ec2 get-console-output --instance-id i-xxxx --latest --query Output --output text | tail -50   # boot log
aws ssm start-session --target i-xxxx                                                               # shell, no SSH
aws ssm start-session --target i-xxxx --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["capstone-db.cxxxx.us-east-1.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["15432"]}'   # tunnel to RDS
aws ssm send-command --instance-ids i-xxxx --document-name AWS-RunShellScript --parameters 'commands=["uname -m","curl -s -m 3 https://sts.us-east-1.amazonaws.com || echo blocked"]' --query 'Command.CommandId' --output text
aws ssm get-command-invocation --command-id <CMD_ID> --instance-id i-xxxx --query 'StandardOutputContent' --output text
aws ec2 stop-instances --instance-ids i-xxxx && aws ec2 wait instance-stopped --instance-ids i-xxxx
aws ec2 modify-instance-attribute --instance-id i-xxxx --instance-type '{"Value":"t4g.small"}'     # resize while stopped
aws ec2 terminate-instances --instance-ids i-xxxx

# Spot price history for right-sizing a node group
aws ec2 describe-spot-price-history --instance-types t4g.medium m7g.large --product-descriptions "Linux/UNIX" \
  --start-time "$(date -u -d '-1 hour' +%FT%TZ)" --query 'SpotPriceHistory[].{type:InstanceType,az:AvailabilityZone,price:SpotPrice}' --output table

# Instance type discovery
aws ec2 describe-instance-types --filters Name=processor-info.supported-architecture,Values=arm64 Name=vcpu-info.default-vcpus,Values=2 \
  --query 'InstanceTypes[].{type:InstanceType,mem:MemoryInfo.SizeInMiB,net:NetworkInfo.NetworkPerformance}' --output table

# EBS
aws ec2 describe-volumes --filters Name=status,Values=available --query 'Volumes[].{id:VolumeId,size:Size,created:CreateTime}' --output table   # orphans
aws ec2 create-snapshot --volume-id vol-xxxx --description "pre-upgrade" --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Project,Value=capstone}]'
aws ec2 modify-volume --volume-id vol-xxxx --size 20 --iops 4000 --throughput 250    # online gp3 tuning

# Launch template + ASG ($)
aws ec2 create-launch-template --launch-template-name lab-web --version-description v1 --launch-template-data "$(cat <<EOF
{"ImageId":"$AMI","InstanceType":"t4g.micro","IamInstanceProfile":{"Name":"lab-ssm-instance"},
 "SecurityGroupIds":["sg-app"],"MetadataOptions":{"HttpTokens":"required","HttpPutResponseHopLimit":2},
 "BlockDeviceMappings":[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":8,"VolumeType":"gp3","Encrypted":true}}],
 "TagSpecifications":[{"ResourceType":"instance","Tags":[{"Key":"Project","Value":"capstone"}]}],
 "UserData":"$(base64 -w0 cloud-init.sh)"}
EOF
)"
aws autoscaling create-auto-scaling-group --auto-scaling-group-name lab-web \
  --launch-template LaunchTemplateName=lab-web,Version='$Latest' \
  --min-size 2 --max-size 4 --desired-capacity 2 --vpc-zone-identifier "subnet-priv-a,subnet-priv-b" \
  --target-group-arns arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/lab-web/abc123 \
  --health-check-type ELB --health-check-grace-period 120 --default-instance-warmup 60 \
  --tags Key=Project,Value=capstone,PropagateAtLaunch=true
aws autoscaling put-scaling-policy --auto-scaling-group-name lab-web --policy-name cpu50 --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},"TargetValue":50.0}'
aws autoscaling start-instance-refresh --auto-scaling-group-name lab-web --preferences '{"MinHealthyPercentage":90,"InstanceWarmup":60}'   # rolling AMI update
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names lab-web --query 'AutoScalingGroups[0].Instances[].{id:InstanceId,az:AvailabilityZone,health:HealthStatus,lc:LifecycleState}' --output table
aws autoscaling describe-scaling-activities --auto-scaling-group-name lab-web --max-items 5 --query 'Activities[].{cause:Cause,status:StatusCode}'
aws autoscaling update-auto-scaling-group --auto-scaling-group-name lab-web --min-size 0 --desired-capacity 0   # scale to zero before delete
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name lab-web --force-delete
```

---

## 6. Load Balancing and Route 53

```bash
# ALB ($) in public subnets
ALB_ARN=$(aws elbv2 create-load-balancer --name lab-alb --type application --scheme internet-facing \
  --subnets subnet-pub-a subnet-pub-b --security-groups sg-alb --query 'LoadBalancers[0].LoadBalancerArn' --output text)
TG_ARN=$(aws elbv2 create-target-group --name lab-web --protocol HTTP --port 8080 --vpc-id "$VPC_ID" --target-type ip \
  --health-check-path /healthz --health-check-interval-seconds 15 --healthy-threshold-count 2 --unhealthy-threshold-count 2 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 create-listener --load-balancer-arn "$ALB_ARN" --protocol HTTPS --port 443 \
  --certificates CertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/xxxx \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 --default-actions Type=forward,TargetGroupArn="$TG_ARN"
aws elbv2 create-listener --load-balancer-arn "$ALB_ARN" --protocol HTTP --port 80 \
  --default-actions 'Type=redirect,RedirectConfig={Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'
aws elbv2 modify-load-balancer-attributes --load-balancer-arn "$ALB_ARN" --attributes \
  Key=access_logs.s3.enabled,Value=true Key=access_logs.s3.bucket,Value=capstone-alb-logs-123456789012 Key=routing.http.drop_invalid_header_fields.enabled,Value=true
aws elbv2 register-targets --target-group-arn "$TG_ARN" --targets Id=10.0.10.15,Port=8080
aws elbv2 describe-target-health --target-group-arn "$TG_ARN" --query 'TargetHealthDescriptions[].{t:Target.Id,s:TargetHealth.State,r:TargetHealth.Reason}' --output table
aws elbv2 describe-load-balancers --names lab-alb --query 'LoadBalancers[0].DNSName' --output text
aws elbv2 delete-load-balancer --load-balancer-arn "$ALB_ARN" && aws elbv2 delete-target-group --target-group-arn "$TG_ARN"

# ACM certificate with DNS validation
aws acm request-certificate --domain-name app.example.com --validation-method DNS --query CertificateArn --output text
aws acm describe-certificate --certificate-arn arn:aws:acm:us-east-1:123456789012:certificate/xxxx \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord'    # CNAME to create in Route 53

# Route 53
aws route53 list-hosted-zones --query 'HostedZones[].{id:Id,name:Name,private:Config.PrivateZone}' --output table
aws route53 create-hosted-zone --name internal.example.com --caller-reference "$(date +%s)" \
  --vpc VPCRegion=us-east-1,VPCId="$VPC_ID" --hosted-zone-config PrivateZone=true
aws route53 change-resource-record-sets --hosted-zone-id Z0123456789ABCDEFGHIJ --change-batch '{
  "Changes":[{"Action":"UPSERT","ResourceRecordSet":{"Name":"app.example.com","Type":"A",
   "AliasTarget":{"HostedZoneId":"Z35SXDOTRQ7X7K","DNSName":"lab-alb-123456.us-east-1.elb.amazonaws.com","EvaluateTargetHealth":true}}}]}'
aws route53 list-resource-record-sets --hosted-zone-id Z0123456789ABCDEFGHIJ --query 'ResourceRecordSets[?Type==`A`].[Name,AliasTarget.DNSName]' --output table
aws route53 create-health-check --caller-reference "$(date +%s)" --health-check-config '{"Type":"HTTPS","FullyQualifiedDomainName":"app.example.com","Port":443,"ResourcePath":"/healthz","RequestInterval":30,"FailureThreshold":3}'
aws route53 test-dns-answer --hosted-zone-id Z0123456789ABCDEFGHIJ --record-name app.example.com --record-type A
```

`Z35SXDOTRQ7X7K` is the fixed hosted zone ID for ALBs in `us-east-1`; look up other regions in the ELB endpoints table.

---

## 7. EKS

```bash
# Cluster ($0.10/h) with private endpoint and control plane logging
aws eks create-cluster --name capstone --kubernetes-version 1.31 \
  --role-arn arn:aws:iam::123456789012:role/eks-cluster-role \
  --resources-vpc-config subnetIds=subnet-priv-a,subnet-priv-b,securityGroupIds=sg-eks-cp,endpointPublicAccess=true,endpointPrivateAccess=true,publicAccessCidrs=203.0.113.10/32 \
  --access-config authenticationMode=API_AND_CONFIG_MAP,bootstrapClusterCreatorAdminPermissions=true \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}' \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"arn:aws:kms:us-east-1:123456789012:key/mrk-xxxx"}}]' \
  --tags Project=capstone,Environment=lab
aws eks wait cluster-active --name capstone
aws eks describe-cluster --name capstone --query 'cluster.{status:status,version:version,endpoint:endpoint,oidc:identity.oidc.issuer,platform:platformVersion}'
aws eks update-kubeconfig --name capstone --alias capstone --region us-east-1     # writes exec plugin using aws eks get-token
kubectl get nodes -o wide

# Add-ons
aws eks describe-addon-versions --kubernetes-version 1.31 --addon-name vpc-cni --query 'addons[0].addonVersions[0].addonVersion' --output text
for ADDON in vpc-cni coredns kube-proxy eks-pod-identity-agent; do
  aws eks create-addon --cluster-name capstone --addon-name "$ADDON" --resolve-conflicts OVERWRITE
done
aws eks create-addon --cluster-name capstone --addon-name aws-ebs-csi-driver --pod-identity-associations \
  serviceAccount=ebs-csi-controller-sa,roleArn=arn:aws:iam::123456789012:role/eks-ebs-csi
aws eks list-addons --cluster-name capstone
aws eks describe-addon --cluster-name capstone --addon-name vpc-cni --query 'addon.{status:status,version:addonVersion,health:health.issues}'
aws eks update-addon --cluster-name capstone --addon-name vpc-cni --addon-version v1.19.0-eksbuild.1 --resolve-conflicts PRESERVE

# Managed node group ($) on Graviton Spot
aws eks create-nodegroup --cluster-name capstone --nodegroup-name system-arm64 \
  --node-role arn:aws:iam::123456789012:role/eks-node-role --subnets subnet-priv-a subnet-priv-b \
  --instance-types t4g.medium m7g.large --capacity-type SPOT --ami-type AL2023_ARM_64_STANDARD \
  --scaling-config minSize=1,maxSize=3,desiredSize=2 --update-config maxUnavailable=1 \
  --labels role=system --tags Project=capstone
aws eks wait nodegroup-active --cluster-name capstone --nodegroup-name system-arm64
aws eks describe-nodegroup --cluster-name capstone --nodegroup-name system-arm64 --query 'nodegroup.{status:status,health:health.issues,asg:resources.autoScalingGroups[0].name}'
aws eks update-nodegroup-version --cluster-name capstone --nodegroup-name system-arm64        # roll to latest AMI for the k8s version
aws eks update-nodegroup-config --cluster-name capstone --nodegroup-name system-arm64 --scaling-config minSize=0,maxSize=3,desiredSize=0

# Fargate profile
aws eks create-fargate-profile --cluster-name capstone --fargate-profile-name argocd \
  --pod-execution-role-arn arn:aws:iam::123456789012:role/eks-fargate-pod --subnets subnet-priv-a subnet-priv-b \
  --selectors namespace=argocd

# Access entries (IAM -> Kubernetes authorization)
aws eks create-access-entry --cluster-name capstone --principal-arn arn:aws:iam::123456789012:role/gha-capstone-deploy --type STANDARD
aws eks associate-access-policy --cluster-name capstone --principal-arn arn:aws:iam::123456789012:role/gha-capstone-deploy \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy --access-scope type=cluster
aws eks associate-access-policy --cluster-name capstone --principal-arn arn:aws:iam::123456789012:role/team-a-dev \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy --access-scope type=namespace,namespaces=team-a
aws eks list-access-entries --cluster-name capstone
aws eks list-associated-access-policies --cluster-name capstone --principal-arn arn:aws:iam::123456789012:role/team-a-dev
aws eks list-access-policies --query 'accessPolicies[].name'
aws eks update-cluster-config --name capstone --access-config authenticationMode=API    # drop aws-auth ConfigMap once migrated

# Pod Identity
aws eks create-pod-identity-association --cluster-name capstone --namespace orders --service-account orders-api \
  --role-arn arn:aws:iam::123456789012:role/app-orders
aws eks list-pod-identity-associations --cluster-name capstone --namespace orders
aws eks describe-pod-identity-association --cluster-name capstone --association-id a-xxxx

# IRSA (when needed): OIDC provider from the cluster issuer
ISSUER=$(aws eks describe-cluster --name capstone --query 'cluster.identity.oidc.issuer' --output text)
aws iam create-open-id-connect-provider --url "$ISSUER" --client-id-list sts.amazonaws.com --thumbprint-list ffffffffffffffffffffffffffffffffffffffff
# trust: "Federated": "<provider arn>", condition StringEquals "<issuer-host-path>:sub": "system:serviceaccount:orders:orders-api", "<issuer>:aud": "sts.amazonaws.com"

# Upgrades and maintenance
aws eks describe-cluster-versions --query 'clusterVersions[].{v:clusterVersion,status:status,eol:endOfStandardSupportDate}' --output table
aws eks update-cluster-version --name capstone --kubernetes-version 1.32
aws eks list-updates --name capstone && aws eks describe-update --name capstone --update-id <UPDATE_ID>
aws eks list-insights --cluster-name capstone --query 'insights[].{name:name,status:insightStatus.status}'   # upgrade readiness
aws eks update-cluster-config --name capstone --upgrade-policy supportType=STANDARD   # avoid extended support charges

# Teardown order: nodegroups/fargate -> addons (optional) -> cluster
aws eks delete-nodegroup --cluster-name capstone --nodegroup-name system-arm64 && aws eks wait nodegroup-deleted --cluster-name capstone --nodegroup-name system-arm64
aws eks delete-fargate-profile --cluster-name capstone --fargate-profile-name argocd
aws eks delete-cluster --name capstone && aws eks wait cluster-deleted --name capstone
```

---

## 8. Lambda

```bash
# Zip deploy (Python 3.12, ARM64)
zip -j function.zip handler.py
aws lambda create-function --function-name capstone-hello --runtime python3.12 --architectures arm64 \
  --handler handler.lambda_handler --zip-file fileb://function.zip \
  --role arn:aws:iam::123456789012:role/lambda-hello --memory-size 256 --timeout 10 \
  --environment 'Variables={LOG_LEVEL=INFO}' --logging-config LogFormat=JSON,ApplicationLogLevel=INFO \
  --tracing-config Mode=Active --tags Project=capstone
aws lambda update-function-code --function-name capstone-hello --zip-file fileb://function.zip --publish
aws lambda wait function-updated --function-name capstone-hello

# Container image deploy
aws lambda create-function --function-name capstone-api --package-type Image \
  --code ImageUri=123456789012.dkr.ecr.us-east-1.amazonaws.com/capstone/api:1.4.0 --architectures arm64 \
  --role arn:aws:iam::123456789012:role/lambda-api --memory-size 1024 --timeout 30
aws lambda update-function-code --function-name capstone-api --image-uri 123456789012.dkr.ecr.us-east-1.amazonaws.com/capstone/api:1.4.1

# Invoke
aws lambda invoke --function-name capstone-hello --payload '{"name":"world"}' --cli-binary-format raw-in-base64-out /dev/stdout
aws lambda invoke --function-name capstone-hello --invocation-type Event --payload '{}' --cli-binary-format raw-in-base64-out /dev/null   # async
aws lambda invoke --function-name capstone-hello --log-type Tail --payload '{}' --cli-binary-format raw-in-base64-out out.json --query LogResult --output text | base64 -d

# Configuration
aws lambda update-function-configuration --function-name capstone-hello --memory-size 512 --timeout 20 \
  --vpc-config SubnetIds=subnet-priv-a,subnet-priv-b,SecurityGroupIds=sg-lambda
aws lambda put-function-concurrency --function-name capstone-hello --reserved-concurrent-executions 20
aws lambda publish-version --function-name capstone-hello --description "v3"
aws lambda create-alias --function-name capstone-hello --name prod --function-version 3
aws lambda update-alias --function-name capstone-hello --name prod --function-version 4 --routing-config AdditionalVersionWeights={3=0.9}   # 10% canary to v4
aws lambda put-provisioned-concurrency-config --function-name capstone-hello --qualifier prod --provisioned-concurrent-executions 2   # ($)
aws lambda publish-layer-version --layer-name capstone-deps --zip-file fileb://layer.zip --compatible-runtimes python3.12 --compatible-architectures arm64
aws lambda update-function-configuration --function-name capstone-hello --layers arn:aws:lambda:us-east-1:123456789012:layer:capstone-deps:1

# Event sources and permissions
aws lambda create-event-source-mapping --function-name capstone-worker --event-source-arn arn:aws:sqs:us-east-1:123456789012:orders \
  --batch-size 10 --maximum-batching-window-in-seconds 5 --function-response-types ReportBatchItemFailures
aws lambda create-event-source-mapping --function-name capstone-cdc --event-source-arn arn:aws:dynamodb:us-east-1:123456789012:table/capstone/stream/2025-01-01T00:00:00.000 \
  --starting-position LATEST --batch-size 100 --bisect-batch-on-function-error --maximum-retry-attempts 3 \
  --destination-config '{"OnFailure":{"Destination":"arn:aws:sqs:us-east-1:123456789012:cdc-dlq"}}'
aws lambda add-permission --function-name capstone-hello --statement-id s3invoke --action lambda:InvokeFunction \
  --principal s3.amazonaws.com --source-arn arn:aws:s3:::capstone-uploads-123456789012 --source-account 123456789012
aws lambda create-function-url-config --function-name capstone-hello --auth-type AWS_IAM
aws lambda get-policy --function-name capstone-hello --query Policy --output text | jq .

# Observe
aws logs tail /aws/lambda/capstone-hello --follow --since 10m --format short
aws lambda get-function --function-name capstone-hello --query 'Configuration.{state:State,reason:LastUpdateStatusReason,arch:Architectures[0],mem:MemorySize}'
aws lambda list-functions --query 'Functions[].{name:FunctionName,rt:Runtime,arch:Architectures[0],mem:MemorySize}' --output table
aws lambda delete-function --function-name capstone-hello
```

---

## 9. ECR

```bash
aws ecr create-repository --repository-name capstone/api --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true --encryption-configuration encryptionType=KMS,kmsKey=alias/capstone-ecr --tags Key=Project,Value=capstone
aws ecr put-registry-scanning-configuration --scan-type ENHANCED --rules '[{"scanFrequency":"CONTINUOUS_SCAN","repositoryFilters":[{"filter":"*","filterType":"WILDCARD"}]}]'
aws ecr get-login-password | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
docker buildx build --platform linux/arm64,linux/amd64 --tag 123456789012.dkr.ecr.us-east-1.amazonaws.com/capstone/api:1.4.0 --push .
aws ecr describe-images --repository-name capstone/api --query 'sort_by(imageDetails,&imagePushedAt)[-5:].{tags:imageTags,size:imageSizeInBytes,pushed:imagePushedAt,scan:imageScanStatus.status}' --output table
aws ecr describe-image-scan-findings --repository-name capstone/api --image-id imageTag=1.4.0 --query 'imageScanFindings.findingSeverityCounts'
aws ecr batch-get-image --repository-name capstone/api --image-ids imageTag=1.4.0 --query 'images[0].imageManifest' --output text | jq '.mediaType'
aws ecr put-lifecycle-policy --repository-name capstone/api --lifecycle-policy-text '{"rules":[
  {"rulePriority":1,"description":"expire untagged after 7d","selection":{"tagStatus":"untagged","countType":"sinceImagePushed","countUnit":"days","countNumber":7},"action":{"type":"expire"}},
  {"rulePriority":2,"description":"keep last 30 releases","selection":{"tagStatus":"tagged","tagPatternList":["v*"],"countType":"imageCountMoreThan","countNumber":30},"action":{"type":"expire"}}]}'
aws ecr set-repository-policy --repository-name capstone/api --policy-text '{"Version":"2012-10-17","Statement":[{"Sid":"OrgPull","Effect":"Allow","Principal":"*","Action":["ecr:BatchGetImage","ecr:GetDownloadUrlForLayer"],"Condition":{"StringEquals":{"aws:PrincipalOrgID":"o-exampleorgid"}}}]}'
aws ecr create-pull-through-cache-rule --ecr-repository-prefix docker-hub --upstream-registry-url registry-1.docker.io --credential-arn arn:aws:secretsmanager:us-east-1:123456789012:secret:ecr-pullthroughcache/docker-hub
aws ecr batch-delete-image --repository-name capstone/api --image-ids imageTag=1.3.9
aws ecr delete-repository --repository-name capstone/api --force
```

---

## 10. S3

```bash
# Bucket creation (us-east-1 needs no LocationConstraint; other regions do)
aws s3api create-bucket --bucket capstone-tfstate-123456789012 --region us-east-1
aws s3api create-bucket --bucket capstone-dr-123456789012 --region us-east-2 --create-bucket-configuration LocationConstraint=us-east-2
aws s3api put-bucket-versioning --bucket capstone-tfstate-123456789012 --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket capstone-tfstate-123456789012 --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms","KMSMasterKeyID":"alias/capstone-state"},"BucketKeyEnabled":true}]}'
aws s3api put-public-access-block --bucket capstone-tfstate-123456789012 --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-ownership-controls --bucket capstone-tfstate-123456789012 --ownership-controls 'Rules=[{ObjectOwnership=BucketOwnerEnforced}]'
aws s3api put-bucket-policy --bucket capstone-tfstate-123456789012 --policy file://bucket-policy.json
aws s3api put-bucket-lifecycle-configuration --bucket capstone-tfstate-123456789012 --lifecycle-configuration '{"Rules":[
  {"ID":"noncurrent","Status":"Enabled","Filter":{},"NoncurrentVersionExpiration":{"NoncurrentDays":90,"NewerNoncurrentVersions":10}},
  {"ID":"mpu","Status":"Enabled","Filter":{},"AbortIncompleteMultipartUpload":{"DaysAfterInitiation":7}},
  {"ID":"tier","Status":"Enabled","Filter":{"Prefix":"logs/"},"Transitions":[{"Days":30,"StorageClass":"STANDARD_IA"},{"Days":90,"StorageClass":"GLACIER_IR"}],"Expiration":{"Days":365}}]}'
aws s3api put-bucket-logging --bucket capstone-tfstate-123456789012 --bucket-logging-status '{"LoggingEnabled":{"TargetBucket":"capstone-s3-logs-123456789012","TargetPrefix":"tfstate/"}}'
aws s3api put-bucket-tagging --bucket capstone-tfstate-123456789012 --tagging 'TagSet=[{Key=Project,Value=capstone},{Key=Environment,Value=lab}]'

# Object Lock (bucket must be created with it)
aws s3api create-bucket --bucket capstone-audit-123456789012 --object-lock-enabled-for-bucket
aws s3api put-object-lock-configuration --bucket capstone-audit-123456789012 --object-lock-configuration \
  '{"ObjectLockEnabled":"Enabled","Rule":{"DefaultRetention":{"Mode":"COMPLIANCE","Days":365}}}'

# Replication (needs versioning both sides and a role)
aws s3api put-bucket-replication --bucket capstone-tfstate-123456789012 --replication-configuration '{"Role":"arn:aws:iam::123456789012:role/s3-replication",
  "Rules":[{"ID":"dr","Status":"Enabled","Priority":1,"Filter":{},"DeleteMarkerReplication":{"Status":"Enabled"},
  "Destination":{"Bucket":"arn:aws:s3:::capstone-dr-123456789012","EncryptionConfiguration":{"ReplicaKmsKeyID":"arn:aws:kms:us-east-2:123456789012:key/mrk-xxxx"}},
  "SourceSelectionCriteria":{"SseKmsEncryptedObjects":{"Status":"Enabled"}}}]}'

# Inspect
aws s3api get-bucket-location --bucket capstone-tfstate-123456789012
aws s3api get-bucket-policy --bucket capstone-tfstate-123456789012 --query Policy --output text | jq .
aws s3api get-bucket-policy-status --bucket capstone-tfstate-123456789012 --query 'PolicyStatus.IsPublic'
aws s3api get-bucket-encryption --bucket capstone-tfstate-123456789012
aws s3api list-object-versions --bucket capstone-tfstate-123456789012 --prefix env/lab.tfstate --query 'Versions[].{v:VersionId,latest:IsLatest,mod:LastModified,size:Size}' --output table
aws s3api head-object --bucket capstone-tfstate-123456789012 --key env/lab.tfstate --query '{sse:ServerSideEncryption,kms:SSEKMSKeyId,etag:ETag}'

# High-level s3 commands
aws s3 ls                                                        # buckets
aws s3 ls s3://capstone-tfstate-123456789012/env/ --recursive --human-readable --summarize
aws s3 cp ./artefact.tgz s3://capstone-artefacts-123456789012/releases/1.4.0/ --sse aws:kms --sse-kms-key-id alias/capstone-state
aws s3 sync ./site s3://capstone-site-123456789012/ --delete --exclude '*.map' --cache-control 'max-age=300'
aws s3 mv s3://bucket/a.txt s3://bucket/b.txt
aws s3 rm s3://capstone-artefacts-123456789012/releases/1.3.0/ --recursive
aws s3 presign s3://capstone-artefacts-123456789012/releases/1.4.0/artefact.tgz --expires-in 900   # 15-minute URL

# Empty a versioned bucket (required before delete)
aws s3api list-object-versions --bucket capstone-tfstate-123456789012 --output json \
  --query '{Objects: [Versions[].{Key:Key,VersionId:VersionId}, DeleteMarkers[].{Key:Key,VersionId:VersionId}][]}' > /tmp/del.json
[ "$(jq '.Objects | length' /tmp/del.json)" -gt 0 ] && aws s3api delete-objects --bucket capstone-tfstate-123456789012 --delete file:///tmp/del.json
aws s3api delete-bucket --bucket capstone-tfstate-123456789012
```

---

## 11. DynamoDB

```bash
aws dynamodb create-table --table-name capstone --billing-mode PAY_PER_REQUEST \
  --attribute-definitions AttributeName=PK,AttributeType=S AttributeName=SK,AttributeType=S AttributeName=GSI1PK,AttributeType=S AttributeName=GSI1SK,AttributeType=S \
  --key-schema AttributeName=PK,KeyType=HASH AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes '[{"IndexName":"GSI1","KeySchema":[{"AttributeName":"GSI1PK","KeyType":"HASH"},{"AttributeName":"GSI1SK","KeyType":"RANGE"}],"Projection":{"ProjectionType":"ALL"}}]' \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=alias/capstone-data \
  --deletion-protection-enabled --tags Key=Project,Value=capstone
aws dynamodb wait table-exists --table-name capstone
aws dynamodb update-continuous-backups --table-name capstone --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
aws dynamodb update-time-to-live --table-name capstone --time-to-live-specification Enabled=true,AttributeName=expiresAt

aws dynamodb put-item --table-name capstone --item '{"PK":{"S":"USER#u1"},"SK":{"S":"PROFILE"},"email":{"S":"user@example.com"},"GSI1PK":{"S":"EMAIL#user@example.com"},"GSI1SK":{"S":"USER#u1"}}' \
  --condition-expression 'attribute_not_exists(PK)'                                                    # idempotent create
aws dynamodb get-item --table-name capstone --key '{"PK":{"S":"USER#u1"},"SK":{"S":"PROFILE"}}' --consistent-read
aws dynamodb query --table-name capstone --key-condition-expression 'PK = :pk AND begins_with(SK, :sk)' \
  --expression-attribute-values '{":pk":{"S":"USER#u1"},":sk":{"S":"ORDER#"}}' --scan-index-forward false --limit 20
aws dynamodb query --table-name capstone --index-name GSI1 --key-condition-expression 'GSI1PK = :e' --expression-attribute-values '{":e":{"S":"EMAIL#user@example.com"}}'
aws dynamodb update-item --table-name capstone --key '{"PK":{"S":"USER#u1"},"SK":{"S":"PROFILE"}}' \
  --update-expression 'SET #n = :n ADD loginCount :one' --expression-attribute-names '{"#n":"name"}' --expression-attribute-values '{":n":{"S":"Ada"},":one":{"N":"1"}}' --return-values ALL_NEW
aws dynamodb transact-write-items --transact-items file://txn.json
aws dynamodb execute-statement --statement "SELECT * FROM capstone WHERE PK = 'USER#u1'"      # PartiQL
aws dynamodb scan --table-name capstone --select COUNT                                        # expensive; avoid in prod
aws dynamodb describe-table --table-name capstone --query 'Table.{status:TableStatus,items:ItemCount,bytes:TableSizeBytes,stream:LatestStreamArn,gsis:GlobalSecondaryIndexes[].IndexName}'
aws dynamodb update-table --table-name capstone --billing-mode PROVISIONED --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
aws dynamodb create-backup --table-name capstone --backup-name pre-migration
aws dynamodb restore-table-to-point-in-time --source-table-name capstone --target-table-name capstone-restored --use-latest-restorable-time
aws dynamodb update-table --table-name capstone --no-deletion-protection-enabled && aws dynamodb delete-table --table-name capstone
```

---

## 12. RDS and Aurora

```bash
aws rds create-db-subnet-group --db-subnet-group-name capstone-isolated --db-subnet-group-description "isolated tier" \
  --subnet-ids subnet-iso-a subnet-iso-b --tags Key=Project,Value=capstone
aws rds create-db-parameter-group --db-parameter-group-name capstone-pg16 --db-parameter-group-family postgres16 --description "capstone"
aws rds modify-db-parameter-group --db-parameter-group-name capstone-pg16 --parameters \
  'ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=immediate' \
  'ParameterName=log_min_duration_statement,ParameterValue=500,ApplyMethod=immediate' \
  'ParameterName=shared_preload_libraries,ParameterValue=pg_stat_statements,ApplyMethod=pending-reboot'

# RDS PostgreSQL ($): lab sizing shown; Multi-AZ flag flips HA
aws rds create-db-instance --db-instance-identifier capstone-db --engine postgres --engine-version 16.4 \
  --db-instance-class db.t4g.micro --allocated-storage 20 --storage-type gp3 --storage-encrypted --kms-key-id alias/capstone-data \
  --master-username capstone_admin --manage-master-user-password --master-user-secret-kms-key-id alias/capstone-data \
  --db-subnet-group-name capstone-isolated --vpc-security-group-ids sg-db --db-parameter-group-name capstone-pg16 \
  --no-publicly-accessible --no-multi-az --backup-retention-period 7 --preferred-backup-window 03:00-04:00 \
  --enable-iam-database-authentication --enable-cloudwatch-logs-exports '["postgresql"]' \
  --enable-performance-insights --performance-insights-retention-period 7 --monitoring-interval 60 --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring \
  --copy-tags-to-snapshot --deletion-protection --auto-minor-version-upgrade --tags Key=Project,Value=capstone
aws rds wait db-instance-available --db-instance-identifier capstone-db
aws rds describe-db-instances --db-instance-identifier capstone-db --query 'DBInstances[0].{status:DBInstanceStatus,endpoint:Endpoint.Address,port:Endpoint.Port,az:AvailabilityZone,multiAZ:MultiAZ,secret:MasterUserSecret.SecretArn}'
aws rds modify-db-instance --db-instance-identifier capstone-db --multi-az --apply-immediately          # enable HA (lab 7b)
aws rds reboot-db-instance --db-instance-identifier capstone-db --force-failover                        # DR drill
aws rds describe-events --source-identifier capstone-db --source-type db-instance --duration 60 --query 'Events[].{t:Date,m:Message}' --output table
aws rds generate-db-auth-token --hostname capstone-db.cxxxx.us-east-1.rds.amazonaws.com --port 5432 --username app_iam   # 15-min IAM token
aws rds create-db-snapshot --db-instance-identifier capstone-db --db-snapshot-identifier capstone-db-pre-upgrade
aws rds restore-db-instance-to-point-in-time --source-db-instance-identifier capstone-db --target-db-instance-identifier capstone-db-pitr --use-latest-restorable-time --db-subnet-group-name capstone-isolated --vpc-security-group-ids sg-db
aws rds describe-db-engine-versions --engine postgres --engine-version 16.4 --query 'DBEngineVersions[0].ValidUpgradeTarget[].EngineVersion'
aws rds create-blue-green-deployment --blue-green-deployment-name capstone-pg17 --source arn:aws:rds:us-east-1:123456789012:db:capstone-db --target-engine-version 17.2
aws rds modify-db-instance --db-instance-identifier capstone-db --no-deletion-protection --apply-immediately
aws rds delete-db-instance --db-instance-identifier capstone-db --skip-final-snapshot --delete-automated-backups   # lab teardown only

# RDS Proxy ($)
aws rds create-db-proxy --db-proxy-name capstone-proxy --engine-family POSTGRESQL --role-arn arn:aws:iam::123456789012:role/rds-proxy \
  --auth '[{"AuthScheme":"SECRETS","SecretArn":"arn:aws:secretsmanager:us-east-1:123456789012:secret:rds!db-xxxx","IAMAuth":"REQUIRED"}]' \
  --vpc-subnet-ids subnet-priv-a subnet-priv-b --vpc-security-group-ids sg-db --require-tls
aws rds register-db-proxy-targets --db-proxy-name capstone-proxy --db-instance-identifiers capstone-db
aws rds describe-db-proxy-targets --db-proxy-name capstone-proxy --query 'Targets[].TargetHealth'

# Aurora PostgreSQL Serverless v2 ($)
aws rds create-db-cluster --db-cluster-identifier capstone-aurora --engine aurora-postgresql --engine-version 16.4 \
  --master-username capstone_admin --manage-master-user-password --db-subnet-group-name capstone-isolated --vpc-security-group-ids sg-db \
  --storage-encrypted --kms-key-id alias/capstone-data --serverless-v2-scaling-configuration MinCapacity=0,MaxCapacity=4 \
  --enable-iam-database-authentication --backup-retention-period 7 --deletion-protection
aws rds create-db-instance --db-instance-identifier capstone-aurora-1 --db-cluster-identifier capstone-aurora --engine aurora-postgresql --db-instance-class db.serverless
aws rds describe-db-clusters --db-cluster-identifier capstone-aurora --query 'DBClusters[0].{writer:Endpoint,reader:ReaderEndpoint,members:DBClusterMembers[].[DBInstanceIdentifier,IsClusterWriter]}'
aws rds failover-db-cluster --db-cluster-identifier capstone-aurora
aws rds create-global-cluster --global-cluster-identifier capstone-global --source-db-cluster-identifier arn:aws:rds:us-east-1:123456789012:cluster:capstone-aurora
aws rds switchover-global-cluster --global-cluster-identifier capstone-global --target-db-cluster-identifier arn:aws:rds:us-east-2:123456789012:cluster:capstone-aurora-dr
```

---

## 13. KMS, Secrets Manager, Parameter Store

```bash
# KMS ($1/month per CMK)
KEY_ID=$(aws kms create-key --description "capstone terraform state" --key-usage ENCRYPT_DECRYPT --key-spec SYMMETRIC_DEFAULT \
  --multi-region --policy file://key-policy.json --tags TagKey=Project,TagValue=capstone --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/capstone-state --target-key-id "$KEY_ID"
aws kms enable-key-rotation --key-id "$KEY_ID" --rotation-period-in-days 365
aws kms describe-key --key-id alias/capstone-state --query 'KeyMetadata.{id:KeyId,arn:Arn,state:KeyState,mr:MultiRegion,rotation:KeyManager}'
aws kms get-key-policy --key-id alias/capstone-state --policy-name default --query Policy --output text | jq .
aws kms put-key-policy --key-id alias/capstone-state --policy-name default --policy file://key-policy.json
aws kms list-aliases --query 'Aliases[?starts_with(AliasName, `alias/capstone`)].{alias:AliasName,key:TargetKeyId}' --output table
aws kms list-grants --key-id "$KEY_ID" --query 'Grants[].{grantee:GranteePrincipal,ops:Operations}'
aws kms replicate-key --key-id "$KEY_ID" --replica-region us-east-2 --policy file://key-policy.json
echo -n "hello" | base64 | xargs -I{} aws kms encrypt --key-id alias/capstone-state --plaintext {} --encryption-context purpose=test --query CiphertextBlob --output text > /tmp/ct.b64
aws kms decrypt --ciphertext-blob fileb://<(base64 -d /tmp/ct.b64) --encryption-context purpose=test --query Plaintext --output text | base64 -d
aws kms generate-data-key --key-id alias/capstone-state --key-spec AES_256 --query '{ct:CiphertextBlob}'   # envelope encryption primitive
aws kms update-alias --alias-name alias/capstone-state --target-key-id "$NEW_KEY_ID"              # manual key rotation
aws kms schedule-key-deletion --key-id "$KEY_ID" --pending-window-in-days 7                          # min 7 days; irreversible after
aws kms cancel-key-deletion --key-id "$KEY_ID"

# Secrets Manager ($0.40/secret/month)
aws secretsmanager create-secret --name /capstone/lab/api/db --description "orders api DB creds" --kms-key-id alias/capstone-data \
  --secret-string '{"username":"app_user","password":"<GENERATED_BY_ROTATION>","host":"capstone-proxy.proxy-cxxxx.us-east-1.rds.amazonaws.com","port":5432,"dbname":"orders"}' \
  --tags Key=Project,Value=capstone
aws secretsmanager get-random-password --password-length 32 --exclude-punctuation --query RandomPassword --output text   # generate, never type
aws secretsmanager get-secret-value --secret-id /capstone/lab/api/db --query SecretString --output text | jq -r .host    # fetch one field
aws secretsmanager put-secret-value --secret-id /capstone/lab/api/db --secret-string file://new.json
aws secretsmanager list-secret-version-ids --secret-id /capstone/lab/api/db --query 'Versions[].{id:VersionId,stages:VersionStages}'
aws secretsmanager rotate-secret --secret-id /capstone/lab/api/db --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:SecretsManagerRDSPostgreSQLRotationSingleUser \
  --rotation-rules AutomaticallyAfterDays=30
aws secretsmanager describe-secret --secret-id /capstone/lab/api/db --query '{rotation:RotationEnabled,last:LastRotatedDate,next:NextRotationDate,lambda:RotationLambdaARN}'
aws secretsmanager put-resource-policy --secret-id /capstone/lab/api/db --resource-policy file://secret-policy.json --block-public-policy
aws secretsmanager list-secrets --filters Key=tag-key,Values=Project --query 'SecretList[].{name:Name,rot:RotationEnabled}' --output table
aws secretsmanager delete-secret --secret-id /capstone/lab/api/db --recovery-window-in-days 7      # recoverable
aws secretsmanager delete-secret --secret-id /capstone/lab/api/db --force-delete-without-recovery  # lab teardown only
aws secretsmanager restore-secret --secret-id /capstone/lab/api/db

# Parameter Store (standard tier free)
aws ssm put-parameter --name /capstone/lab/api/log_level --type String --value INFO --tags Key=Project,Value=capstone
aws ssm put-parameter --name /capstone/lab/api/feature_flags --type StringList --value "new_checkout,dark_mode" --overwrite
aws ssm put-parameter --name /capstone/lab/api/webhook_key --type SecureString --key-id alias/capstone-data --value "$(aws secretsmanager get-random-password --password-length 40 --query RandomPassword --output text)"
aws ssm get-parameter --name /capstone/lab/api/webhook_key --with-decryption --query 'Parameter.Value' --output text
aws ssm get-parameters-by-path --path /capstone/lab/api --recursive --with-decryption --query 'Parameters[].{name:Name,type:Type,version:Version}' --output table
aws ssm get-parameter-history --name /capstone/lab/api/log_level --query 'Parameters[].{v:Version,val:Value,who:LastModifiedUser}'
aws ssm label-parameter-version --name /capstone/lab/api/log_level --labels prod --parameter-version 3
aws ssm delete-parameter --name /capstone/lab/api/log_level
```

---

## 14. CloudWatch, CloudTrail, Config, GuardDuty, Security Hub

```bash
# CloudWatch Logs
aws logs describe-log-groups --query 'logGroups[?!retentionInDays].{name:logGroupName,bytes:storedBytes}' --output table   # groups with no retention
aws logs put-retention-policy --log-group-name /aws/eks/capstone/cluster --retention-in-days 30
aws logs tail /aws/eks/capstone/cluster --follow --filter-pattern '{ $.verb = "delete" }' --format short
aws logs filter-log-events --log-group-name /aws/lambda/capstone-api --start-time $(( $(date +%s) - 3600 ))000 --filter-pattern 'ERROR' --query 'events[].message' --output text
aws logs start-query --log-group-names /aws/lambda/capstone-api --start-time $(date -d '-24 hours' +%s) --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter level = "ERROR" | stats count() by bin(1h)'
aws logs put-metric-filter --log-group-name /aws/lambda/capstone-api --filter-name errors --filter-pattern '{ $.level = "ERROR" }' \
  --metric-transformations metricName=ApiErrors,metricNamespace=Capstone,metricValue=1,defaultValue=0
aws logs put-subscription-filter --log-group-name /aws/lambda/capstone-api --filter-name to-otel --filter-pattern '' \
  --destination-arn arn:aws:lambda:us-east-1:123456789012:function:log-forwarder

# CloudWatch Metrics and Alarms
aws cloudwatch list-metrics --namespace AWS/RDS --dimensions Name=DBInstanceIdentifier,Value=capstone-db --query 'Metrics[].MetricName' --output text
aws cloudwatch get-metric-statistics --namespace AWS/NATGateway --metric-name BytesOutToDestination --dimensions Name=NatGatewayId,Value=nat-xxxx \
  --start-time "$(date -u -d '-1 day' +%FT%TZ)" --end-time "$(date -u +%FT%TZ)" --period 3600 --statistics Sum --query 'Datapoints[].[Timestamp,Sum]' --output table
aws cloudwatch get-metric-data --metric-data-queries file://queries.json --start-time "$(date -u -d '-3 hours' +%FT%TZ)" --end-time "$(date -u +%FT%TZ)"
aws cloudwatch put-metric-alarm --alarm-name rds-free-storage-low --namespace AWS/RDS --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=capstone-db --statistic Minimum --period 300 --evaluation-periods 2 \
  --threshold 2147483648 --comparison-operator LessThanThreshold --treat-missing-data missing \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts --ok-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
aws cloudwatch put-metric-alarm --alarm-name nat-egress-anomaly --metrics file://anomaly-metrics.json --comparison-operator GreaterThanUpperThreshold --threshold-metric-id ad1 --evaluation-periods 2
aws cloudwatch put-composite-alarm --alarm-name api-degraded --alarm-rule 'ALARM("api-5xx") AND ALARM("api-latency-p99")' --alarm-actions arn:aws:sns:us-east-1:123456789012:pager
aws cloudwatch describe-alarms --state-value ALARM --query 'MetricAlarms[].{name:AlarmName,reason:StateReason}' --output table
aws cloudwatch set-alarm-state --alarm-name rds-free-storage-low --state-value ALARM --state-reason "test notification"
aws cloudwatch put-metric-data --namespace Capstone --metric-name DeployCount --value 1 --dimensions Service=orders-api --unit Count

# CloudTrail
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket --start-time "$(date -u -d '-7 days' +%FT%TZ)" \
  --query 'Events[].{t:EventTime,who:Username,src:CloudTrailEvent}' --output json | jq '.[] | {t, who, ip: (.src|fromjson).sourceIPAddress}'
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=gha-capstone-deploy --max-results 20 --query 'Events[].[EventTime,EventName]' --output table
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=sg-app --query 'Events[].[EventTime,EventName,Username]' --output table
aws cloudtrail put-event-selectors --trail-name capstone-trail --advanced-event-selectors '[{"Name":"S3 data events on state bucket","FieldSelectors":[{"Field":"eventCategory","Equals":["Data"]},{"Field":"resources.type","Equals":["AWS::S3::Object"]},{"Field":"resources.ARN","StartsWith":["arn:aws:s3:::capstone-tfstate-123456789012/"]}]}]'
aws cloudtrail get-event-selectors --trail-name capstone-trail
aws cloudtrail validate-logs --trail-arn arn:aws:cloudtrail:us-east-1:123456789012:trail/capstone-trail --start-time "$(date -u -d '-1 day' +%FT%TZ)"
aws cloudtrail describe-trails --query 'trailList[].{name:Name,multi:IsMultiRegionTrail,org:IsOrganizationTrail,bucket:S3BucketName,validation:LogFileValidationEnabled}' --output table

# AWS Config
aws configservice put-configuration-recorder --configuration-recorder name=default,roleARN=arn:aws:iam::123456789012:role/aws-service-role/config.amazonaws.com/AWSServiceRoleForConfig \
  --recording-group allSupported=true,includeGlobalResourceTypes=true
aws configservice put-delivery-channel --delivery-channel name=default,s3BucketName=capstone-config-123456789012,configSnapshotDeliveryProperties={deliveryFrequency=TwentyFour_Hours}
aws configservice start-configuration-recorder --configuration-recorder-name default
aws configservice describe-configuration-recorder-status --query 'ConfigurationRecordersStatus[0].{recording:recording,status:lastStatus}'
aws configservice put-config-rule --config-rule '{"ConfigRuleName":"s3-bucket-public-read-prohibited","Source":{"Owner":"AWS","SourceIdentifier":"S3_BUCKET_PUBLIC_READ_PROHIBITED"}}'
aws configservice put-config-rule --config-rule '{"ConfigRuleName":"required-tags","Source":{"Owner":"AWS","SourceIdentifier":"REQUIRED_TAGS"},"InputParameters":"{\"tag1Key\":\"Project\",\"tag2Key\":\"Environment\",\"tag3Key\":\"Owner\"}"}'
aws configservice put-conformance-pack --conformance-pack-name operational-best-practices-for-cis --template-s3-uri s3://aws-configservice-conformance-packs/Operational-Best-Practices-for-CIS-AWS-v1.4-Level2.yaml
aws configservice describe-compliance-by-config-rule --compliance-types NON_COMPLIANT --query 'ComplianceByConfigRules[].ConfigRuleName' --output text
aws configservice get-compliance-details-by-config-rule --config-rule-name required-tags --compliance-types NON_COMPLIANT --query 'EvaluationResults[].EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId' --output text
aws configservice get-resource-config-history --resource-type AWS::EC2::SecurityGroup --resource-id sg-app --limit 5 --query 'configurationItems[].{t:configurationItemCaptureTime,status:configurationItemStatus}'
aws configservice select-resource-config --expression "SELECT resourceId, resourceType WHERE resourceType = 'AWS::EC2::Volume' AND configuration.state = 'available'"   # SQL over inventory
aws configservice put-remediation-configurations --remediation-configurations '[{"ConfigRuleName":"s3-bucket-public-read-prohibited","TargetType":"SSM_DOCUMENT","TargetId":"AWSConfigRemediation-RemoveS3BucketPolicyPublicAccess","Automatic":true,"MaximumAutomaticAttempts":3,"RetryAttemptSeconds":60,"Parameters":{"AutomationAssumeRole":{"StaticValue":{"Values":["arn:aws:iam::123456789012:role/config-remediation"]}},"BucketName":{"ResourceValue":{"Value":"RESOURCE_ID"}}}}]'

# GuardDuty
DETECTOR=$(aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES \
  --features '[{"Name":"S3_DATA_EVENTS","Status":"ENABLED"},{"Name":"EKS_AUDIT_LOGS","Status":"ENABLED"},{"Name":"RUNTIME_MONITORING","Status":"ENABLED","AdditionalConfiguration":[{"Name":"EKS_ADDON_MANAGEMENT","Status":"ENABLED"}]},{"Name":"LAMBDA_NETWORK_LOGS","Status":"ENABLED"},{"Name":"RDS_LOGIN_EVENTS","Status":"ENABLED"}]' \
  --query DetectorId --output text)
aws guardduty list-detectors
aws guardduty get-detector --detector-id "$DETECTOR" --query '{status:Status,features:Features[].[Name,Status]}'
aws guardduty list-findings --detector-id "$DETECTOR" --finding-criteria '{"Criterion":{"severity":{"Gte":7},"service.archived":{"Eq":["false"]}}}' --query FindingIds --output text
aws guardduty get-findings --detector-id "$DETECTOR" --finding-ids <ID> --query 'Findings[].{type:Type,sev:Severity,res:Resource.ResourceType,desc:Description}'
aws guardduty create-sample-findings --detector-id "$DETECTOR" --finding-types UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS   # test the pipeline
aws guardduty archive-findings --detector-id "$DETECTOR" --finding-ids <ID>
aws guardduty create-threat-intel-set --detector-id "$DETECTOR" --name corp-ioc --format TXT --location s3://capstone-sec-123456789012/ioc.txt --activate
aws guardduty delete-detector --detector-id "$DETECTOR"

# Security Hub
aws securityhub enable-security-hub --enable-default-standards --control-finding-generator SECURITY_CONTROL --tags Project=capstone
aws securityhub describe-standards --query 'Standards[].{arn:StandardsArn,name:Name}' --output table
aws securityhub batch-enable-standards --standards-subscription-requests StandardsArn=arn:aws:securityhub:us-east-1::standards/cis-aws-foundations-benchmark/v/3.0.0
aws securityhub get-enabled-standards --query 'StandardsSubscriptions[].{arn:StandardsArn,status:StandardsStatus}'
aws securityhub get-findings --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}],"RecordState":[{"Value":"ACTIVE","Comparison":"EQUALS"}],"WorkflowStatus":[{"Value":"NEW","Comparison":"EQUALS"}]}' \
  --query 'Findings[].{title:Title,res:Resources[0].Id,gen:GeneratorId}' --output table --max-items 20
aws securityhub get-findings --filters '{"ComplianceStatus":[{"Value":"FAILED","Comparison":"EQUALS"}],"ProductName":[{"Value":"Security Hub","Comparison":"EQUALS"}]}' --query 'length(Findings)'
aws securityhub batch-update-findings --finding-identifiers Id=<FINDING_ID>,ProductArn=<PRODUCT_ARN> --workflow Status=SUPPRESSED --note Text="lab exception, tracked in ADR-0007",UpdatedBy=platform
aws securityhub update-security-control --security-control-id CloudTrail.1 --parameters file://control-params.json
aws securityhub batch-update-standards-control-associations --standards-control-association-updates '[{"StandardsArn":"arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0","SecurityControlId":"EC2.10","AssociationStatus":"DISABLED","UpdatedReason":"VPC endpoints not required in lab"}]'
aws securityhub create-automation-rule --rule-name "suppress-lab-informational" --rule-order 1 --rule-status ENABLED --description "Lab noise" \
  --criteria '{"SeverityLabel":[{"Value":"INFORMATIONAL","Comparison":"EQUALS"}],"ResourceTags":[{"Key":"Environment","Value":"lab","Comparison":"EQUALS"}]}' \
  --actions '[{"Type":"FINDING_FIELDS_UPDATE","FindingFieldsUpdate":{"Workflow":{"Status":"SUPPRESSED"}}}]'
aws securityhub get-insight-results --insight-arn arn:aws:securityhub:::insight/securityhub/default/1   # findings by resource type
aws securityhub disable-security-hub

# EventBridge rule to route GuardDuty high-severity findings to SNS
aws events put-rule --name guardduty-high --event-pattern '{"source":["aws.guardduty"],"detail-type":["GuardDuty Finding"],"detail":{"severity":[{"numeric":[">=",7]}]}}'
aws events put-targets --rule guardduty-high --targets Id=sns,Arn=arn:aws:sns:us-east-1:123456789012:security-alerts
```

---

## 15. Tagging and Resource Discovery

```bash
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values=capstone --query 'ResourceTagMappingList[].ResourceARN' --output text | tr '\t' '\n'
aws resourcegroupstaggingapi get-resources --resource-type-filters ec2:instance rds:db eks:cluster --tag-filters Key=TTL --query 'ResourceTagMappingList[].{arn:ResourceARN,ttl:Tags[?Key==`TTL`].Value|[0]}' --output table
aws resourcegroupstaggingapi tag-resources --resource-arn-list arn:aws:ec2:us-east-1:123456789012:instance/i-xxxx --tags Owner=platform
aws resourcegroupstaggingapi get-tag-keys --query TagKeys --output text
aws resource-explorer-2 search --query-string "region:us-east-1 -tag:Project" --query 'Resources[].{arn:Arn,type:ResourceType}' --output table   # untagged resources (needs an index)
aws ec2 describe-tags --filters Name=resource-type,Values=instance Name=key,Values=Name --query 'Tags[].{id:ResourceId,name:Value}' --output table
aws ec2 create-tags --resources i-xxxx vol-xxxx --tags Key=TTL,Value=24h
```

---

## Cost and Cleanup

### 16.1 Cost queries

```bash
# Daily spend for the last week
aws ce get-cost-and-usage --time-period Start=$(date -d '-7 days' +%F),End=$(date +%F) --granularity DAILY --metrics UnblendedCost \
  --query 'ResultsByTime[].{date:TimePeriod.Start,usd:Total.UnblendedCost.Amount}' --output table

# Month-to-date by service, descending
aws ce get-cost-and-usage --time-period Start=$(date +%Y-%m-01),End=$(date +%F) --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE --query 'ResultsByTime[0].Groups[?Metrics.UnblendedCost.Amount > `0.01`].[Keys[0],Metrics.UnblendedCost.Amount]' --output text | sort -k2 -rn | head -15

# By Project tag
aws ce get-cost-and-usage --time-period Start=$(date +%Y-%m-01),End=$(date +%F) --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=TAG,Key=Project --query 'ResultsByTime[0].Groups[].[Keys[0],Metrics.UnblendedCost.Amount]' --output table

# Forecast to end of month
aws ce get-cost-forecast --time-period Start=$(date +%F),End=$(date -d "$(date +%Y-%m-01) +1 month" +%F) --metric UNBLENDED_COST --granularity MONTHLY --query 'Total.Amount'

# Usage-type drill-down for NAT and data transfer
aws ce get-cost-and-usage --time-period Start=$(date -d '-30 days' +%F),End=$(date +%F) --granularity MONTHLY --metrics UnblendedCost UsageQuantity \
  --filter '{"Dimensions":{"Key":"USAGE_TYPE_GROUP","Values":["EC2: NAT Gateway - Running Hours","EC2: NAT Gateway - Data Processed","EC2: Data Transfer - Inter AZ"]}}' \
  --group-by Type=DIMENSION,Key=USAGE_TYPE --query 'ResultsByTime[0].Groups[].[Keys[0],Metrics.UsageQuantity.Amount,Metrics.UnblendedCost.Amount]' --output table

# Savings Plans recommendations (only meaningful after 30+ days)
aws ce get-savings-plans-purchase-recommendation --savings-plans-type COMPUTE_SP --term-in-years ONE_YEAR --payment-option NO_UPFRONT --lookback-period-in-days THIRTY_DAYS \
  --query 'SavingsPlansPurchaseRecommendation.SavingsPlansPurchaseRecommendationSummary.{hourly:HourlyCommitmentToPurchase,savingsPct:EstimatedSavingsPercentage}'
aws ce get-rightsizing-recommendation --service AmazonEC2 --query 'RightsizingRecommendations[].{id:CurrentInstance.ResourceId,action:RightsizingType,save:ModifyRecommendationDetail.TargetInstances[0].EstimatedMonthlySavings}'
aws compute-optimizer get-ec2-instance-recommendations --query 'instanceRecommendations[].{id:instanceArn,finding:finding,rec:recommendationOptions[0].instanceType}' --output table
aws budgets describe-budget-performance-history --account-id 123456789012 --budget-name capstone-lab-monthly --query 'BudgetPerformanceHistory.BudgetedAndActualAmountsList[-3:]'
aws ce get-anomalies --date-interval Start=$(date -d '-30 days' +%F) --query 'Anomalies[].{svc:RootCauses[0].Service,impact:Impact.TotalImpact,start:AnomalyStartDate}' --output table
```

### 16.2 Orphan sweep

Run after every teardown. Each block prints candidates; uncomment the delete lines once you have reviewed them.

```bash
#!/usr/bin/env bash
# orphan-sweep.sh: find resources that cost money and are not attached to anything
set -euo pipefail
R=us-east-1

echo "== Unattached EBS volumes"
aws ec2 describe-volumes --region $R --filters Name=status,Values=available --query 'Volumes[].[VolumeId,Size,CreateTime]' --output text
# aws ec2 describe-volumes --region $R --filters Name=status,Values=available --query 'Volumes[].VolumeId' --output text | xargs -r -n1 aws ec2 delete-volume --region $R --volume-id

echo "== Unassociated Elastic IPs"
aws ec2 describe-addresses --region $R --query 'Addresses[?AssociationId==null].[AllocationId,PublicIp]' --output text
# aws ec2 describe-addresses --region $R --query 'Addresses[?AssociationId==null].AllocationId' --output text | xargs -r -n1 aws ec2 release-address --region $R --allocation-id

echo "== NAT gateways (any state but deleted)"
aws ec2 describe-nat-gateways --region $R --filter Name=state,Values=available,pending --query 'NatGateways[].[NatGatewayId,VpcId,CreateTime]' --output text

echo "== Interface VPC endpoints"
aws ec2 describe-vpc-endpoints --region $R --filters Name=vpc-endpoint-type,Values=Interface --query 'VpcEndpoints[].[VpcEndpointId,ServiceName]' --output text

echo "== Load balancers"
aws elbv2 describe-load-balancers --region $R --query 'LoadBalancers[].[LoadBalancerName,Type,CreatedTime]' --output text

echo "== EKS clusters"
aws eks list-clusters --region $R --query clusters --output text

echo "== RDS instances and clusters"
aws rds describe-db-instances --region $R --query 'DBInstances[].[DBInstanceIdentifier,DBInstanceClass,MultiAZ]' --output text
aws rds describe-db-clusters --region $R --query 'DBClusters[].[DBClusterIdentifier,Engine]' --output text

echo "== Running EC2"
aws ec2 describe-instances --region $R --filters Name=instance-state-name,Values=running,stopped --query 'Reservations[].Instances[].[InstanceId,InstanceType,State.Name,Tags[?Key==`Name`].Value|[0]]' --output text

echo "== Snapshots older than 30 days owned by me"
aws ec2 describe-snapshots --region $R --owner-ids self --query "Snapshots[?StartTime<'$(date -d '-30 days' +%F)'].[SnapshotId,VolumeSize,StartTime]" --output text

echo "== Log groups without retention"
aws logs describe-log-groups --region $R --query 'logGroups[?!retentionInDays].logGroupName' --output text | tr '\t' '\n'
# ... | xargs -r -I{} aws logs put-retention-policy --region $R --log-group-name {} --retention-in-days 14

echo "== KMS customer keys (1 USD/month each)"
aws kms list-keys --region $R --query 'Keys[].KeyId' --output text | xargs -r -n1 -I{} aws kms describe-key --region $R --key-id {} \
  --query 'KeyMetadata[?KeyManager==`CUSTOMER` && KeyState==`Enabled`].[KeyId,Description]' --output text

echo "== Secrets"
aws secretsmanager list-secrets --region $R --query 'SecretList[].[Name,LastAccessedDate]' --output text

echo "== Buckets"
aws s3api list-buckets --query 'Buckets[].Name' --output text | tr '\t' '\n'

echo "== Resources tagged TTL (should all be gone after teardown)"
aws resourcegroupstaggingapi get-resources --region $R --tag-filters Key=TTL --query 'ResourceTagMappingList[].ResourceARN' --output text | tr '\t' '\n'
```

### 16.3 Nuclear option

For a dedicated lab account only, [aws-nuke](https://github.com/ekristen/aws-nuke) deletes everything not on an allow-list. Configure it to exclude the SSO roles, the CloudTrail bucket, the budget, and the OIDC provider; run it from the nightly GitHub Actions cron as a last resort after `terraform destroy`. Never point it at an account that shares anything with production.

Next: [04-hands-on-labs.md](04-hands-on-labs.md).
