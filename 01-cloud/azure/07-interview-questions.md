# Azure: Interview Questions

| | |
|---|---|
| **Cluster** | 01 Hyperscale Cloud |
| **Role** | Bridge (anchor is AWS) |
| **Prerequisites** | All previous files in this folder |
| **Target version** | Azure CLI 2.60+, AKS 1.30+ |
| **Capstone phase** | Phase 2 (Months 5-8) |

> Thirty questions from junior to principal level, each with a model answer. Many interviewers for multi-cloud roles will ask you to compare against AWS; the answers do so where it strengthens the response. Practise saying these out loud in under two minutes each.

## Junior

### Q1. What is a Resource Group and why does Azure have one when AWS does not? (Junior)

**Answer.** A Resource Group is a mandatory logical container that every Azure resource belongs to exactly once. It is a scope for RBAC, Policy, locks, tags, and deployments, and deleting it deletes everything inside. AWS relies on tags and CloudFormation stacks for grouping; Azure makes the lifecycle boundary explicit so that the platform, not convention, enforces it. The trade-off is that you must choose boundaries early; moving resources between RGs has per-service restrictions.

### Q2. Explain the hierarchy Tenant, Management Group, Subscription, Resource Group. (Junior)

**Answer.** The Entra ID tenant is the identity directory that owns subscriptions. Management Groups form a tree (up to six levels under the root) used purely for inheriting Policy and RBAC. Subscriptions are billing and quota boundaries and the unit of resource provider registration, roughly an AWS account. Resource Groups sit inside subscriptions and hold resources. Policy and RBAC assigned at any level inherit downward.

### Q3. What is Azure Resource Manager? (Junior)

**Answer.** ARM is the single control-plane API (`management.azure.com`) that every tool uses: portal, `az`, Bicep, Terraform, SDKs. It authenticates the caller with Entra ID, authorises with RBAC, evaluates Azure Policy, then forwards to the resource provider (`Microsoft.Compute`, `Microsoft.Network`). Resource operations are idempotent `PUT`s, and templates (ARM JSON, compiled from Bicep) are deployed as `Microsoft.Resources/deployments`.

### Q4. What is the difference between a system-assigned and a user-assigned managed identity? (Junior)

**Answer.** Both are Entra service principals whose credentials Azure manages and exposes via IMDS. A system-assigned identity is created and deleted with its resource and belongs to one resource. A user-assigned identity is a standalone resource that can be attached to many resources and can be granted roles before the consuming resource exists, which avoids chicken-and-egg problems in IaC. Default to user-assigned for anything deployed by pipelines.

### Q5. Name the storage redundancy options and the production minimum. (Junior)

**Answer.** LRS (three copies in one datacenter), ZRS (three zones in a region), GRS and RA-GRS (LRS plus async copy to the paired region, read access optional), GZRS and RA-GZRS (ZRS plus paired region). ZRS is the production minimum because LRS does not survive a zone failure; S3 is effectively zone-redundant by default, so an AWS engineer moving to Azure must choose this explicitly.

### Q6. What does `az account set` do and what is the AWS equivalent? (Junior)

**Answer.** It selects the active subscription for subsequent `az` commands. All commands target that subscription unless `--subscription` overrides it. The AWS equivalent is switching `AWS_PROFILE`, except on Azure one login grants tokens valid across all subscriptions in the tenant, so you switch context rather than re-authenticate.

### Q7. What is a Network Security Group and how does it differ from an AWS security group? (Junior)

**Answer.** An NSG is a stateful list of allow and deny rules with priorities 100-4096, attachable to subnets and NICs; both are evaluated if present. Unlike AWS security groups, NSGs support explicit deny rules and have default rules (allow VNet, allow Azure Load Balancer, deny all other inbound; allow all outbound). Service tags and Application Security Groups replace prefix lists and security group references as sources and destinations.

## Mid-Level

### Q8. Walk through how an Azure RBAC authorisation decision is made. (Mid-level)

