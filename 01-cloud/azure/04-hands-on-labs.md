# Azure: Hands-On Labs

| | |
|---|---|
| **Cluster** | 01 Hyperscale Cloud |
| **Role** | Bridge (anchor is AWS) |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md) complete; `rg-capstone-dev` and `law-capstone-dev` exist; Contributor + User Access Administrator on the subscription; Docker locally for Lab 5 |
| **Target version** | Azure CLI 2.60+, AKS 1.30+, PostgreSQL Flexible Server 16 |
| **Capstone phase** | Phase 2 (Months 5-8) |

> Seven labs that build the capstone's Azure environment one layer at a time: governance, identity, network, storage, data, AKS with Workload Identity, and finally GitHub Actions deploying through OIDC. Each lab ends with verification and a teardown. Total cost if you tear down each day: under USD 15 for the whole sequence; the AKS lab is the expensive one (roughly USD 0.60/hour for the nodes), so do not leave it running overnight.

## Lab Conventions

All labs share these variables. Export them at the start of every session (or put them in `~/.capstone-azure.env` and `source` it).

```bash
export LOCATION=eastus2
export RG=rg-capstone-dev
export SUB_ID=$(az account show --query id --output tsv)
export TENANT_ID=$(az account show --query tenantId --output tsv)
export MY_OBJECT_ID=$(az ad signed-in-user show --query id --output tsv)
export SUFFIX=$(echo "$SUB_ID" | cut -c1-6)        # cheap uniqueness for global names
export LAW_ID=$(az monitor log-analytics workspace show --resource-group "$RG" --workspace-name law-capstone-dev --query id --output tsv)
az config set defaults.group="$RG" defaults.location="$LOCATION"
```

Cost control: every lab's resources live in `$RG` except where noted. `az group delete --name $RG --yes` is the universal panic button.

---

## Lab 1: Governance Baseline (Policy, Tags, Locks, Activity Log)

**Objective.** Establish the guardrails that every later lab runs under: allowed regions, required tags inherited from the RG, no public IPs on NICs, and a diagnostic policy that auto-enables logging.

**Prerequisites.** Setup complete.

**Time.** 40 minutes.

### Steps

1. Assign the built-in **Allowed locations** policy at subscription scope.

   ```bash
   az policy assignment create --name allowed-locations \
     --display-name "Allowed locations: eastus2, centralus" \
     --policy "e56962a6-4747-49cd-b67b-bf8b01975c4c" \
     --scope "/subscriptions/$SUB_ID" \
     --params '{"listOfAllowedLocations":{"value":["eastus2","centralus"]}}'
   ```

2. Prove it works by trying to create an RG resource in a disallowed region.

   ```bash
   az storage account create --name "stdenied$SUFFIX" --resource-group "$RG" --location westeurope --sku Standard_LRS
   ```

   Expected output (trimmed):

   ```text
   (RequestDisallowedByPolicy) Resource 'stdenied...' was disallowed by policy. Policy identifiers: '[{"policyAssignment":{"name":"Allowed locations: eastus2, centralus", ...
   ```

   This is the Azure equivalent of an SCP `AccessDenied`, except Policy tells you which assignment blocked you.

3. Assign the built-in **Inherit a tag from the resource group if missing** (`Modify` effect) for `environment` and `costCenter`. This effect needs a managed identity.

   ```bash
   for TAG in environment costCenter; do
     az policy assignment create --name "inherit-tag-$TAG" \
       --policy "ea3f2387-9b95-492a-a190-fcdc54f7b070" \
       --scope "/subscriptions/$SUB_ID" \
       --params "{\"tagName\":{\"value\":\"$TAG\"}}" \
       --mi-system-assigned --location "$LOCATION" \
       --role "Tag Contributor" --identity-scope "/subscriptions/$SUB_ID"
   done
   ```

4. Create a custom **deny public IP on NICs** definition and assign it to the RG.

   ```bash
   cat > /tmp/deny-nic-pip.json <<'EOF'
   {
     "if": {
       "allOf": [
         { "field": "type", "equals": "Microsoft.Network/networkInterfaces" },
         { "not": { "field": "Microsoft.Network/networkInterfaces/ipconfigurations[*].publicIpAddress.id", "notLike": "*" } }
       ]
     },
     "then": { "effect": "deny" }
   }
   EOF
   az policy definition create --name deny-nic-public-ip --display-name "Deny public IPs on NICs" --mode Indexed --rules /tmp/deny-nic-pip.json
   az policy assignment create --name deny-nic-public-ip --policy deny-nic-public-ip --scope "/subscriptions/$SUB_ID/resourceGroups/$RG"
   ```

5. Assign the built-in initiative **Enable Azure Monitor for ... / Configure Azure Key Vault to use diagnostic settings** style policy for Key Vault, pointing at your workspace (`DeployIfNotExists`). Look up the definition ID by display name first, because GUIDs change between regions of the policy catalog less often than display names do, but you should verify.

   ```bash
   DEF_ID=$(az policy definition list --query "[?displayName=='Enable logging by category group for Key vaults (microsoft.keyvault/vaults) to Log Analytics'].id | [0]" --output tsv)
   az policy assignment create --name kv-diag-to-law --policy "$DEF_ID" \
     --scope "/subscriptions/$SUB_ID/resourceGroups/$RG" \
     --params "{\"logAnalytics\":{\"value\":\"$LAW_ID\"}}" \
     --mi-system-assigned --location "$LOCATION" \
     --role "Monitoring Contributor" --identity-scope "/subscriptions/$SUB_ID/resourceGroups/$RG"
   ```

