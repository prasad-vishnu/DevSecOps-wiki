# Azure: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 01 Hyperscale Cloud |
| **Role** | Bridge (anchor is AWS) |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | Azure CLI 2.60+, Bicep 0.28+ |
| **Capstone phase** | Phase 2 (Months 5-8) |

> An `az` reference organised by task, with the `aws` equivalent noted where it helps. Every command has a one-line explanation. Long-form flags are used; the common short forms are `-g` (`--resource-group`), `-n` (`--name`), `-l` (`--location`), `-o` (`--output`), `-s` (`--subscription`). Placeholders: `<SUBSCRIPTION_ID>`, `<TENANT_ID>`, `<APP_ID>`, `<OBJECT_ID>`, `<MI_CLIENT_ID>`, `rg-capstone-dev`, `eastus2`.

## 1. CLI Mechanics

| Task | Command | Notes |
|---|---|---|
| Discover subcommands | `az aks --help`, `az find "aks create"` | `az find` searches examples |
| Interactive mode | `az interactive` | Autocomplete shell |
| Output formats | `--output json\|jsonc\|table\|tsv\|yaml\|none` | `tsv` for shell variables |
| JMESPath filter | `--query "[?location=='eastus2'].{name:name, rg:resourceGroup}"` | Same syntax as `aws --query` |
| Verbose / debug | `--verbose`, `--debug` | `--debug` shows the ARM REST calls |
| Async | `--no-wait` then `az <res> wait --created --name ... --resource-group ...` | Required for scripting long operations |
| Raw ARM call | `az rest --method get --url "https://management.azure.com/subscriptions/<SUBSCRIPTION_ID>/resourceGroups?api-version=2021-04-01"` | Any API not wrapped by a command |
| Generic resource ops | `az resource list --resource-group rg-capstone-dev --output table`, `az resource show --ids <RESOURCE_ID>`, `az resource tag --ids <RESOURCE_ID> --tags env=dev`, `az resource delete --ids <RESOURCE_ID>` | Works for any provider |
| Defaults | `az config set defaults.group=rg-capstone-dev defaults.location=eastus2` | Then omit `--resource-group`/`--location` |
| Current context | `az account show`, `az account list --output table`, `az account set --subscription <SUBSCRIPTION_ID>` | `aws sts get-caller-identity` |
| Login variants | `az login`, `az login --use-device-code`, `az login --tenant <TENANT_ID>`, `az login --identity`, `az login --service-principal --username <APP_ID> --tenant <TENANT_ID> --federated-token "$TOKEN"` | |
| Get a token | `az account get-access-token --resource https://management.azure.com --query accessToken --output tsv` | Scopes: `https://vault.azure.net`, `https://storage.azure.com`, `https://ossrdbms-aad.database.windows.net` |
| Extensions | `az extension list --output table`, `az extension add --name containerapp`, `az extension update --name containerapp`, `az extension remove --name aks-preview` | |
| Upgrade | `az upgrade --yes` | Also updates extensions with `--all` |

Useful JMESPath idioms:

```bash
az vm list --query "[].{name:name, size:hardwareProfile.vmSize, rg:resourceGroup}" --output table   # projection
az vm list --query "[?tags.environment=='prod'].name" --output tsv                                   # filter on tag
az aks list --query "length(@)"                                                                      # count
az network nsg rule list --nsg-name nsg-app --query "sort_by(@, &priority)[].{p:priority, n:name, a:access}" --output table  # sort
az storage account list --query "[?allowSharedKeyAccess!=\`false\`].name" --output tsv               # backtick literal for booleans
```

## 2. Hierarchy and Governance

| Task | Command |
|---|---|
| List management groups | `az account management-group list --output table` |
| Create MG under root | `az account management-group create --name mg-landing-zones --display-name "Landing Zones"` |
| Nest an MG | `az account management-group create --name mg-corp --parent mg-landing-zones` |
| Move subscription into MG | `az account management-group subscription add --name mg-corp --subscription <SUBSCRIPTION_ID>` |
| Show MG tree | `az account management-group show --name mg-landing-zones --expand --recurse` |
| List subscriptions | `az account subscription list --output table` (extension `account`) |
| Create subscription (EA/MCA) | `az account alias create --name capstone-dev --billing-scope "/providers/Microsoft.Billing/billingAccounts/<BA>/billingProfiles/<BP>/invoiceSections/<IS>" --display-name "Capstone Dev" --workload Production` |
| List locations | `az account list-locations --query "[?metadata.regionCategory=='Recommended'].{name:name, pair:metadata.pairedRegion[0].name}" --output table` |
| Check quotas | `az vm list-usage --location eastus2 --query "[?currentValue>0].{name:name.value, used:currentValue, limit:limit}" --output table` |
| Request quota increase | `az quota create --resource-name standardDSv5Family --scope "/subscriptions/<SUBSCRIPTION_ID>/providers/Microsoft.Compute/locations/eastus2" --limit-object value=100 --resource-type dedicated` (extension `quota`) |
| Register provider | `az provider register --namespace Microsoft.ContainerService`; check with `az provider show --namespace Microsoft.ContainerService --query registrationState` |
| Register preview feature | `az feature register --namespace Microsoft.ContainerService --name <FeatureName>`; then re-register the provider |

### Resource Groups and locks

```bash
az group create --name rg-capstone-dev --location eastus2 --tags env=dev owner=platform      # create (aws: no equivalent)
az group list --query "[?tags.env=='dev'].name" --output tsv                                 # filter by tag
az group show --name rg-capstone-dev                                                         # details
az group export --name rg-capstone-dev > rg.json                                             # export ARM template of existing resources
az group delete --name rg-capstone-dev --yes --no-wait                                       # delete everything inside
az lock create --name no-delete --lock-type CanNotDelete --resource-group rg-capstone-data   # prevent deletion (even by Owner)
az lock list --resource-group rg-capstone-data --output table
az lock delete --name no-delete --resource-group rg-capstone-data
az resource move --destination-group rg-new --ids <RESOURCE_ID_1> <RESOURCE_ID_2>            # move between RGs (service-dependent)
```

### Tags

```bash
az tag create --resource-id <RESOURCE_ID> --tags env=dev costCenter=1234     # replace all tags
az tag update --resource-id <RESOURCE_ID> --operation Merge --tags owner=sre  # merge
az tag update --resource-id <RESOURCE_ID> --operation Delete --tags owner=sre # remove one
az tag list --resource-id <RESOURCE_ID>
```

### Azure Policy