**Answer.** Collect all role assignments for the principal and its group memberships at the target scope and every parent scope (resource, RG, subscription, MG). Union the role definitions' `Actions` minus `NotActions` for control plane, and `DataActions` minus `NotDataActions` for data plane. Then subtract any deny assignments (which only Deployment Stacks, Blueprints, or managed apps can create). If the requested operation is in the remaining set, allow. There is no user-authored deny, so preventive controls come from Azure Policy, and ABAC conditions on assignments refine data access.

### Q9. Azure RBAC roles versus Entra ID roles: when do you need each? (Mid-level)

**Answer.** Azure RBAC governs Azure resources and is hierarchical. Entra roles govern the directory: creating app registrations, managing users and groups, Conditional Access. A subscription Owner cannot create an app registration without an Entra role such as Application Administrator, and a Global Administrator cannot touch resources until they elevate to User Access Administrator at the root MG. Workload identity setup touches both: Entra role to create the app and federated credential, Azure RBAC to grant it resource access.

### Q10. How does workload identity federation for GitHub Actions work, and how does it compare to AWS? (Mid-level)

**Answer.** You add a federated credential to an app registration or user-assigned MI naming the issuer (`https://token.actions.githubusercontent.com`), subject (`repo:org/repo:environment:production`), and audience (`api://AzureADTokenExchange`). The workflow requests an OIDC token from GitHub with `id-token: write`, and `azure/login` presents it as a client assertion to Entra, which validates the claims and returns an Azure access token. No secret is stored. AWS does the same with an IAM OIDC provider and a role trust policy conditioned on `sub` and `aud`.

### Q11. Compare Azure Policy with AWS SCPs. (Mid-level)

**Answer.** Both are preventive guardrails inherited through a hierarchy. SCPs only deny and only filter IAM permissions. Azure Policy inspects the resource request body and supports `Deny`, `Audit`, `AuditIfNotExists`, `DeployIfNotExists`, `Modify`, `Append`, and `DenyAction`, so it also covers what AWS Config rules and auto-remediation do. Policy effects that create or modify resources need a managed identity with the appropriate role. Compliance evaluation is immediate on writes but periodic (up to 24 hours) for existing resources, so trigger scans after assignment changes.

### Q12. Explain Private Endpoints and the most common mistake with them. (Mid-level)

**Answer.** A Private Endpoint is a NIC in your subnet with a private IP mapped to one PaaS resource sub-resource (blob, vault, registry). Traffic stays on the Microsoft backbone and the service can disable public access. Resolution works because the service's public FQDN gets a CNAME to a `privatelink.*` name, which a Private DNS zone linked to the VNet resolves to the private IP. The most common mistake is creating the endpoint without the DNS zone group or without linking the zone to the client VNet, so clients still resolve the public IP and are blocked by the firewall.

### Q13. How does AKS Workload Identity work end to end? (Mid-level)

**Answer.** The cluster has an OIDC issuer URL. A user-assigned MI gets a federated credential with that issuer and subject `system:serviceaccount:<ns>:<sa>`. The ServiceAccount is annotated with `azure.workload.identity/client-id`, and pods carry the label `azure.workload.identity/use: "true"`. A mutating webhook projects a service account token and injects `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_FEDERATED_TOKEN_FILE`, and `AZURE_AUTHORITY_HOST`. `DefaultAzureCredential` exchanges the projected token for an Entra access token. It is the same pattern as EKS IRSA.

### Q14. Azure CNI Overlay versus kubenet versus Azure CNI pod subnet: how do you choose? (Mid-level)

**Answer.** Overlay gives pods a private CIDR with NAT at the node, supports up to 5,000 nodes and 250 pods per node, conserves VNet IPs, and is the default recommendation. kubenet is legacy with a 400-node route table limit and is being retired. Azure CNI with pod subnet gives pods real VNet IPs, which you need when pods must be directly addressable (NSG rules on pods, no SNAT to a database allow-list). Add the Cilium dataplane on Overlay or pod subnet for eBPF performance and network policy.

### Q15. Shared Access Signatures versus RBAC for Blob access. (Mid-level)