6. Lock the RG against accidental deletion (you will remove this in teardown).

   ```bash
   az lock create --name no-delete --lock-type CanNotDelete --resource-group "$RG"
   ```

### Verify

```bash
az policy assignment list --scope "/subscriptions/$SUB_ID" --query "[].{name:name, policy:policyDefinitionId}" --output table
az policy assignment list --resource-group "$RG" --query "[].name" --output tsv
az group delete --name "$RG" --yes 2>&1 | head -n 1       # must fail with ScopeLocked
az monitor activity-log list --resource-group "$RG" --offset 1h --query "[?contains(operationName.value,'policyAssignments/write')].{op:operationName.value, who:caller}" --output table
```

Expected: the delete attempt prints `(ScopeLocked) The scope '/subscriptions/.../resourceGroups/rg-capstone-dev' cannot perform delete operation because following scope(s) are locked`.

### Cleanup

Leave the policy assignments; later labs rely on them. Remove only the lock when you intend to delete the RG at the very end:

```bash
az lock delete --name no-delete --resource-group "$RG"
```

### What you learned

Policy evaluates on every ARM write and names the assignment that blocked you; `Modify` and `DeployIfNotExists` need a managed identity with a role; locks stop even Owners.

---

## Lab 2: Identity (Managed Identity, Custom Role, Federated Credential)

**Objective.** Create the user-assigned managed identity the capstone API will run as, a least-privilege custom role, and a federated credential for GitHub Actions, with no secrets created anywhere.

**Prerequisites.** Lab 1.

**Time.** 35 minutes.

### Steps

1. Create the workload identity and capture its IDs.

   ```bash
   az identity create --name id-capstone-api --resource-group "$RG"
   export API_MI_CLIENT_ID=$(az identity show --name id-capstone-api --query clientId --output tsv)
   export API_MI_PRINCIPAL_ID=$(az identity show --name id-capstone-api --query principalId --output tsv)
   export API_MI_ID=$(az identity show --name id-capstone-api --query id --output tsv)
   ```

2. Create a custom role that lets a deployer read AKS credentials and pull from ACR, nothing else.

   ```bash
   cat > /tmp/role-aks-deployer.json <<EOF
   {
     "Name": "Capstone AKS Deployer ($SUFFIX)",
     "IsCustom": true,
     "Description": "List user credentials for AKS and pull from ACR",
     "Actions": [
       "Microsoft.ContainerService/managedClusters/read",
       "Microsoft.ContainerService/managedClusters/listClusterUserCredential/action",
       "Microsoft.ContainerRegistry/registries/read"
     ],
     "DataActions": [ "Microsoft.ContainerRegistry/registries/pull/read", "Microsoft.ContainerRegistry/registries/push/write" ],
     "AssignableScopes": [ "/subscriptions/$SUB_ID" ]
   }
   EOF
   az role definition create --role-definition /tmp/role-aks-deployer.json
   ```

3. Create the GitHub deployer app registration with a federated credential (replace `example-org/capstone`).

   ```bash
   export GH_APP_ID=$(az ad app create --display-name "gh-capstone-deploy-$SUFFIX" --query appId --output tsv)
   export GH_SP_OBJECT_ID=$(az ad sp create --id "$GH_APP_ID" --query id --output tsv)
   az ad app federated-credential create --id "$GH_APP_ID" --parameters "{
     \"name\": \"github-main\",
     \"issuer\": \"https://token.actions.githubusercontent.com\",
     \"subject\": \"repo:example-org/capstone:ref:refs/heads/main\",
     \"audiences\": [\"api://AzureADTokenExchange\"]
   }"
   ```

4. Assign the custom role to the GitHub SP at RG scope.

   ```bash
   az role assignment create --role "Capstone AKS Deployer ($SUFFIX)" \
     --assignee-object-id "$GH_SP_OBJECT_ID" --assignee-principal-type ServicePrincipal \
     --scope "/subscriptions/$SUB_ID/resourceGroups/$RG"
   ```

### Verify

```bash
az ad app federated-credential list --id "$GH_APP_ID" --query "[].{name:name, subject:subject}" --output table
az role assignment list --assignee "$GH_SP_OBJECT_ID" --all --query "[].{role:roleDefinitionName, scope:scope}" --output table
az ad app credential list --id "$GH_APP_ID"       # must be [] : no client secrets exist
```

### Cleanup

Keep everything; Labs 6 and 7 use these identities.

### What you learned

A managed identity is a service principal with a platform-managed credential; a federated credential is the Entra equivalent of an IAM OIDC trust policy; custom roles separate `Actions` from `DataActions`.

---

## Lab 3: Network (Spoke VNet, NSG, NAT Gateway, Private DNS)

**Objective.** Build the spoke VNet that AKS, PostgreSQL, and Private Endpoints will live in, with NSG rules, forced NAT egress, and the Private DNS zones required by later labs.

**Prerequisites.** Lab 1.

**Time.** 40 minutes.

### Steps