```bash
# Built-in definitions
az policy definition list --query "[?policyType=='BuiltIn'] | [?contains(displayName,'Allowed locations')].{name:name, display:displayName}" --output table
az policy set-definition list --query "[?contains(displayName,'CIS')].{name:name, display:displayName}" --output table

# Custom definition from file (rules and params as JSON)
az policy definition create --name deny-public-ip --display-name "Deny public IP addresses" \
  --mode Indexed --rules deny-public-ip.rules.json --params deny-public-ip.params.json \
  --management-group mg-landing-zones

# Assign at subscription with parameters
az policy assignment create --name allowed-locations \
  --policy "e56962a6-4747-49cd-b67b-bf8b01975c4c" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>" \
  --params '{"listOfAllowedLocations":{"value":["eastus2","centralus"]}}'

# Assign a DeployIfNotExists initiative with a managed identity for remediation
az policy assignment create --name deploy-diagnostics \
  --policy-set-definition "<INITIATIVE_ID>" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>" \
  --mi-system-assigned --location eastus2 --role Contributor --identity-scope "/subscriptions/<SUBSCRIPTION_ID>"

# Compliance and remediation
az policy state summarize --subscription <SUBSCRIPTION_ID>
az policy state list --resource-group rg-capstone-dev --filter "complianceState eq 'NonCompliant'" --query "[].{res:resourceId, policy:policyDefinitionName}" --output table
az policy state trigger-scan --resource-group rg-capstone-dev          # force evaluation (otherwise up to 24 h)
az policy remediation create --name fix-diag --policy-assignment deploy-diagnostics --resource-group rg-capstone-dev
az policy exemption create --name temp-exempt --policy-assignment allowed-locations --exemption-category Waiver --expires-on 2026-12-31T00:00:00Z --resource-group rg-sandbox
```

### Resource Graph (instant cross-subscription inventory)

```bash
az graph query -q "Resources | where type =~ 'microsoft.compute/virtualmachines' | project name, location, properties.hardwareProfile.vmSize" --output table
az graph query -q "Resources | where type =~ 'microsoft.storage/storageaccounts' and properties.allowBlobPublicAccess == true | project name, resourceGroup"
az graph query -q "PolicyResources | where type =~ 'microsoft.policyinsights/policystates' | summarize count() by tostring(properties.complianceState)"
```

## 3. Identity: Entra ID and RBAC

| Task | Command | AWS analogue |
|---|---|---|
| Who am I | `az ad signed-in-user show --query "{upn:userPrincipalName, id:id}"` | `aws sts get-caller-identity` |
| Find a user | `az ad user show --id user@example.com --query id --output tsv` | |
| Create group | `az ad group create --display-name sg-platform-admins --mail-nickname sg-platform-admins` | `aws iam create-group` |
| Add member | `az ad group member add --group sg-platform-admins --member-id <OBJECT_ID>` | |
| Create app registration | `az ad app create --display-name gh-capstone-deploy --query appId --output tsv` | OIDC provider + role |
| Create SP for app | `az ad sp create --id <APP_ID>` | |
| App + SP + role in one (creates a secret unless `--create-password false`... avoid secrets) | `az ad sp create-for-rbac --name sp-capstone --role Contributor --scopes "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/rg-capstone-dev"` | `aws iam create-user` + access key (legacy) |
| Add federated credential (GitHub) | `az ad app federated-credential create --id <APP_ID> --parameters fc.json` | Trust policy `Condition` |
| List federated credentials | `az ad app federated-credential list --id <APP_ID>` | |
| Delete SP and app | `az ad sp delete --id <APP_ID>`; `az ad app delete --id <APP_ID>` | |
| Create user-assigned MI | `az identity create --name id-capstone-api --resource-group rg-capstone-dev --location eastus2` | `aws iam create-role` |
| Show MI IDs | `az identity show --name id-capstone-api --resource-group rg-capstone-dev --query "{clientId:clientId, principalId:principalId, id:id}"` | |
| Federate MI with AKS SA | `az identity federated-credential create --name aks-api --identity-name id-capstone-api --resource-group rg-capstone-dev --issuer "<AKS_OIDC_ISSUER>" --subject "system:serviceaccount:capstone:api-sa" --audiences api://AzureADTokenExchange` | IRSA trust |

### Role assignments

```bash
# Built-in roles lookup
az role definition list --query "[?roleType=='BuiltIn'] | [?contains(roleName,'Key Vault')].{name:roleName, id:name}" --output table
az role definition list --name "Storage Blob Data Reader" --query "[0].permissions[0].dataActions"

# Assign (prefer --assignee-object-id + --assignee-principal-type to avoid Graph lookups in CI)
az role assignment create --role "Key Vault Secrets User" \
  --assignee-object-id <MI_PRINCIPAL_ID> --assignee-principal-type ServicePrincipal \
  --scope "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/rg-capstone-dev/providers/Microsoft.KeyVault/vaults/kv-capstone-dev"

# Assign with ABAC condition (only blobs in containers tagged project=capstone)
az role assignment create --role "Storage Blob Data Reader" --assignee-object-id <OBJECT_ID> --assignee-principal-type Group \
  --scope "<STORAGE_ACCOUNT_ID>" \
  --condition "((!(ActionMatches{'Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read'})) OR (@Resource[Microsoft.Storage/storageAccounts/blobServices/containers:name] StringEquals 'capstone'))" \
  --condition-version "2.0"

# Inspect
az role assignment list --assignee <OBJECT_ID> --all --output table                      # everything, all scopes
az role assignment list --scope "/subscriptions/<SUBSCRIPTION_ID>" --include-inherited --output table
az role assignment list --resource-group rg-capstone-dev --include-inherited --include-groups --assignee user@example.com --output table

# Remove
az role assignment delete --assignee <OBJECT_ID> --role "Contributor" --scope "/subscriptions/<SUBSCRIPTION_ID>"

# Custom role
az role definition create --role-definition custom-role.json
az role definition update --role-definition custom-role.json
az role definition delete --name "Capstone AKS Deployer"
```

### PIM (eligible assignments; requires P2; most ops are via `az rest` or portal)

```bash
# List eligible Azure role assignments for the signed-in user
az rest --method get --url "https://management.azure.com/subscriptions/<SUBSCRIPTION_ID>/providers/Microsoft.Authorization/roleEligibilityScheduleInstances?api-version=2020-10-01&\$filter=asTarget()"
```

## 4. Networking