**Answer.** RBAC data roles (`Storage Blob Data Reader`) grant identity-based, auditable, revocable access and should be the default, with `allowSharedKeyAccess=false` to enforce it. SAS tokens are bearer tokens: account or service SAS are signed by the account key and only revocable by rotating the key, so avoid them. User delegation SAS are signed by an Entra-issued delegation key, scoped and time-bounded, and are the acceptable choice when a client (such as a browser upload) cannot do Entra auth.

### Q16. What is KQL and give an example of a security-relevant query. (Mid-level)

**Answer.** Kusto Query Language is the pipe-based query language for Log Analytics, Sentinel, and Resource Graph. Example: `AzureActivity | where OperationNameValue =~ "Microsoft.Authorization/roleAssignments/write" | where ActivityStatusValue == "Success" | project TimeGenerated, Caller, _ResourceId` finds new role assignments. The idiom is time filter first, then `where`, `summarize ... by`, `project`, with `join` across tables and `render` for charts.

### Q17. How do Blob access tiers and lifecycle management map to S3? (Mid-level)

**Answer.** Hot, Cool, Cold, and Archive map to S3 Standard, Infrequent Access, and Glacier-style classes, with minimum retention periods (30, 90, 180 days) and early-deletion fees. Archive is offline and needs rehydration (Standard priority up to 15 hours, High under an hour for small blobs). Lifecycle management rules are JSON with `tierToCool`, `tierToArchive`, and `delete` actions on base blobs, versions, and snapshots, filtered by prefix and blob index tags.

## Senior

### Q18. Design the identity model for a production AKS workload with zero stored secrets. (Senior)

**Answer.** Four identities: a GitHub app registration with federated credentials per environment and a custom role allowing ACR push and AKS credential listing; a cluster user-assigned MI with Network Contributor on the subnet and Private DNS Zone Contributor; a kubelet MI with `AcrPull`; and one workload MI per service with federated credentials to its ServiceAccount, granted `Key Vault Secrets User`, `Storage Blob Data Reader`, and a PostgreSQL Entra database role. Cluster uses Entra integration with Azure RBAC for Kubernetes and local accounts disabled. Secrets reach pods via the Key Vault CSI driver or the SDK. Every identity is auditable in Key Vault audit logs by `appid`.

### Q19. Hub-spoke with VNet peering or Virtual WAN? (Senior)

**Answer.** Hub-spoke with peering is DIY: you own the hub VNet, Azure Firewall, gateways, and UDRs, peering is non-transitive so spoke-to-spoke goes via the firewall, and it is cheaper and more flexible for one or two regions and tens of spokes. Virtual WAN is managed: Microsoft runs the hub, routing is transitive, branch and multi-region connectivity scale better, and Secured Virtual Hub integrates Azure Firewall. Choose vWAN when you have many regions or many branch sites; otherwise hub-spoke. The AWS analogue is DIY transit VPC versus Transit Gateway.

### Q20. How would you enforce "no public network access on PaaS" across 200 subscriptions? (Senior)

**Answer.** Assign at the Landing Zones MG a custom initiative with `Deny` policies on `publicNetworkAccess` for Storage, Key Vault, ACR, PostgreSQL, Cosmos, SQL, and `DeployIfNotExists` policies that create Private Endpoint DNS zone groups pointing at centrally owned Private DNS zones in the Connectivity subscription. Use `Audit` first for a month, review compliance via Resource Graph, grant time-boxed exemptions for exceptions, then flip to `Deny`. Pair with Defender CSPM to catch drift and with Terraform modules that comply by default so teams rarely hit the deny.

### Q21. Explain the AKS production egress problem with Azure Firewall. (Senior)

**Answer.** With `--outbound-type userDefinedRouting` and a 0.0.0.0/0 UDR to the firewall, the cluster's required egress must be explicitly allowed: the `AzureKubernetesService` FQDN tag (API server, MCR, telemetry), `management.azure.com`, `login.microsoftonline.com`, package mirrors for node image updates, NTP (UDP 123), and your own registries (ideally via Private Endpoints so they bypass the firewall). Missing rules show as nodes `NotReady` or `ImagePullBackOff` with timeouts; the diagnosis is the firewall's `AZFWApplicationRule` deny logs. Document the allow-list in the ADR.