1. Create the VNet and four subnets.

   ```bash
   az network vnet create --name vnet-spoke --address-prefixes 10.1.0.0/16 --subnet-name snet-aks --subnet-prefixes 10.1.0.0/22
   az network vnet subnet create --name snet-pe --vnet-name vnet-spoke --address-prefixes 10.1.4.0/24 --private-endpoint-network-policies Disabled
   az network vnet subnet create --name snet-pg --vnet-name vnet-spoke --address-prefixes 10.1.5.0/24 --delegations Microsoft.DBforPostgreSQL/flexibleServers
   az network vnet subnet create --name snet-agw --vnet-name vnet-spoke --address-prefixes 10.1.6.0/24
   export SNET_AKS_ID=$(az network vnet subnet show --name snet-aks --vnet-name vnet-spoke --query id --output tsv)
   export SNET_PE_ID=$(az network vnet subnet show --name snet-pe --vnet-name vnet-spoke --query id --output tsv)
   export VNET_ID=$(az network vnet show --name vnet-spoke --query id --output tsv)
   ```

2. NSG for the AKS subnet: allow from the gateway subnet, deny Internet inbound explicitly (the default already denies; explicit rules make audits easier).

   ```bash
   az network nsg create --name nsg-aks
   az network nsg rule create --nsg-name nsg-aks --name allow-agw-to-aks --priority 200 --direction Inbound --access Allow --protocol Tcp \
     --source-address-prefixes 10.1.6.0/24 --destination-port-ranges 80 443 8000-8999
   az network nsg rule create --nsg-name nsg-aks --name deny-internet-inbound --priority 4000 --direction Inbound --access Deny --protocol '*' \
     --source-address-prefixes Internet --destination-port-ranges '*'
   az network vnet subnet update --name snet-aks --vnet-name vnet-spoke --network-security-group nsg-aks
   ```

3. NAT Gateway for deterministic egress.

   ```bash
   az network public-ip create --name pip-nat --sku Standard --zone 1 2 3
   az network nat gateway create --name nat-spoke --public-ip-addresses pip-nat --idle-timeout 10
   az network vnet subnet update --name snet-aks --vnet-name vnet-spoke --nat-gateway nat-spoke
   export NAT_IP=$(az network public-ip show --name pip-nat --query ipAddress --output tsv)
   ```

4. Private DNS zones for Key Vault, ACR, Blob, and PostgreSQL, linked to the VNet.

   ```bash
   for ZONE in privatelink.vaultcore.azure.net privatelink.azurecr.io privatelink.blob.core.windows.net privatelink.postgres.database.azure.com; do
     az network private-dns zone create --name "$ZONE" --output none
     az network private-dns link vnet create --zone-name "$ZONE" --name link-spoke --virtual-network "$VNET_ID" --registration-enabled false --output none
   done
   ```

5. NSG flow logs with Traffic Analytics into the workspace.

   ```bash
   az storage account create --name "stflow$SUFFIX" --sku Standard_LRS --kind StorageV2 --allow-blob-public-access false --min-tls-version TLS1_2
   az network watcher flow-log create --name fl-nsg-aks --location "$LOCATION" --nsg nsg-aks \
     --storage-account "stflow$SUFFIX" --workspace "$LAW_ID" --traffic-analytics true --interval 10 --enabled true --retention 7
   ```

### Verify

```bash
az network vnet subnet list --vnet-name vnet-spoke --query "[].{name:name, prefix:addressPrefix, nsg:networkSecurityGroup.id, nat:natGateway.id, deleg:delegations[0].serviceName}" --output table
az network nsg rule list --nsg-name nsg-aks --include-default --query "sort_by(@,&priority)[].{p:priority, name:name, dir:direction, access:access}" --output table
az network private-dns link vnet list --zone-name privatelink.vaultcore.azure.net --query "[].{name:name, state:virtualNetworkLinkState}" --output table
echo "Egress IP for everything in snet-aks: $NAT_IP"
```

### Cleanup

Keep for later labs. Full teardown is in Lab 7.

### What you learned

Azure subnets span zones; NSGs are priority-ordered with explicit deny; NAT Gateway attaches to subnets rather than routes; Private DNS zones must exist before Private Endpoints are useful.

---

## Lab 4: Storage and Key Vault with Private Endpoints and RBAC-Only Access

**Objective.** Create a Storage Account and a Key Vault that have no public network access, no shared keys, and no access policies; reach them only via Private Endpoints and Entra RBAC.

**Prerequisites.** Labs 2 and 3.

**Time.** 45 minutes.

### Steps

1. Storage account, hardened, with versioning and soft delete, then a lifecycle rule.

   ```bash
   export SA=stcapstone$SUFFIX
   az storage account create --name "$SA" --sku Standard_ZRS --kind StorageV2 --access-tier Hot \
     --min-tls-version TLS1_2 --https-only true --allow-blob-public-access false --allow-shared-key-access false \
     --public-network-access Disabled --default-action Deny
   az storage account blob-service-properties update --account-name "$SA" --enable-versioning true \
     --enable-delete-retention true --delete-retention-days 14 --enable-container-delete-retention true --container-delete-retention-days 14
   cat > /tmp/lifecycle.json <<'EOF'
   { "rules": [ { "name": "tier-backups", "enabled": true, "type": "Lifecycle", "definition": {
       "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["backups/"] },
       "actions": { "baseBlob": { "tierToCool": {"daysAfterModificationGreaterThan": 30}, "tierToArchive": {"daysAfterModificationGreaterThan": 180} },
                    "version": { "delete": {"daysAfterCreationGreaterThan": 90} } } } } ] }
   EOF
   az storage account management-policy create --account-name "$SA" --policy @/tmp/lifecycle.json
   export SA_ID=$(az storage account show --name "$SA" --query id --output tsv)
   ```