```bash
# VNet and subnets
az network vnet create --name vnet-spoke --resource-group rg-net --location eastus2 --address-prefixes 10.1.0.0/16 \
  --subnet-name snet-aks --subnet-prefixes 10.1.0.0/22
az network vnet subnet create --name snet-pe --vnet-name vnet-spoke --resource-group rg-net --address-prefixes 10.1.4.0/24 \
  --private-endpoint-network-policies Disabled
az network vnet subnet create --name snet-pg --vnet-name vnet-spoke --resource-group rg-net --address-prefixes 10.1.5.0/24 \
  --delegations Microsoft.DBforPostgreSQL/flexibleServers
az network vnet subnet list --vnet-name vnet-spoke --resource-group rg-net --output table
az network vnet update --name vnet-spoke --resource-group rg-net --address-prefixes 10.1.0.0/16 10.2.0.0/16   # add address space in place

# NSG, rules, ASG
az network nsg create --name nsg-aks --resource-group rg-net --location eastus2
az network nsg rule create --nsg-name nsg-aks --resource-group rg-net --name deny-internet-inbound --priority 4000 \
  --direction Inbound --access Deny --protocol '*' --source-address-prefixes Internet --destination-port-ranges '*'
az network nsg rule create --nsg-name nsg-aks --resource-group rg-net --name allow-agw --priority 200 \
  --direction Inbound --access Allow --protocol Tcp --source-address-prefixes 10.1.6.0/24 --destination-port-ranges 80 443
az network nsg rule list --nsg-name nsg-aks --resource-group rg-net --include-default --output table
az network vnet subnet update --name snet-aks --vnet-name vnet-spoke --resource-group rg-net --network-security-group nsg-aks
az network asg create --name asg-web --resource-group rg-net --location eastus2
az network nic ip-config update --name ipconfig1 --nic-name nic-web01 --resource-group rg-net --application-security-groups asg-web

# Route table and UDR
az network route-table create --name rt-spoke --resource-group rg-net --location eastus2 --disable-bgp-route-propagation true
az network route-table route create --route-table-name rt-spoke --resource-group rg-net --name default-to-fw \
  --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.1.4
az network vnet subnet update --name snet-aks --vnet-name vnet-spoke --resource-group rg-net --route-table rt-spoke
az network nic show-effective-route-table --name nic-web01 --resource-group rg-net --output table    # what a NIC actually sees

# NAT Gateway
az network public-ip create --name pip-nat --resource-group rg-net --sku Standard --zone 1 --location eastus2
az network nat gateway create --name nat-spoke --resource-group rg-net --public-ip-addresses pip-nat --idle-timeout 10 --location eastus2
az network vnet subnet update --name snet-aks --vnet-name vnet-spoke --resource-group rg-net --nat-gateway nat-spoke

# Peering (both directions)
az network vnet peering create --name hub-to-spoke --vnet-name vnet-hub --resource-group rg-hub \
  --remote-vnet "<SPOKE_VNET_ID>" --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
az network vnet peering create --name spoke-to-hub --vnet-name vnet-spoke --resource-group rg-net \
  --remote-vnet "<HUB_VNET_ID>" --allow-vnet-access --allow-forwarded-traffic --use-remote-gateways
az network vnet peering list --vnet-name vnet-spoke --resource-group rg-net --query "[].{name:name, state:peeringState}" --output table

# Private DNS zone + link (needed for every Private Endpoint type)
az network private-dns zone create --name privatelink.vaultcore.azure.net --resource-group rg-net
az network private-dns link vnet create --zone-name privatelink.vaultcore.azure.net --resource-group rg-net \
  --name link-spoke --virtual-network "<SPOKE_VNET_ID>" --registration-enabled false

# Private Endpoint for Key Vault with DNS zone group (auto-creates the A record)
az network private-endpoint create --name pe-kv --resource-group rg-net --vnet-name vnet-spoke --subnet snet-pe \
  --private-connection-resource-id "<KEY_VAULT_ID>" --group-id vault --connection-name kv-conn --location eastus2
az network private-endpoint dns-zone-group create --endpoint-name pe-kv --resource-group rg-net --name default \
  --private-dns-zone "<PRIVATE_DNS_ZONE_ID>" --zone-name vault

# Group IDs: storage: blob, file, dfs, queue, table | KV: vault | ACR: registry | PostgreSQL Flexible: postgresqlServer | AKS: management | Cosmos: Sql
az network private-link-resource list --id "<RESOURCE_ID>" --query "[].properties.groupId" --output tsv

# Azure Firewall (Standard) with policy
az network firewall policy create --name fwpol-hub --resource-group rg-hub --location eastus2 --sku Standard --threat-intel-mode Deny
az network firewall policy rule-collection-group create --name rcg-aks --policy-name fwpol-hub --resource-group rg-hub --priority 200
az network firewall policy rule-collection-group collection add-filter-collection --name aks-egress --policy-name fwpol-hub \
  --resource-group rg-hub --rule-collection-group-name rcg-aks --collection-priority 100 --action Allow \
  --rule-name aks-fqdn-tag --rule-type ApplicationRule --source-addresses 10.1.0.0/22 --fqdn-tags AzureKubernetesService --protocols Https=443
az network firewall create --name fw-hub --resource-group rg-hub --location eastus2 --sku AZFW_VNet --tier Standard \
  --firewall-policy fwpol-hub --vnet-name vnet-hub --conf-name fwcfg --public-ip pip-fw
az network firewall show --name fw-hub --resource-group rg-hub --query "ipConfigurations[0].privateIPAddress" --output tsv

# Diagnostics
az network watcher test-ip-flow --vm vm-web01 --resource-group rg-net --direction Inbound --protocol TCP --local 10.1.0.4:443 --remote 203.0.113.10:50000
az network watcher show-next-hop --vm vm-web01 --resource-group rg-net --source-ip 10.1.0.4 --dest-ip 8.8.8.8
az network watcher flow-log create --name fl-nsg-aks --resource-group rg-net --location eastus2 --nsg nsg-aks --storage-account <STORAGE_ID> --workspace <LAW_ID> --traffic-analytics true --enabled true
```

## 5. Compute: VMs and VMSS