### Q22. PostgreSQL Flexible Server HA and DR design. (Senior)

**Answer.** Zone-redundant HA gives a synchronous standby in another zone with automatic failover (RPO zero, RTO about a minute) at double compute cost. Backups with 35-day retention and geo-redundant backup enable PITR and geo-restore to the paired region (RPO up to an hour). A read replica in the paired region can be promoted for warm standby. Combine with Entra-only auth, Private Endpoint or VNet injection with a linked DNS zone, `pgaudit` to Log Analytics, and Premium SSD v2 for independent IOPS. Test failover with `az postgres flexible-server restart --failover Forced`.

### Q23. Defender for Cloud versus Security Hub plus GuardDuty. (Senior)

**Answer.** Defender for Cloud combines CSPM (Secure Score, regulatory compliance dashboards for CIS, NIST, PCI), workload protection plans (Servers, Containers with registry scanning and runtime sensor, Storage malware scanning, Key Vault, Databases, APIs), attack path analysis, and multi-cloud connectors for AWS and GCP. AWS splits these across Security Hub, GuardDuty, Inspector, and Macie. Defender's foundational CSPM is free; plans are priced per resource. For the capstone, Defender becomes the single CSPM pane for all three clouds.

### Q24. How do you troubleshoot `RequestDisallowedByPolicy` in a pipeline? (Senior)

**Answer.** The error body names the policy assignment and definition. Run `az policy assignment show` to read its parameters and scope, then decide whether the request should comply (most cases: add the tag, pick the allowed SKU, disable public access) or whether an exemption is warranted (`az policy exemption create` with an expiry and a ticket reference). Never fix it by removing the assignment. Add `az deployment group what-if` or a Terraform plan step with Checkov so policy violations surface before apply.

## Principal

### Q25. You inherit a tenant with 300 subscriptions, standing Owner assignments, and client secrets everywhere. Lay out a 90-day remediation plan. (Principal)

**Answer.** Days 1-15: inventory with Resource Graph and Entra workbooks (role assignments by scope and principal type, app registrations with secrets and expiry, PIM status), export to a tracker, enable Entra sign-in and audit log export, enable Defender foundational CSPM everywhere, create break-glass accounts. Days 16-45: build the MG hierarchy, move subscriptions, assign `Audit` policies for the target baseline, convert Owner assignments to PIM-eligible group assignments, require MFA via Conditional Access for Azure Management, start replacing client secrets with federated credentials in CI (highest-risk first), set secret expiry alerts. Days 46-90: flip key policies to `Deny`, disable shared key on storage and migrate Key Vaults to RBAC, deploy diagnostic settings by policy, stand up Sentinel with identity analytics, produce a compliance report and ADRs. Measure with Secure Score, count of standing privileged assignments, and count of app secrets.

### Q26. Argue for and against Container Apps versus AKS for the capstone API. (Principal)

**Answer.** Container Apps: managed Kubernetes you do not operate, KEDA scale-to-zero, Dapr, revisions and traffic splitting, managed identity and Key Vault references, VNet integration, lower operational cost and no upgrades to manage; limitations are no Kubernetes API, no CRDs or DaemonSets, limited control over networking and sidecars, and ingress is opinionated. AKS: full Kubernetes with Workload Identity, Cilium policies, GitOps with Flux, custom admission control, GPU node pools for Phase 6 vLLM; cost is cluster operations. For the capstone the learning objective (Kubernetes anchor in Cluster 4, GPU in Cluster 10) dictates AKS, and I would record in an ADR that a team without those needs should choose Container Apps.

### Q27. Design cost governance for AKS across dev, staging, and prod. (Principal)

**Answer.** Separate subscriptions per environment under different MGs with Dev/Test pricing on non-production. Tag inheritance policy and required-tag deny on RGs. AKS cost analysis add-on for namespace showback feeding Cost Management, with exports in FOCUS format to Storage for a FinOps dashboard. Dev clusters stopped nightly via scheduled GitHub Actions calling `az aks stop`, Spot node pools for batch, Node Auto Provisioning for right-sizing, VPA recommendations reviewed monthly. Prod baseline covered by a Compute Savings Plan at roughly 70% of steady-state usage, reservations for PostgreSQL and Cosmos. Budgets with Action Groups paging at forecast 100%. Log Analytics on commitment tier with Basic plan tables for container logs.