2. Key Vault with RBAC authorisation and purge protection.

   ```bash
   export KV=kv-capstone-$SUFFIX
   az keyvault create --name "$KV" --sku standard --enable-rbac-authorization true --enable-purge-protection true --retention-days 90 \
     --public-network-access Disabled --default-action Deny --bypass AzureServices
   export KV_ID=$(az keyvault show --name "$KV" --query id --output tsv)
   ```

3. RBAC: you administer; the API identity reads secrets and blobs.

   ```bash
   az role assignment create --role "Key Vault Administrator" --assignee-object-id "$MY_OBJECT_ID" --assignee-principal-type User --scope "$KV_ID"
   az role assignment create --role "Storage Blob Data Contributor" --assignee-object-id "$MY_OBJECT_ID" --assignee-principal-type User --scope "$SA_ID"
   az role assignment create --role "Key Vault Secrets User" --assignee-object-id "$API_MI_PRINCIPAL_ID" --assignee-principal-type ServicePrincipal --scope "$KV_ID"
   az role assignment create --role "Storage Blob Data Reader" --assignee-object-id "$API_MI_PRINCIPAL_ID" --assignee-principal-type ServicePrincipal --scope "$SA_ID"
   ```

4. Private Endpoints with DNS zone groups.

   ```bash
   for PAIR in "blob:$SA_ID:privatelink.blob.core.windows.net" "vault:$KV_ID:privatelink.vaultcore.azure.net"; do
     GROUP=${PAIR%%:*}; REST=${PAIR#*:}; RES_ID=${REST%:*}; ZONE=${REST##*:}
     ZONE_ID=$(az network private-dns zone show --name "$ZONE" --query id --output tsv)
     az network private-endpoint create --name "pe-$GROUP" --vnet-name vnet-spoke --subnet snet-pe \
       --private-connection-resource-id "$RES_ID" --group-id "$GROUP" --connection-name "conn-$GROUP" --output none
     az network private-endpoint dns-zone-group create --endpoint-name "pe-$GROUP" --name default --private-dns-zone "$ZONE_ID" --zone-name "$GROUP" --output none
   done
   ```

5. Since your laptop is not inside the VNet, temporarily allow your public IP so you can write a secret and a blob (an explicit insecure shortcut for lab purposes; in production an operator reaches these through Bastion or a jump host).

   ```bash
   MY_IP=$(curl -s https://api.ipify.org)
   az keyvault update --name "$KV" --public-network-access Enabled
   az keyvault network-rule add --name "$KV" --ip-address "$MY_IP/32"
   az storage account update --name "$SA" --public-network-access Enabled
   az storage account network-rule add --account-name "$SA" --ip-address "$MY_IP"
   sleep 60   # RBAC and firewall propagation
   az keyvault secret set --vault-name "$KV" --name greeting --value "hello-from-key-vault" --output none
   az storage container create --name backups --account-name "$SA" --auth-mode login --output none
   echo "test $(date)" > /tmp/test.txt
   az storage blob upload --account-name "$SA" --container-name backups --name backups/test.txt --file /tmp/test.txt --auth-mode login --output none
   ```

6. Re-close public access.

   ```bash
   az keyvault update --name "$KV" --public-network-access Disabled
   az storage account update --name "$SA" --public-network-access Disabled
   ```

### Verify

```bash
az keyvault show --name "$KV" --query "{rbac:properties.enableRbacAuthorization, purge:properties.enablePurgeProtection, net:properties.publicNetworkAccess}"
az storage account show --name "$SA" --query "{sharedKey:allowSharedKeyAccess, public:allowBlobPublicAccess, net:publicNetworkAccess, sku:sku.name}"
az network private-endpoint list --query "[].{name:name, state:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status, ip:customDnsConfigs[0].ipAddresses[0]}" --output table
az network private-dns record-set a list --zone-name privatelink.vaultcore.azure.net --query "[].{name:name, ip:aRecords[0].ipv4Address}" --output table
# Audit trail: who read the secret (the DeployIfNotExists from Lab 1 should have created the diagnostic setting)
az monitor diagnostic-settings list --resource "$KV_ID" --query "[].name" --output tsv
```

The Private Endpoint shows `Approved` and the A record resolves the vault name to a `10.1.4.x` address.

### Cleanup

Keep; Lab 6 mounts these into pods.

### What you learned

Azure data planes have two authorisation models and you should turn the legacy one off; Private Endpoints are useless without the matching Private DNS zone; purge protection is irreversible.

---

## Lab 5: PostgreSQL Flexible Server with Entra-Only Authentication and ACR

**Objective.** Provision a VNet-injected PostgreSQL Flexible Server that accepts only Entra tokens, grant the API identity a database role, and stand up the private ACR the cluster will pull from.

**Prerequisites.** Labs 2-4; `psql` and Docker installed locally.

**Time.** 45 minutes (server creation is 5-10 minutes).

### Steps

1. Create the server (Burstable for lab cost; HA off; production flags are in [05-production-patterns.md](05-production-patterns.md)).

   ```bash
   export PG=pg-capstone-$SUFFIX
   az postgres flexible-server create --name "$PG" --tier Burstable --sku-name Standard_B1ms --storage-size 32 --version 16 \
     --active-directory-auth Enabled --password-auth Disabled \
     --vnet vnet-spoke --subnet snet-pg --private-dns-zone privatelink.postgres.database.azure.com \
     --database-name capstone --backup-retention 7 --high-availability Disabled --yes --output none
   az postgres flexible-server ad-admin create --server-name "$PG" --display-name "$(az ad signed-in-user show --query userPrincipalName --output tsv)" --object-id "$MY_OBJECT_ID" --type User
   az postgres flexible-server parameter set --server-name "$PG" --name azure.extensions --value VECTOR,PG_STAT_STATEMENTS --output none
   ```