```bash
# Images and sizes
az vm image list --publisher Canonical --offer ubuntu-24_04-lts --sku server --all --query "[-1].urn" --output tsv
az vm list-sizes --location eastus2 --query "[?numberOfCores==\`4\`].name" --output tsv
az vm list-skus --location eastus2 --size Standard_D4s_v5 --query "[].{name:name, restrictions:restrictions[].reasonCode}" --output table   # capacity restrictions

# Create VM with system-assigned MI, no public IP, SSH key, cloud-init
az vm create --name vm-bastion-target --resource-group rg-capstone-dev --location eastus2 --zone 1 \
  --image Canonical:ubuntu-24_04-lts:server:latest --size Standard_B2s \
  --vnet-name vnet-spoke --subnet snet-aks --public-ip-address "" --nsg "" \
  --admin-username azureuser --generate-ssh-keys \
  --assign-identity [system] --custom-data cloud-init.yaml \
  --os-disk-delete-option Delete --nic-delete-option Delete

az vm list --resource-group rg-capstone-dev --show-details --query "[].{name:name, power:powerState, ip:privateIps}" --output table
az vm start|stop|deallocate|restart --name vm-x --resource-group rg-capstone-dev      # stop still bills; deallocate does not
az vm run-command invoke --name vm-x --resource-group rg-capstone-dev --command-id RunShellScript --scripts "systemctl status nginx"
az vm extension set --name AzureMonitorLinuxAgent --publisher Microsoft.Azure.Monitor --vm-name vm-x --resource-group rg-capstone-dev
az vm identity assign --name vm-x --resource-group rg-capstone-dev --identities <USER_ASSIGNED_MI_ID>
az vm resize --name vm-x --resource-group rg-capstone-dev --size Standard_D4s_v5
az vm deallocate --name vm-x --resource-group rg-capstone-dev && az vm delete --name vm-x --resource-group rg-capstone-dev --yes

# Bastion (no public IPs on VMs)
az network bastion create --name bas-hub --resource-group rg-hub --vnet-name vnet-hub --public-ip-address pip-bas --sku Standard --enable-tunneling true
az network bastion ssh --name bas-hub --resource-group rg-hub --target-resource-id <VM_ID> --auth-type ssh-key --username azureuser --ssh-key ~/.ssh/id_ed25519

# VMSS Flexible with autoscale
az vmss create --name vmss-web --resource-group rg-capstone-dev --location eastus2 --orchestration-mode Flexible \
  --image Canonical:ubuntu-24_04-lts:server:latest --vm-sku Standard_D2s_v5 --instance-count 2 --zones 1 2 3 \
  --vnet-name vnet-spoke --subnet snet-aks --public-ip-address "" --load-balancer "" \
  --admin-username azureuser --generate-ssh-keys --assign-identity [system] --upgrade-policy-mode Automatic
az monitor autoscale create --resource <VMSS_ID> --resource-group rg-capstone-dev --name as-web --min-count 2 --max-count 10 --count 2
az monitor autoscale rule create --autoscale-name as-web --resource-group rg-capstone-dev --condition "Percentage CPU > 70 avg 5m" --scale out 2
az monitor autoscale rule create --autoscale-name as-web --resource-group rg-capstone-dev --condition "Percentage CPU < 30 avg 10m" --scale in 1
az vmss list-instances --name vmss-web --resource-group rg-capstone-dev --output table
az vmss scale --name vmss-web --resource-group rg-capstone-dev --new-capacity 4
```

## 6. AKS

```bash
# Versions and features
az aks get-versions --location eastus2 --output table
az aks get-upgrades --name aks-dev --resource-group rg-capstone-dev --output table

# Create: private, Azure CNI Overlay + Cilium, Workload Identity, Entra + Azure RBAC, no local accounts, KV CSI, Policy, Container Insights
az aks create --name aks-dev --resource-group rg-capstone-dev --location eastus2 \
  --tier standard --kubernetes-version 1.30 \
  --node-count 3 --node-vm-size Standard_D4ds_v5 --zones 1 2 3 --nodepool-name system --node-osdisk-type Ephemeral \
  --vnet-subnet-id "<SNET_AKS_ID>" --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium --network-policy cilium \
  --pod-cidr 192.168.0.0/16 --service-cidr 172.16.0.0/16 --dns-service-ip 172.16.0.10 \
  --outbound-type userAssignedNATGateway \
  --enable-private-cluster --private-dns-zone system \
  --enable-oidc-issuer --enable-workload-identity \
  --enable-aad --enable-azure-rbac --disable-local-accounts \
  --aad-admin-group-object-ids <ADMIN_GROUP_OBJECT_ID> \
  --assign-identity "<CLUSTER_MI_ID>" --assign-kubelet-identity "<KUBELET_MI_ID>" \
  --enable-addons azure-keyvault-secrets-provider,azure-policy,monitoring --enable-secret-rotation --rotation-poll-interval 2m \
  --workspace-resource-id "<LAW_ID>" \
  --attach-acr acrcapstonedev \
  --auto-upgrade-channel patch --node-os-upgrade-channel NodeImage \
  --enable-cluster-autoscaler --min-count 3 --max-count 6 \
  --tags env=dev

# Credentials (kubelogin handles Entra auth; no static admin creds)
az aks get-credentials --name aks-dev --resource-group rg-capstone-dev --overwrite-existing
kubelogin convert-kubeconfig --login azurecli
kubectl get nodes
az aks get-credentials --name aks-dev --resource-group rg-capstone-dev --admin     # fails if local accounts disabled (good)

# Node pools
az aks nodepool add --cluster-name aks-dev --resource-group rg-capstone-dev --name user1 --mode User \
  --node-count 2 --node-vm-size Standard_D4ds_v5 --zones 1 2 3 --enable-cluster-autoscaler --min-count 2 --max-count 20 \
  --labels workload=api --max-surge 33% --os-sku AzureLinux
az aks nodepool add --cluster-name aks-dev --resource-group rg-capstone-dev --name spot1 --mode User \
  --priority Spot --eviction-policy Delete --spot-max-price -1 --node-count 0 --enable-cluster-autoscaler --min-count 0 --max-count 10 \
  --node-taints kubernetes.azure.com/scalesetpriority=spot:NoSchedule --labels workload=batch
az aks nodepool update --cluster-name aks-dev --resource-group rg-capstone-dev --name system --node-taints CriticalAddonsOnly=true:NoSchedule
az aks nodepool list --cluster-name aks-dev --resource-group rg-capstone-dev --output table
az aks nodepool upgrade --cluster-name aks-dev --resource-group rg-capstone-dev --name user1 --kubernetes-version 1.31.1
az aks nodepool scale --cluster-name aks-dev --resource-group rg-capstone-dev --name user1 --node-count 3
az aks nodepool delete --cluster-name aks-dev --resource-group rg-capstone-dev --name spot1 --no-wait

# Cluster ops
az aks upgrade --name aks-dev --resource-group rg-capstone-dev --kubernetes-version 1.31.1 --control-plane-only
az aks show --name aks-dev --resource-group rg-capstone-dev --query "oidcIssuerProfile.issuerUrl" --output tsv     # for federated credentials
az aks show --name aks-dev --resource-group rg-capstone-dev --query "identityProfile.kubeletidentity.objectId" --output tsv
az aks update --name aks-dev --resource-group rg-capstone-dev --attach-acr acrcapstonedev
az aks enable-addons --name aks-dev --resource-group rg-capstone-dev --addons azure-keyvault-secrets-provider --enable-secret-rotation
az aks approuting enable --name aks-dev --resource-group rg-capstone-dev                                            # managed NGINX ingress
az aks update --name aks-dev --resource-group rg-capstone-dev --enable-cost-analysis
az aks update --name aks-dev --resource-group rg-capstone-dev --enable-defender
az aks maintenanceconfiguration add --cluster-name aks-dev --resource-group rg-capstone-dev --name aksManagedAutoUpgradeSchedule \
  --schedule-type Weekly --day-of-week Sunday --interval-weeks 1 --start-time 02:00 --duration 4
az aks command invoke --name aks-dev --resource-group rg-capstone-dev --command "kubectl get pods -A"   # run kubectl inside a private cluster without VPN
az aks stop --name aks-dev --resource-group rg-capstone-dev      # stop control plane + nodes (dev cost saver)
az aks start --name aks-dev --resource-group rg-capstone-dev
az aks delete --name aks-dev --resource-group rg-capstone-dev --yes --no-wait

# Azure RBAC for Kubernetes: namespace-scoped
az role assignment create --role "Azure Kubernetes Service RBAC Writer" --assignee-object-id <DEV_GROUP_ID> --assignee-principal-type Group \
  --scope "<AKS_ID>/namespaces/capstone"
```