### Q28. Where is Azure architecturally weaker than AWS, and how do you design around it? (Principal)

**Answer.** Regional capacity and quota friction (`SkuNotAvailable`) is more common; mitigate with multi-size node pools, capacity reservations for critical pools, and paired-region readiness. Two authorisation planes on Storage and Key Vault create audit gaps; mitigate by disabling the legacy plane with policy. Private Endpoint DNS is complex, especially hybrid; centralise zones and use Private DNS Resolver. Global uniqueness of PaaS names leaks tenancy information; use unpredictable suffixes. The CLI is slow for long-running operations; use `--no-wait` and GitOps. Entra as a single tenant-wide identity plane concentrates risk; enforce PIM, Conditional Access, and tenant restrictions. Honest comparison: Azure's hierarchy, Policy, and Managed Identity model are stronger than AWS Organizations plus IAM for enterprise governance.

### Q29. A pod reads a Key Vault secret successfully in dev but gets 403 in prod with identical manifests. Diagnose. (Principal)

**Answer.** Identical manifests means the Kubernetes side is fine; the difference is in Azure. Check in order: (1) the prod workload MI has a federated credential whose issuer is the prod cluster's OIDC URL (each cluster has its own); (2) the MI has `Key Vault Secrets User` on the prod vault specifically, allowing for propagation delay; (3) the prod vault has `publicNetworkAccess: Disabled` and a Private Endpoint whose `privatelink.vaultcore.azure.net` zone is linked to the prod spoke VNet (a 403 with "not from a trusted service nor via an approved private link" confirms this); (4) prod vault uses RBAC while dev uses access policies or vice versa; (5) an Azure Policy or Conditional Access difference. Verify with `KeyVaultAuditLogs | where ResultSignature != "OK"` filtered by the MI's `appid`.

### Q30. How would you present the Azure landing zone decision to an architecture review board? (Principal)

**Answer.** Frame it as the roadmap frames it: four primitives (identity, network, compute, storage) with vocabulary translation from the AWS baseline the board already approved. Show the MG hierarchy and the Policy initiative list with effects, the identity model (PIM, federated credentials, managed identities, no secrets), the hub-spoke diagram with Azure Firewall egress and Private Endpoints, the AKS baseline flags, data protection settings per service, the observability and alerting design in one workspace, cost controls, and DR tiers with RPO/RTO. Attach the ADRs for hub-spoke versus vWAN, Overlay versus pod subnet, AKS versus Container Apps, and Terraform versus Bicep. Close with the verification evidence: Resource Graph compliance queries and Secure Score, not slides.

## Rapid-Fire Vocabulary Round

| They say (AWS) | You say (Azure) |
|---|---|
| Account | Subscription |
| OU / SCP | Management Group / Azure Policy |
| IAM role for a workload | Managed Identity |
| IRSA | AKS Workload Identity |
| Instance profile | System-assigned managed identity on a VM |
| Interface endpoint | Private Endpoint |
| Security group | NSG (plus ASG) |
| Transit Gateway | Virtual WAN (or hub-spoke peering) |
| ECR | ACR |
| EKS managed node group | AKS node pool |
| CloudTrail | Activity Log plus diagnostic settings |
| CloudWatch Logs Insights | Log Analytics with KQL |
| KMS + Secrets Manager | Key Vault |
| S3 Glacier | Blob Archive tier |
| S3 Object Lock | Immutable blob storage |
| Presigned URL | SAS (user delegation) |
| CloudFormation | ARM templates / Bicep |
| Security Hub + GuardDuty | Defender for Cloud |
| ALB / NLB / CloudFront | Application Gateway / Load Balancer / Front Door |
| `aws sts get-caller-identity` | `az account show` + `az ad signed-in-user show` |