2. Create a database principal for the API managed identity. Because the server is VNet-injected you need a client in the VNet; run `psql` from a throwaway Azure Container Instance in the AKS subnet (deleted in step 3).

   ```bash
   export PG_TOKEN=$(az account get-access-token --resource https://ossrdbms-aad.database.windows.net --query accessToken --output tsv)
   export MY_UPN=$(az ad signed-in-user show --query userPrincipalName --output tsv)
   az container create --name aci-psql --image postgres:16-alpine --subnet "$SNET_AKS_ID" --os-type Linux --restart-policy Never \
     --secure-environment-variables PGPASSWORD="$PG_TOKEN" \
     --command-line "psql \"host=$PG.postgres.database.azure.com dbname=capstone user=$MY_UPN sslmode=require\" -c \"select * from pgaadauth_create_principal('id-capstone-api', false, false);\" -c \"grant all privileges on database capstone to \\\"id-capstone-api\\\";\" -c \"create extension if not exists vector;\""
   az container logs --name aci-psql
   ```

   Expected output:

   ```text
            pgaadauth_create_principal
   ----------------------------------------------
    Created role for "id-capstone-api"
   (1 row)
   GRANT
   CREATE EXTENSION
   ```

   Note the insecure shortcut: the token is passed as an environment variable to a container; it expires in about an hour and is scoped to PostgreSQL only.

3. Delete the helper container.

   ```bash
   az container delete --name aci-psql --yes --output none
   ```

4. Create the Premium ACR with a Private Endpoint and push the FastAPI image from the Phase 1 capstone (or any small image).

   ```bash
   export ACR=acrcapstone$SUFFIX
   az acr create --name "$ACR" --sku Premium --admin-enabled false --public-network-enabled true --zone-redundancy Enabled --output none
   export ACR_ID=$(az acr show --name "$ACR" --query id --output tsv)
   ZONE_ID=$(az network private-dns zone show --name privatelink.azurecr.io --query id --output tsv)
   az network private-endpoint create --name pe-acr --vnet-name vnet-spoke --subnet snet-pe --private-connection-resource-id "$ACR_ID" --group-id registry --connection-name conn-acr --output none
   az network private-endpoint dns-zone-group create --endpoint-name pe-acr --name default --private-dns-zone "$ZONE_ID" --zone-name registry --output none

   # Build in ACR (no local Docker needed) from a tiny Dockerfile
   mkdir -p /tmp/api && cat > /tmp/api/Dockerfile <<'EOF'
   FROM python:3.12-slim
   RUN pip install --no-cache-dir fastapi uvicorn azure-identity azure-keyvault-secrets azure-storage-blob
   RUN useradd --uid 10001 --no-create-home app
   USER 10001
   WORKDIR /app
   COPY main.py .
   EXPOSE 8000
   CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
   EOF
   cat > /tmp/api/main.py <<'EOF'
   import os
   from fastapi import FastAPI
   from azure.identity import DefaultAzureCredential
   from azure.keyvault.secrets import SecretClient

   app = FastAPI()
   cred = DefaultAzureCredential()

   @app.get("/healthz")
   def healthz():
       return {"ok": True}

   @app.get("/secret")
   def secret():
       client = SecretClient(vault_url=os.environ["KEY_VAULT_URL"], credential=cred)
       return {"greeting": client.get_secret("greeting").value}
   EOF
   az acr build --registry "$ACR" --image api/fastapi:1.0.0 /tmp/api
   ```

5. Close ACR public access now that the image is in (later pushes come from GitHub Actions through a public endpoint with IP rules, or from a self-hosted runner in the VNet).

   ```bash
   az acr update --name "$ACR" --public-network-enabled false --output none
   ```

### Verify

```bash
az postgres flexible-server show --name "$PG" --query "{state:state, aad:authConfig.activeDirectoryAuth, pwd:authConfig.passwordAuth, net:network.publicNetworkAccess}"
az acr repository show-tags --name "$ACR" --repository api/fastapi --output tsv
az acr show --name "$ACR" --query "{public:publicNetworkAccess, admin:adminUserEnabled, zr:zoneRedundancy}"
```

### Cleanup

Keep; Lab 6 depends on all of it. To save money between sessions: `az postgres flexible-server stop --name "$PG"` (auto-restarts after 7 days).

### What you learned

Flexible Server can refuse passwords entirely; Entra database roles are created with `pgaadauth_create_principal`; ACR is one registry with many repositories and builds images in the cloud.

---

## Lab 6: Private AKS with Workload Identity and Key Vault CSI

**Objective.** Create a private AKS cluster using Azure CNI Overlay, Entra authentication with Azure RBAC, no local accounts, the Key Vault CSI and Azure Policy add-ons, and run the API pod with Workload Identity reading a Key Vault secret both via the SDK and via a CSI mount.

**Prerequisites.** Labs 2-5.

**Time.** 90 minutes (cluster creation is 8-12 minutes). Cost: about USD 0.60/hour. Tear down the same day.

### Steps