## 7. Container Registry

```bash
az acr create --name acrcapstonedev --resource-group rg-capstone-dev --sku Premium --location eastus2 \
  --admin-enabled false --public-network-enabled false --zone-redundancy Enabled
az acr login --name acrcapstonedev                                   # docker login via Entra token (aws ecr get-login-password)
az acr login --name acrcapstonedev --expose-token                    # token for non-Docker clients
az acr build --registry acrcapstonedev --image api/fastapi:1.0.0 --file Dockerfile .      # cloud build (no local Docker)
az acr import --name acrcapstonedev --source docker.io/library/nginx:1.27 --image mirror/nginx:1.27   # mirror public image
az acr repository list --name acrcapstonedev --output table
az acr repository show-tags --name acrcapstonedev --repository api/fastapi --orderby time_desc --output table
az acr manifest list-metadata --registry acrcapstonedev --name api/fastapi --query "[].{digest:digest, tags:tags}" --output table
az acr repository delete --name acrcapstonedev --image api/fastapi:0.9.0 --yes
az acr replication create --registry acrcapstonedev --location westus3                    # geo-replication (Premium)
az acr update --name acrcapstonedev --default-action Deny                                  # firewall
az acr network-rule add --name acrcapstonedev --ip-address 203.0.113.0/24
az acr config retention update --registry acrcapstonedev --status Enabled --days 7 --type UntaggedManifests
az acr cache create --registry acrcapstonedev --name dockerhub-nginx --source-repo docker.io/library/nginx --target-repo cache/nginx
az acr check-health --name acrcapstonedev --yes
az acr show-usage --name acrcapstonedev --output table
az acr task create --registry acrcapstonedev --name build-api --image api/fastapi:{{.Run.ID}} --context https://github.com/example-org/capstone.git#main:services/api --file Dockerfile --git-access-token <PAT_IN_KV>   # CI in ACR (prefer GitHub Actions)
```

## 8. App Service, Container Apps, Functions

```bash
# App Service
az appservice plan create --name asp-capstone --resource-group rg-capstone-dev --location eastus2 --is-linux --sku P1v3 --zone-redundant
az webapp create --name app-capstone-java --resource-group rg-capstone-dev --plan asp-capstone --container-image-name acrcapstonedev.azurecr.io/api/spring:1.0.0 --assign-identity [system] --acr-use-identity
az webapp config appsettings set --name app-capstone-java --resource-group rg-capstone-dev --settings SPRING_PROFILES_ACTIVE=prod DB_PASSWORD="@Microsoft.KeyVault(SecretUri=https://kv-capstone-dev.vault.azure.net/secrets/db-password/)"
az webapp deployment slot create --name app-capstone-java --resource-group rg-capstone-dev --slot staging --configuration-source app-capstone-java
az webapp deployment slot swap --name app-capstone-java --resource-group rg-capstone-dev --slot staging --target-slot production
az webapp vnet-integration add --name app-capstone-java --resource-group rg-capstone-dev --vnet vnet-spoke --subnet snet-appsvc
az webapp log tail --name app-capstone-java --resource-group rg-capstone-dev
az webapp auth update --name app-capstone-java --resource-group rg-capstone-dev --enabled true --action RedirectToLoginPage   # Easy Auth

# Container Apps
az containerapp env create --name cae-capstone --resource-group rg-capstone-dev --location eastus2 --logs-workspace-id <LAW_CUSTOMER_ID> --logs-workspace-key <LAW_KEY> --infrastructure-subnet-resource-id <SNET_CAE_ID> --internal-only false
az containerapp create --name ca-api --resource-group rg-capstone-dev --environment cae-capstone \
  --image acrcapstonedev.azurecr.io/api/fastapi:1.0.0 --registry-server acrcapstonedev.azurecr.io --registry-identity <MI_ID> \
  --user-assigned <MI_ID> --target-port 8000 --ingress external --min-replicas 0 --max-replicas 10 \
  --cpu 0.5 --memory 1.0Gi --env-vars AZURE_CLIENT_ID=<MI_CLIENT_ID> --secrets db-url=keyvaultref:https://kv-capstone-dev.vault.azure.net/secrets/db-url,identityref:<MI_ID>
az containerapp update --name ca-api --resource-group rg-capstone-dev --image acrcapstonedev.azurecr.io/api/fastapi:1.1.0
az containerapp revision list --name ca-api --resource-group rg-capstone-dev --output table
az containerapp ingress traffic set --name ca-api --resource-group rg-capstone-dev --revision-weight ca-api--rev1=80 ca-api--rev2=20
az containerapp logs show --name ca-api --resource-group rg-capstone-dev --follow
az containerapp job create --name job-migrate --resource-group rg-capstone-dev --environment cae-capstone --trigger-type Manual --image acrcapstonedev.azurecr.io/api/migrate:1.0.0 --replica-timeout 600

# Functions (Flex Consumption, Python 3.12, MI to storage, no connection string)
az storage account create --name stfunccapstone001 --resource-group rg-capstone-dev --location eastus2 --sku Standard_ZRS --allow-shared-key-access false
az functionapp create --name func-capstone --resource-group rg-capstone-dev --flexconsumption-location eastus2 --runtime python --runtime-version 3.12 --storage-account stfunccapstone001 --assign-identity [system] --deployment-storage-auth-type SystemAssignedIdentity
az functionapp config appsettings set --name func-capstone --resource-group rg-capstone-dev --settings "AzureWebJobsStorage__accountName=stfunccapstone001"
az functionapp deployment source config-zip --name func-capstone --resource-group rg-capstone-dev --src app.zip
az functionapp function list --name func-capstone --resource-group rg-capstone-dev --output table
az functionapp log tail --name func-capstone --resource-group rg-capstone-dev
```

## 9. Storage