1. Cluster and kubelet identities, with the RBAC the cluster needs before creation (user-assigned identities avoid the chicken-and-egg problem).

   ```bash
   az identity create --name id-aks-cluster --output none
   az identity create --name id-aks-kubelet --output none
   export AKS_MI_ID=$(az identity show --name id-aks-cluster --query id --output tsv)
   export AKS_MI_PRINCIPAL=$(az identity show --name id-aks-cluster --query principalId --output tsv)
   export KUBELET_MI_ID=$(az identity show --name id-aks-kubelet --query id --output tsv)
   export KUBELET_MI_PRINCIPAL=$(az identity show --name id-aks-kubelet --query principalId --output tsv)
   az role assignment create --role "Network Contributor" --assignee-object-id "$AKS_MI_PRINCIPAL" --assignee-principal-type ServicePrincipal --scope "$VNET_ID" --output none
   az role assignment create --role "Managed Identity Operator" --assignee-object-id "$AKS_MI_PRINCIPAL" --assignee-principal-type ServicePrincipal --scope "$KUBELET_MI_ID" --output none
   az role assignment create --role "AcrPull" --assignee-object-id "$KUBELET_MI_PRINCIPAL" --assignee-principal-type ServicePrincipal --scope "$ACR_ID" --output none
   # Private DNS zone for the API server, so the cluster MI can write records
   az network private-dns zone create --name "privatelink.$LOCATION.azmk8s.io" --output none
   export AKS_DNS_ZONE_ID=$(az network private-dns zone show --name "privatelink.$LOCATION.azmk8s.io" --query id --output tsv)
   az network private-dns link vnet create --zone-name "privatelink.$LOCATION.azmk8s.io" --name link-spoke --virtual-network "$VNET_ID" --registration-enabled false --output none
   az role assignment create --role "Private DNS Zone Contributor" --assignee-object-id "$AKS_MI_PRINCIPAL" --assignee-principal-type ServicePrincipal --scope "$AKS_DNS_ZONE_ID" --output none
   ```

2. Admin group for Kubernetes RBAC (yourself, for the lab).

   ```bash
   export ADMIN_GROUP_ID=$(az ad group create --display-name "sg-aks-admins-$SUFFIX" --mail-nickname "sg-aks-admins-$SUFFIX" --query id --output tsv)
   az ad group member add --group "$ADMIN_GROUP_ID" --member-id "$MY_OBJECT_ID"
   ```

3. Create the cluster. One system node pool, zone-spread, Overlay with Cilium dataplane, NAT Gateway egress from Lab 3.

   ```bash
   az aks create --name aks-dev --location "$LOCATION" \
     --tier standard --kubernetes-version 1.30 \
     --node-count 2 --node-vm-size Standard_D2ds_v5 --zones 1 2 --nodepool-name system --node-osdisk-type Ephemeral --os-sku AzureLinux \
     --vnet-subnet-id "$SNET_AKS_ID" --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium --network-policy cilium \
     --pod-cidr 192.168.0.0/16 --service-cidr 172.16.0.0/16 --dns-service-ip 172.16.0.10 \
     --outbound-type userAssignedNATGateway \
     --enable-private-cluster --private-dns-zone "$AKS_DNS_ZONE_ID" \
     --enable-oidc-issuer --enable-workload-identity \
     --enable-aad --enable-azure-rbac --disable-local-accounts --aad-admin-group-object-ids "$ADMIN_GROUP_ID" \
     --assign-identity "$AKS_MI_ID" --assign-kubelet-identity "$KUBELET_MI_ID" \
     --enable-addons azure-keyvault-secrets-provider,azure-policy,monitoring --enable-secret-rotation --rotation-poll-interval 2m \
     --workspace-resource-id "$LAW_ID" \
     --auto-upgrade-channel patch --node-os-upgrade-channel NodeImage \
     --tags environment=dev --output none
   export AKS_ID=$(az aks show --name aks-dev --query id --output tsv)
   export OIDC_ISSUER=$(az aks show --name aks-dev --query oidcIssuerProfile.issuerUrl --output tsv)
   echo "$OIDC_ISSUER"
   ```

   Expected: an issuer URL like `https://eastus2.oic.prod-aks.azure.com/<TENANT_ID>/<GUID>/`.

4. Grant yourself Kubernetes RBAC via Azure and get credentials. Because the cluster is private, `kubectl` from your laptop cannot reach the API server; use `az aks command invoke` (it runs `kubectl` inside the cluster) for the rest of the lab.

   ```bash
   az role assignment create --role "Azure Kubernetes Service RBAC Cluster Admin" --assignee-object-id "$MY_OBJECT_ID" --assignee-principal-type User --scope "$AKS_ID" --output none
   az aks command invoke --name aks-dev --command "kubectl get nodes -o wide"
   ```

5. Federate the API identity with the Kubernetes ServiceAccount `capstone/api-sa`.

   ```bash
   az identity federated-credential create --name aks-api-sa --identity-name id-capstone-api \
     --issuer "$OIDC_ISSUER" --subject "system:serviceaccount:capstone:api-sa" --audiences api://AzureADTokenExchange --output none
   ```

6. Deploy the workload: namespace, ServiceAccount with the client-id annotation, SecretProviderClass for Key Vault CSI, and the Deployment.

   ```bash
   cat > /tmp/api.yaml <<EOF
   apiVersion: v1
   kind: Namespace
   metadata:
     name: capstone
     labels:
       pod-security.kubernetes.io/enforce: restricted
   ---
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: api-sa
     namespace: capstone
     annotations:
       azure.workload.identity/client-id: "$API_MI_CLIENT_ID"
   ---
   apiVersion: secrets-store.csi.x-k8s.io/v1
   kind: SecretProviderClass
   metadata:
     name: kv-capstone
     namespace: capstone
   spec:
     provider: azure
     parameters:
       usePodIdentity: "false"
       clientID: "$API_MI_CLIENT_ID"
       keyvaultName: "$KV"
       tenantId: "$TENANT_ID"
       objects: |
         array:
           - |
             objectName: greeting
             objectType: secret
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: api
     namespace: capstone
   spec:
     replicas: 2
     selector:
       matchLabels: { app: api }
     template:
       metadata:
         labels:
           app: api
           azure.workload.identity/use: "true"
       spec:
         serviceAccountName: api-sa
         securityContext:
           runAsNonRoot: true
           runAsUser: 10001
           seccompProfile: { type: RuntimeDefault }
         containers:
           - name: api
             image: $ACR.azurecr.io/api/fastapi:1.0.0
             ports: [{ containerPort: 8000 }]
             env:
               - name: KEY_VAULT_URL
                 value: https://$KV.vault.azure.net
             securityContext:
               allowPrivilegeEscalation: false
               capabilities: { drop: ["ALL"] }
               readOnlyRootFilesystem: true
             resources:
               requests: { cpu: 100m, memory: 128Mi }
               limits: { cpu: 500m, memory: 256Mi }
             volumeMounts:
               - name: kv
                 mountPath: /mnt/secrets
                 readOnly: true
         volumes:
           - name: kv
             csi:
               driver: secrets-store.csi.k8s.io
               readOnly: true
               volumeAttributes:
                 secretProviderClass: kv-capstone
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: api
     namespace: capstone
   spec:
     selector: { app: api }
     ports: [{ port: 80, targetPort: 8000 }]
   EOF
   az aks command invoke --name aks-dev --command "kubectl apply -f api.yaml" --file /tmp/api.yaml
   ```

7. Assign the Azure Policy for Kubernetes baseline initiative to the cluster (enforced in-cluster by Gatekeeper via the add-on).

   ```bash
   INIT_ID=$(az policy set-definition list --query "[?displayName=='Kubernetes cluster pod security baseline standards for Linux-based workloads'].id | [0]" --output tsv)
   az policy assignment create --name aks-psb --policy-set-definition "$INIT_ID" --scope "$AKS_ID" --params '{"effect":{"value":"Deny"}}' --output none
   ```

### Verify

```bash
az aks command invoke --name aks-dev --command "kubectl get pods -n capstone -o wide && kubectl get sa api-sa -n capstone -o yaml | grep client-id"
# SDK path: token obtained via projected SA token exchanged for an Entra token
az aks command invoke --name aks-dev --command "kubectl run curl --rm -i --restart=Never --image=curlimages/curl -n capstone -- curl -s http://api/secret"
# CSI path: file mounted from Key Vault
az aks command invoke --name aks-dev --command "kubectl exec deploy/api -n capstone -- cat /mnt/secrets/greeting"
# Workload identity env injected by the webhook
az aks command invoke --name aks-dev --command "kubectl exec deploy/api -n capstone -- env | grep AZURE_"
# Policy add-on is running and (after ~20 min) reports compliance
az aks command invoke --name aks-dev --command "kubectl get pods -n gatekeeper-system && kubectl get constrainttemplates | head"
# Negative test: a privileged pod must be rejected by Gatekeeper
az aks command invoke --name aks-dev --command "kubectl run bad --image=nginx -n capstone --privileged" 2>&1 | tail -n 2
```

Expected: `/secret` returns `{"greeting":"hello-from-key-vault"}`; the CSI file contains the same value; env shows `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_FEDERATED_TOKEN_FILE`, `AZURE_AUTHORITY_HOST`; the privileged pod is denied with `admission webhook "validation.gatekeeper.sh" denied the request`.

Check the audit trail in Log Analytics (allow 5-10 minutes for ingestion):

```bash
LAW_CID=$(az monitor log-analytics workspace show --resource-group "$RG" --workspace-name law-capstone-dev --query customerId --output tsv)
az monitor log-analytics query --workspace "$LAW_CID" --analytics-query "AzureDiagnostics | where ResourceProvider == 'MICROSOFT.KEYVAULT' | where OperationName == 'SecretGet' | project TimeGenerated, identity_claim_appid_g, ResultType | take 5" --output table
```

The `identity_claim_appid_g` column equals `$API_MI_CLIENT_ID`: the pod's identity, not a shared secret.

### Cleanup

Stop the cluster between sessions (`az aks stop --name aks-dev`) or delete it:

```bash
az policy assignment delete --name aks-psb --scope "$AKS_ID"
az aks delete --name aks-dev --yes --no-wait
```

### What you learned

Workload Identity is IRSA with Entra: OIDC issuer, ServiceAccount annotation, federated credential; the Key Vault CSI driver and the SDK are two consumers of the same identity; Azure Policy for Kubernetes is Gatekeeper managed by Azure; private clusters are operated via `az aks command invoke`, Bastion, or VPN.

---

## Lab 7: GitHub Actions Deploys via OIDC, Then Full Teardown

**Objective.** Deploy to the cluster from GitHub Actions with no stored cloud credential, then destroy every lab resource and confirm the subscription is clean.

**Prerequisites.** Lab 6 running; a GitHub repository you control (`example-org/capstone`); Lab 2's federated credential subject matches the repository and branch.

**Time.** 40 minutes.

### Steps