```bash
# Account: ZRS, no public blob access, no shared key, TLS 1.2, no public network, versioning + soft delete
az storage account create --name stcapstonedev001 --resource-group rg-capstone-dev --location eastus2 \
  --sku Standard_ZRS --kind StorageV2 --access-tier Hot --min-tls-version TLS1_2 --https-only true \
  --allow-blob-public-access false --allow-shared-key-access false --public-network-access Disabled --default-action Deny
az storage account blob-service-properties update --account-name stcapstonedev001 --resource-group rg-capstone-dev \
  --enable-versioning true --enable-delete-retention true --delete-retention-days 14 --enable-container-delete-retention true --container-delete-retention-days 14 --enable-change-feed true

# Data plane with Entra auth (--auth-mode login); needs Storage Blob Data * role
az storage container create --name backups --account-name stcapstonedev001 --auth-mode login
az storage blob upload --account-name stcapstonedev001 --container-name backups --name db/2026-10-27.dump --file ./dump.sql --auth-mode login --tier Cool
az storage blob list --account-name stcapstonedev001 --container-name backups --auth-mode login --output table
az storage blob download --account-name stcapstonedev001 --container-name backups --name db/2026-10-27.dump --file ./restore.sql --auth-mode login
az storage blob set-tier --account-name stcapstonedev001 --container-name backups --name db/2026-10-27.dump --tier Archive --auth-mode login
az storage blob set-tier --account-name stcapstonedev001 --container-name backups --name db/2026-10-27.dump --tier Hot --rehydrate-priority High --auth-mode login
az storage blob copy start-batch --source-container backups --destination-container backups-copy --account-name stcapstonedev001 --auth-mode login
az storage blob delete --account-name stcapstonedev001 --container-name backups --name db/old.dump --auth-mode login
az storage blob undelete --account-name stcapstonedev001 --container-name backups --name db/old.dump --auth-mode login
azcopy copy ./localdir "https://stcapstonedev001.blob.core.windows.net/backups/" --recursive   # bulk; azcopy login uses Entra

# User delegation SAS (signed by Entra, not account key), 1 hour read
az storage blob generate-sas --account-name stcapstonedev001 --container-name backups --name db/2026-10-27.dump \
  --permissions r --expiry "$(date -u -d '+1 hour' +%Y-%m-%dT%H:%MZ)" --auth-mode login --as-user --https-only --full-uri

# Lifecycle, immutability, network
az storage account management-policy create --account-name stcapstonedev001 --resource-group rg-capstone-dev --policy @lifecycle.json
az storage container immutability-policy create --account-name stcapstonedev001 --container-name audit --period 365 --allow-protected-append-writes true --auth-mode login
az storage container immutability-policy lock --account-name stcapstonedev001 --container-name audit --if-match "<ETAG>"
az storage container legal-hold set --account-name stcapstonedev001 --container-name audit --tags case-2026-01
az storage account network-rule add --account-name stcapstonedev001 --resource-group rg-capstone-dev --ip-address 203.0.113.10
az storage account update --name stcapstonedev001 --resource-group rg-capstone-dev --encryption-key-source Microsoft.Keyvault --encryption-key-vault https://kv-capstone-dev.vault.azure.net --encryption-key-name sa-cmk --identity-type UserAssigned --user-identity-id <MI_ID> --key-vault-user-identity-id <MI_ID>
az storage account keys renew --account-name stcapstonedev001 --resource-group rg-capstone-dev --key primary   # if shared key is still on
az storage account show --name stcapstonedev001 --resource-group rg-capstone-dev --query "{shared:allowSharedKeyAccess, public:allowBlobPublicAccess, net:publicNetworkAccess}"
```

## 10. Databases

```bash
# PostgreSQL Flexible Server: Entra-only auth, zone-redundant HA, private access via delegated subnet
az postgres flexible-server create --name pg-capstone-dev --resource-group rg-capstone-dev --location eastus2 \
  --tier GeneralPurpose --sku-name Standard_D2ds_v5 --storage-size 128 --storage-auto-grow Enabled --version 16 \
  --high-availability ZoneRedundant --backup-retention 14 --geo-redundant-backup Disabled \
  --active-directory-auth Enabled --password-auth Disabled \
  --vnet vnet-spoke --subnet snet-pg --private-dns-zone pg-capstone-dev.private.postgres.database.azure.com \
  --database-name capstone --yes
az postgres flexible-server ad-admin create --server-name pg-capstone-dev --resource-group rg-capstone-dev --display-name sg-dba --object-id <GROUP_OBJECT_ID> --type Group
az postgres flexible-server parameter set --server-name pg-capstone-dev --resource-group rg-capstone-dev --name azure.extensions --value VECTOR,PG_STAT_STATEMENTS
az postgres flexible-server parameter set --server-name pg-capstone-dev --resource-group rg-capstone-dev --name log_min_duration_statement --value 1000
az postgres flexible-server db create --server-name pg-capstone-dev --resource-group rg-capstone-dev --database-name app
az postgres flexible-server replica create --replica-name pg-capstone-dev-r1 --source-server pg-capstone-dev --resource-group rg-capstone-dev --location westus3
az postgres flexible-server restore --name pg-capstone-dev-pitr --source-server pg-capstone-dev --resource-group rg-capstone-dev --restore-time "2026-10-27T03:00:00Z"
az postgres flexible-server stop|start --name pg-capstone-dev --resource-group rg-capstone-dev    # stop up to 7 days (dev cost)
az postgres flexible-server show-connection-string --server-name pg-capstone-dev --database-name capstone --admin-user "sg-dba"
# Connect with Entra token as password
export PGPASSWORD=$(az account get-access-token --resource https://ossrdbms-aad.database.windows.net --query accessToken --output tsv)
psql "host=pg-capstone-dev.postgres.database.azure.com dbname=capstone user=sg-dba sslmode=require"

# Azure SQL Database (Entra-only)
az sql server create --name sql-capstone-dev --resource-group rg-capstone-dev --location eastus2 --enable-ad-only-auth --external-admin-principal-type Group --external-admin-name sg-dba --external-admin-sid <GROUP_OBJECT_ID> --public-network-access Disabled
az sql db create --name appdb --server sql-capstone-dev --resource-group rg-capstone-dev --edition GeneralPurpose --compute-model Serverless --family Gen5 --capacity 2 --auto-pause-delay 60 --zone-redundant false --backup-storage-redundancy Zone

# Cosmos DB for NoSQL (serverless for dev; autoscale for prod)
az cosmosdb create --name cosmos-capstone-dev --resource-group rg-capstone-dev --locations regionName=eastus2 failoverPriority=0 isZoneRedundant=true \
  --default-consistency-level Session --capabilities EnableServerless --public-network-access Disabled --disable-key-based-metadata-write-access true --minimal-tls-version Tls12
az cosmosdb sql database create --account-name cosmos-capstone-dev --resource-group rg-capstone-dev --name appdb
az cosmosdb sql container create --account-name cosmos-capstone-dev --resource-group rg-capstone-dev --database-name appdb --name orders --partition-key-path /tenantId --ttl 2592000
az cosmosdb sql role assignment create --account-name cosmos-capstone-dev --resource-group rg-capstone-dev --role-definition-name "Cosmos DB Built-in Data Contributor" --principal-id <MI_PRINCIPAL_ID> --scope "/"   # data-plane RBAC (separate from ARM RBAC)
az cosmosdb update --name cosmos-capstone-dev --resource-group rg-capstone-dev --disable-key-based-metadata-write-access true
```

## 11. Key Vault

```bash
az keyvault create --name kv-capstone-dev --resource-group rg-capstone-dev --location eastus2 --sku standard \
  --enable-rbac-authorization true --enable-purge-protection true --retention-days 90 \
  --public-network-access Disabled --default-action Deny --bypass AzureServices
az role assignment create --role "Key Vault Administrator" --assignee-object-id "$(az ad signed-in-user show --query id --output tsv)" --assignee-principal-type User --scope "<KV_ID>"

az keyvault secret set --vault-name kv-capstone-dev --name db-url --value "postgresql://..." --expires "2027-01-01T00:00:00Z" --content-type "text/plain"
az keyvault secret show --vault-name kv-capstone-dev --name db-url --query value --output tsv
az keyvault secret list --vault-name kv-capstone-dev --query "[].{name:name, exp:attributes.expires}" --output table
az keyvault secret list-versions --vault-name kv-capstone-dev --name db-url --output table
az keyvault secret set-attributes --vault-name kv-capstone-dev --name db-url --enabled false
az keyvault secret delete --vault-name kv-capstone-dev --name db-url && az keyvault secret recover --vault-name kv-capstone-dev --name db-url

az keyvault key create --vault-name kv-capstone-dev --name sa-cmk --kty RSA --size 3072 --ops wrapKey unwrapKey --exportable false
az keyvault key rotation-policy update --vault-name kv-capstone-dev --name sa-cmk --value rotation-policy.json   # auto-rotate
az keyvault key show --vault-name kv-capstone-dev --name sa-cmk --query key.kid --output tsv

az keyvault certificate create --vault-name kv-capstone-dev --name tls-api --policy "$(az keyvault certificate get-default-policy)"
az keyvault certificate import --vault-name kv-capstone-dev --name tls-api --file cert.pfx

az keyvault network-rule add --name kv-capstone-dev --ip-address 203.0.113.10/32
az keyvault update --name kv-capstone-dev --resource-group rg-capstone-dev --public-network-access Disabled
az keyvault list-deleted --output table
az keyvault recover --name kv-capstone-dev
az keyvault purge --name kv-capstone-dev --location eastus2     # fails with purge protection (by design)
```

## 12. Monitoring and Logs

```bash
# Diagnostic settings on a resource (CloudTrail data events + resource logs analogue)
az monitor diagnostic-settings categories list --resource "<RESOURCE_ID>" --query "value[].{name:name, type:categoryType}" --output table
az monitor diagnostic-settings create --name to-law --resource "<KV_ID>" --workspace "<LAW_ID>" \
  --logs '[{"categoryGroup":"audit","enabled":true}]' --metrics '[{"category":"AllMetrics","enabled":true}]'
az monitor diagnostic-settings create --name to-law --resource "<AKS_ID>" --workspace "<LAW_ID>" \
  --logs '[{"category":"kube-audit-admin","enabled":true},{"category":"kube-apiserver","enabled":true},{"category":"guard","enabled":true},{"category":"cluster-autoscaler","enabled":true}]' --export-to-resource-specific true
az monitor diagnostic-settings list --resource "<RESOURCE_ID>" --output table

# Log Analytics and KQL
az monitor log-analytics workspace create --resource-group rg-capstone-dev --workspace-name law-capstone-dev --retention-time 30
az monitor log-analytics workspace table update --resource-group rg-capstone-dev --workspace-name law-capstone-dev --name ContainerLogV2 --plan Basic   # cheap verbose logs
az monitor log-analytics workspace update --resource-group rg-capstone-dev --workspace-name law-capstone-dev --quota 5   # daily cap GB
LAW_CID=$(az monitor log-analytics workspace show --resource-group rg-capstone-dev --workspace-name law-capstone-dev --query customerId --output tsv)
az monitor log-analytics query --workspace "$LAW_CID" --analytics-query "AzureActivity | where TimeGenerated > ago(1h) | summarize count() by OperationNameValue | top 10 by count_" --output table
az monitor log-analytics query --workspace "$LAW_CID" --analytics-query "ContainerLogV2 | where PodNamespace == 'capstone' | where LogMessage has 'ERROR' | take 50" --timespan PT1H

# Metrics
az monitor metrics list-definitions --resource "<AKS_ID>" --query "[].name.value" --output tsv
az monitor metrics list --resource "<PG_ID>" --metric cpu_percent --interval PT5M --aggregation Average --start-time "$(date -u -d '-1 hour' +%Y-%m-%dT%H:%MZ)"

# Alerts and action groups
az monitor action-group create --name ag-oncall --resource-group rg-capstone-dev --short-name oncall --action email oncall oncall@example.com --action webhook pd https://events.pagerduty.com/integration/<KEY>/enqueue
az monitor metrics alert create --name pg-cpu-high --resource-group rg-capstone-dev --scopes "<PG_ID>" --condition "avg cpu_percent > 80" --window-size 5m --evaluation-frequency 1m --severity 2 --action ag-oncall
az monitor scheduled-query create --name kv-forbidden --resource-group rg-capstone-dev --scopes "<LAW_ID>" --condition "count > 10" \
  --condition-query "AzureDiagnostics | where ResourceProvider == 'MICROSOFT.KEYVAULT' and ResultSignature == 'Forbidden'" --window-size 15m --evaluation-frequency 5m --severity 1 --action-groups "<AG_ID>"
az monitor activity-log list --resource-group rg-capstone-dev --start-time "$(date -u -d '-1 day' +%Y-%m-%dT%H:%MZ)" --query "[].{time:eventTimestamp, op:operationName.value, who:caller, status:status.value}" --output table
az monitor activity-log alert create --name sa-deleted --resource-group rg-capstone-dev --scope "/subscriptions/<SUBSCRIPTION_ID>" --condition "category=Administrative and operationName=Microsoft.Storage/storageAccounts/delete" --action-group "<AG_ID>"
```

## 13. Defender for Cloud

```bash
az security pricing list --query "value[].{plan:name, tier:pricingTier}" --output table
az security pricing create --name Containers --tier Standard
az security pricing create --name KeyVaults --tier Standard
az security pricing create --name StorageAccounts --tier Standard --subplan DefenderForStorageV2 --extensions '[{"name":"OnUploadMalwareScanning","isEnabled":"True","additionalExtensionProperties":{"CapGBPerMonthPerStorageAccount":"5000"}},{"name":"SensitiveDataDiscovery","isEnabled":"True"}]'
az security pricing create --name CloudPosture --tier Standard
az security assessment list --query "[?properties.status.code=='Unhealthy'].{name:properties.displayName, res:properties.resourceDetails.Id}" --output table
az security secure-score-controls list --query "[].{control:properties.displayName, current:properties.score.current, max:properties.score.max}" --output table
az security alert list --query "[].{name:properties.alertDisplayName, sev:properties.severity, time:properties.timeGeneratedUtc}" --output table
az security regulatory-compliance-standards list --query "[].name" --output tsv
az security contact create --name default --emails secops@example.com --alert-notifications on --alerts-admins on
az security auto-provisioning-setting update --name default --auto-provision On
```