1. Grant the GitHub SP the ability to deploy into the `capstone` namespace and pull/push ACR (the custom role from Lab 2 covers ACR; add namespace-scoped Kubernetes RBAC).

   ```bash
   az role assignment create --role "Azure Kubernetes Service RBAC Writer" --assignee-object-id "$GH_SP_OBJECT_ID" --assignee-principal-type ServicePrincipal --scope "$AKS_ID/namespaces/capstone" --output none
   az acr update --name "$ACR" --public-network-enabled true --default-action Deny --output none   # GitHub-hosted runners need a public path; allow only during the job or use a self-hosted runner in the VNet
   ```

2. Set three repository variables in GitHub (not secrets, because none of them are secret): `AZURE_CLIENT_ID=$GH_APP_ID`, `AZURE_TENANT_ID=$TENANT_ID`, `AZURE_SUBSCRIPTION_ID=$SUB_ID`.

3. Add the workflow.

   ```yaml
   # .github/workflows/deploy-azure.yml
   name: deploy-azure
   on:
     push:
       branches: [main]
   permissions:
     id-token: write     # required for OIDC
     contents: read
   env:
     RG: rg-capstone-dev
     ACR: acrcapstone${{ vars.SUFFIX }}
   jobs:
     build-deploy:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: azure/login@v2
           with:
             client-id: ${{ vars.AZURE_CLIENT_ID }}
             tenant-id: ${{ vars.AZURE_TENANT_ID }}
             subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
         - name: Allow this runner through the ACR firewall for the duration of the job
           run: |
             RUNNER_IP=$(curl -s https://api.ipify.org)
             echo "RUNNER_IP=$RUNNER_IP" >> "$GITHUB_ENV"
             az acr network-rule add --name "$ACR" --ip-address "$RUNNER_IP"
         - name: Build and push in ACR
           run: az acr build --registry "$ACR" --image api/fastapi:${{ github.sha }} ./services/api
         - name: Deploy
           run: |
             az aks command invoke --name aks-dev --resource-group "$RG" \
               --command "kubectl set image deployment/api api=$ACR.azurecr.io/api/fastapi:${{ github.sha }} -n capstone && kubectl rollout status deployment/api -n capstone --timeout=120s"
         - name: Remove runner IP
           if: always()
           run: az acr network-rule remove --name "$ACR" --ip-address "$RUNNER_IP"
   ```

4. Push to `main` and watch the run. The `azure/login` step prints the exchanged identity; there is no secret in the repository.

### Verify

```bash
az aks command invoke --name aks-dev --command "kubectl get deployment api -n capstone -o jsonpath='{.spec.template.spec.containers[0].image}'"
az monitor activity-log list --resource-group "$RG" --offset 1h --query "[?caller=='$GH_APP_ID' || contains(caller,'gh-capstone')].{op:operationName.value, status:status.value}" --output table
```

The image tag equals the commit SHA and the Activity Log shows the GitHub app as the caller.

### Full Teardown

Order matters only for the lock and for identities referenced outside the RG.

```bash
az policy assignment delete --name aks-psb --scope "$AKS_ID" 2>/dev/null || true
az lock delete --name no-delete --resource-group "$RG" 2>/dev/null || true
az group delete --name "$RG" --yes --no-wait
az group wait --name "$RG" --deleted --timeout 1800
az group delete --name "MC_${RG}_aks-dev_${LOCATION}" --yes --no-wait 2>/dev/null || true   # node RG is normally deleted with the cluster

# Tenant-level objects not in the RG
az ad app delete --id "$GH_APP_ID"
az ad group delete --group "$ADMIN_GROUP_ID"
az role definition delete --name "Capstone AKS Deployer ($SUFFIX)"

# Subscription-level policy assignments and definitions from Lab 1
for A in allowed-locations inherit-tag-environment inherit-tag-costCenter; do az policy assignment delete --name "$A" --scope "/subscriptions/$SUB_ID"; done
az policy definition delete --name deny-nic-public-ip

# Key Vault is soft-deleted with purge protection: it will sit in the deleted state for 90 days (no charge). Verify:
az keyvault list-deleted --query "[].{name:name, purgeDate:properties.scheduledPurgeDate}" --output table
```

Confirm nothing is left billing:

```bash
az resource list --query "[?resourceGroup=='$RG']" --output table      # empty
az graph query -q "Resources | where tags.environment == 'dev' | project name, type, resourceGroup" --output table
az consumption usage list --start-date "$(date -u +%Y-%m-01)" --end-date "$(date -u +%Y-%m-%d)" --query "sum([].to_number(pretaxCost))"
```

### What you learned

The end-to-end chain from a Git push to a running pod involves four identities (GitHub app, cluster MI, kubelet MI, workload MI), none of which has a stored secret; a Resource Group delete removes everything deployed except tenant-level directory objects and subscription-level policy, which is exactly why those are the things to script separately.

---

## Stretch Labs (no step-by-step; design them yourself)

1. **Hub-spoke with Azure Firewall egress.** Add a hub VNet with Azure Firewall, peer it, and switch the cluster to `--outbound-type userDefinedRouting` with the required `AzureKubernetesService` FQDN tag rule. Observe what breaks when the rule is missing.
2. **Container Apps instead of AKS.** Deploy the same image to a Container Apps environment in the VNet with scale-to-zero and a Key Vault secret reference. Compare cold start, cost, and operational surface.
3. **Bicep what-if.** Rewrite Lab 4 as `main.bicep`, run `az deployment group what-if`, change one property, and read the diff.
4. **Cosmos DB serverless** with data-plane RBAC for the API identity and a change-feed-triggered Azure Function (Flex Consumption) using a managed identity to Storage.
5. **Immutable audit container** with a locked time-based retention policy; attempt to delete a blob and read the error.