## 14. Load Balancing and Edge

```bash
# Standard Load Balancer (internal)
az network lb create --name lb-int --resource-group rg-net --sku Standard --vnet-name vnet-spoke --subnet snet-aks --frontend-ip-name fe --backend-pool-name be --private-ip-address 10.1.0.250
az network lb probe create --lb-name lb-int --resource-group rg-net --name hp --protocol Http --port 8080 --path /healthz
az network lb rule create --lb-name lb-int --resource-group rg-net --name r443 --protocol Tcp --frontend-port 443 --backend-port 8443 --frontend-ip-name fe --backend-pool-name be --probe-name hp

# Application Gateway v2 + WAF policy
az network application-gateway waf-policy create --name wafpol-api --resource-group rg-net --location eastus2
az network application-gateway waf-policy managed-rule rule-set add --policy-name wafpol-api --resource-group rg-net --type OWASP --version 3.2
az network application-gateway waf-policy policy-setting update --policy-name wafpol-api --resource-group rg-net --state Enabled --mode Prevention
az network application-gateway create --name agw-api --resource-group rg-net --location eastus2 --sku WAF_v2 --min-capacity 2 --max-capacity 10 --zones 1 2 3 \
  --vnet-name vnet-spoke --subnet snet-agw --public-ip-address pip-agw --waf-policy wafpol-api --priority 100 --http-settings-protocol Https --http-settings-port 443 --frontend-port 443 --cert-file cert.pfx --cert-password "<PFX_PASSWORD>"

# Front Door Standard/Premium
az afd profile create --profile-name afd-capstone --resource-group rg-edge --sku Premium_AzureFrontDoor
az afd endpoint create --profile-name afd-capstone --resource-group rg-edge --endpoint-name api --enabled-state Enabled
az afd origin-group create --profile-name afd-capstone --resource-group rg-edge --origin-group-name og-api --probe-request-type GET --probe-protocol Https --probe-interval-in-seconds 30 --probe-path /healthz --sample-size 4 --successful-samples-required 3 --additional-latency-in-milliseconds 50
az afd origin create --profile-name afd-capstone --resource-group rg-edge --origin-group-name og-api --origin-name agw --host-name agw-api.example.com --origin-host-header api.example.com --priority 1 --weight 1000 --enabled-state Enabled --https-port 443
az afd route create --profile-name afd-capstone --resource-group rg-edge --endpoint-name api --route-name r-api --origin-group og-api --supported-protocols Https --https-redirect Enabled --forwarding-protocol HttpsOnly --link-to-default-domain Enabled
az afd security-policy create --profile-name afd-capstone --resource-group rg-edge --security-policy-name sp-waf --domains "<AFD_ENDPOINT_ID>" --waf-policy "<FD_WAF_POLICY_ID>"

# DNS
az network dns zone create --name example.com --resource-group rg-edge
az network dns record-set a add-record --zone-name example.com --resource-group rg-edge --record-set-name api --ipv4-address 203.0.113.10
az network dns record-set cname set-record --zone-name example.com --resource-group rg-edge --record-set-name www --cname api.azurefd.net
```

## 15. Cost Management

```bash
az consumption usage list --start-date 2026-10-01 --end-date 2026-10-27 --query "[].{name:instanceName, cost:pretaxCost}" --output table   # per-resource usage
az consumption budget list --output table
az costmanagement query --type ActualCost --timeframe MonthToDate --scope "/subscriptions/<SUBSCRIPTION_ID>" \
  --dataset-aggregation '{"totalCost":{"name":"PreTaxCost","function":"Sum"}}' --dataset-grouping name=ResourceGroupName type=Dimension --output table
az costmanagement export create --name daily-cost --scope "/subscriptions/<SUBSCRIPTION_ID>" --storage-account-id <STORAGE_ID> --storage-container costexports --timeframe MonthToDate --recurrence Daily --recurrence-period from=2026-11-01T00:00:00Z to=2027-11-01T00:00:00Z --schedule-status Active --type ActualCost
az advisor recommendation list --category Cost --query "[].{impact:impact, problem:shortDescription.problem, res:resourceMetadata.resourceId}" --output table
az reservations reservation-order list --output table                   # extension `reservation`
az vm list --show-details --query "[?powerState=='VM running'].{name:name, size:hardwareProfile.vmSize}" --output table   # what is burning money right now
```

## 16. Bicep Essentials

```bash
az bicep build --file main.bicep                                   # compile to ARM JSON
az bicep decompile --file template.json                            # reverse
az bicep lint --file main.bicep
az deployment group what-if --resource-group rg-capstone-dev --template-file main.bicep --parameters main.bicepparam     # terraform plan analogue
az deployment group create --resource-group rg-capstone-dev --template-file main.bicep --parameters main.bicepparam --name deploy-$(date +%s)
az deployment sub create --location eastus2 --template-file sub.bicep        # subscription-scoped (creates RGs, policy)
az deployment mg create --management-group-id mg-landing-zones --location eastus2 --template-file mg.bicep
az deployment group list --resource-group rg-capstone-dev --query "[].{name:name, state:properties.provisioningState, ts:properties.timestamp}" --output table
az deployment operation group list --resource-group rg-capstone-dev --name deploy-1730000000 --query "[?properties.provisioningState=='Failed'].properties.statusMessage"
az stack group create --name stack-capstone --resource-group rg-capstone-dev --template-file main.bicep --action-on-unmanage deleteAll --deny-settings-mode denyDelete   # Deployment Stacks: lifecycle + deny
```

`.bicepparam` file:

```bicep
using 'main.bicep'
param storageName = 'stcapstonedev001'
param location = 'eastus2'
```

## 17. Scripting Patterns

```bash
set -euo pipefail
SUB=$(az account show --query id --output tsv)
RG=rg-capstone-dev
AKS_ID=$(az aks show --name aks-dev --resource-group "$RG" --query id --output tsv)

# Wait for an async create
az aks create ... --no-wait
az aks wait --name aks-dev --resource-group "$RG" --created --interval 30 --timeout 1800

# Existence check without erroring
if az group exists --name "$RG" | grep -q true; then echo "exists"; fi
az keyvault show --name kv-capstone-dev --only-show-errors --output none 2>/dev/null || echo "missing"

# Loop over resources
for id in $(az resource list --resource-group "$RG" --query "[?type=='Microsoft.Storage/storageAccounts'].id" --output tsv); do
  az resource tag --ids "$id" --tags backup=true --is-incremental
done
```

## Next

[04-hands-on-labs.md](04-hands-on-labs.md) uses these commands in sequence to build the capstone's Azure environment.
