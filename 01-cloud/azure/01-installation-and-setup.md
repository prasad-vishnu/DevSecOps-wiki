# Azure: Installation and Setup

| | |
|---|---|
| **Cluster** | 01 Hyperscale Cloud |
| **Role** | Bridge (anchor is AWS) |
| **Prerequisites** | A Microsoft account or work account; a free Azure subscription or Visual Studio subscription; AWS CLI experience |
| **Target version** | Azure CLI 2.60+, Bicep 0.28+, kubectl 1.30+ |
| **Capstone phase** | Phase 2 (Months 5-8) |

> After this file you will have the Azure CLI installed on Linux, macOS, Windows, or in a container; you will be signed in with a short-lived token (not a stored secret); you will know how to switch subscriptions and tenants; you will have registered the resource providers the capstone needs; and you will have a Resource Group, a budget alert, and a Log Analytics workspace ready for the labs.

## 1. Mental Model Before Installing Anything

The `az` CLI is a Python application that calls the Azure Resource Manager REST API at `https://management.azure.com` and, for some data-plane operations, service-specific endpoints (for example `https://<account>.blob.core.windows.net`). Authentication produces an Entra ID access token (a JWT, valid for roughly one hour) which `az` caches and refreshes using a refresh token stored in `~/.azure/`.

Coming from AWS:

| AWS | Azure |
|---|---|
| `~/.aws/credentials` and `~/.aws/config` | `~/.azure/` (token cache, `azureProfile.json`, `config`) |
| `AWS_PROFILE` named profiles | `az account set --subscription` (one active subscription per CLI context) |
| `aws sts get-caller-identity` | `az account show` and `az ad signed-in-user show` |
| `aws configure sso` + `aws sso login` | `az login` (browser or device code) |
| `aws --query` (JMESPath) | `az --query` (JMESPath, identical syntax) |
| `--output table/json/text` | `--output table/json/jsonc/tsv/yaml/none` |

## 2. Install the Azure CLI

### Linux (Debian / Ubuntu)

Use Microsoft's package repository so `apt upgrade` keeps the CLI current.

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl apt-transport-https lsb-release gnupg
sudo mkdir -p /etc/apt/keyrings
curl -sLS https://packages.microsoft.com/keys/microsoft.asc \
  | gpg --dearmor \
  | sudo tee /etc/apt/keyrings/microsoft.gpg > /dev/null
sudo chmod go+r /etc/apt/keyrings/microsoft.gpg

AZ_DIST=$(lsb_release -cs)
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/microsoft.gpg] https://packages.microsoft.com/repos/azure-cli/ ${AZ_DIST} main" \
  | sudo tee /etc/apt/sources.list.d/azure-cli.list

sudo apt-get update
sudo apt-get install -y azure-cli
```

### Linux (RHEL / Fedora / Amazon Linux)

```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo dnf install -y https://packages.microsoft.com/config/rhel/9/packages-microsoft-prod.rpm
sudo dnf install -y azure-cli
```

### macOS

```bash
brew update && brew install azure-cli
```

### Windows

```powershell
winget install --exact --id Microsoft.AzureCLI
```

Or with the MSI from https://aka.ms/installazurecliwindows. Restart the terminal afterwards so `PATH` is refreshed.

### Container (for CI runners and reproducible labs)

```bash
docker run -it --rm \
  --volume "${HOME}/.azure:/root/.azure" \
  mcr.microsoft.com/azure-cli:latest \
  az version
```

The volume mount persists the token cache between runs. Never bake a token cache into an image.

### Verify

```bash
az version
```

Expected output (versions will differ):

```json
{
  "azure-cli": "2.64.0",
  "azure-cli-core": "2.64.0",
  "azure-cli-telemetry": "1.1.0",
  "extensions": {}
}
```

Upgrade in place at any time:

```bash
az upgrade --yes
```

## 3. Sign In Without Storing Secrets

### Interactive (your laptop)

```bash
az login
```

This opens a browser. On a headless machine or over SSH:

```bash
az login --use-device-code
```

Expected output after authentication (trimmed):

```json
[
  {
    "cloudName": "AzureCloud",
    "homeTenantId": "<TENANT_ID>",
    "id": "<SUBSCRIPTION_ID>",
    "isDefault": true,
    "name": "Visual Studio Enterprise Subscription",
    "state": "Enabled",
    "tenantId": "<TENANT_ID>",
    "user": {
      "name": "you@example.com",
      "type": "user"
    }
  }
]
```

Since CLI 2.61 `az login` shows an interactive subscription picker when you have access to more than one. Disable it in automation with `az config set core.login_experience_v2=off`.

### Sign in to a specific tenant

If your account is a guest in multiple tenants (common in consulting), target the tenant explicitly:

```bash
az login --tenant <TENANT_ID_OR_DOMAIN>
```

### CI/CD: workload identity federation (preferred, no secret)

In GitHub Actions use the `azure/login` action with an app registration or user-assigned managed identity that has a federated credential trusting the GitHub OIDC issuer. There is no secret. The setup is in [05-production-patterns.md](05-production-patterns.md#github-actions-with-workload-identity-federation). The equivalent raw CLI call, for understanding only:

```bash
az login --service-principal \
  --username <APP_CLIENT_ID> \
  --tenant <TENANT_ID> \
  --federated-token "$(cat /path/to/oidc-token.jwt)"
```

### Inside Azure: managed identity

On a VM, in Container Apps, in Azure Functions, or in an AKS pod with Workload Identity:

```bash
az login --identity                                # system-assigned
az login --identity --username <USER_ASSIGNED_CLIENT_ID>   # user-assigned
```

### Service principal with a client secret (legacy; avoid)

Shown only so you recognise it. This is the equivalent of an IAM user access key and should be replaced with federated credentials.

```bash
az login --service-principal --username <APP_CLIENT_ID> --password "<CLIENT_SECRET>" --tenant <TENANT_ID>
```

### Sign out

```bash
az logout
az account clear   # removes all cached subscriptions and tokens
```

## 4. Select the Subscription and Tenant

```bash
az account list --output table
```

Expected output:

```text
Name                                   CloudName    SubscriptionId                        TenantId                              State    IsDefault
-------------------------------------  -----------  ------------------------------------  ------------------------------------  -------  -----------
Visual Studio Enterprise Subscription  AzureCloud   <SUBSCRIPTION_ID>                     <TENANT_ID>                           Enabled  True
Capstone Dev                           AzureCloud   <SUBSCRIPTION_ID_2>                   <TENANT_ID>                           Enabled  False
```

Set the active subscription (equivalent to picking an AWS profile):

```bash
az account set --subscription "<SUBSCRIPTION_ID>"
az account show --query "{name:name, id:id, tenant:tenantId}" --output table
```

Who am I:

```bash
az ad signed-in-user show --query "{upn:userPrincipalName, id:id}" --output table
```

## 5. CLI Configuration Defaults

Set defaults so you stop repeating `--resource-group` and `--location`:

```bash
az config set defaults.location=eastus2
az config set defaults.group=rg-capstone-dev
az config set core.output=table          # json is the default; table is friendlier interactively
az config set core.collect_telemetry=no
az config set core.only_show_errors=yes  # suppress preview warnings in scripts (optional)
```

Inspect:

```bash
az config get
cat ~/.azure/config
```

Enable tab completion (bash):

```bash
echo 'source /etc/bash_completion.d/azure-cli' >> ~/.bashrc   # path varies by package; run `az completion --help` for alternatives
```

Zsh users: the Homebrew formula installs completion automatically.

## 6. Register Resource Providers

A fresh subscription has only a handful of resource providers registered. Deployments fail with `MissingSubscriptionRegistration` until the provider is registered. There is no AWS equivalent (all service APIs are available by default), so this step surprises everyone coming from AWS.

```bash
for ns in Microsoft.Compute Microsoft.Network Microsoft.Storage Microsoft.KeyVault \
          Microsoft.ContainerService Microsoft.ContainerRegistry Microsoft.App \
          Microsoft.DBforPostgreSQL Microsoft.OperationalInsights Microsoft.Insights \
          Microsoft.ManagedIdentity Microsoft.Authorization Microsoft.PolicyInsights \
          Microsoft.Security Microsoft.Web Microsoft.DocumentDB; do
  az provider register --namespace "$ns"
done
```

Registration is asynchronous. Check:

```bash
az provider list --query "[?registrationState!='Registered' && namespace in ['Microsoft.ContainerService','Microsoft.App']].{ns:namespace, state:registrationState}" --output table
az provider show --namespace Microsoft.ContainerService --query registrationState --output tsv
```

Expected output once done:

```text
Registered
```

Some AKS features are behind preview feature flags registered with `az feature register --namespace Microsoft.ContainerService --name <FeatureName>` followed by re-registering the provider. The labs in this folder use only GA features.

## 7. Install Companion Tools

```bash
# Bicep (bundled with az; this downloads the compiler binary)
az bicep install
az bicep version

# kubectl and kubelogin for AKS (kubelogin handles Entra ID auth for kubectl)
az aks install-cli
kubectl version --client
kubelogin --version

# Useful extensions (extensions are versioned separately from core)
az extension add --name containerapp --upgrade
az extension add --name resource-graph --upgrade
az extension add --name aks-preview --upgrade    # only if a lab explicitly needs a preview feature
az extension list --output table
```

Allow extensions to be installed automatically when a command needs one:

```bash
az config set extension.use_dynamic_install=yes_without_prompt
```

## 8. First Resources: Resource Group, Budget, Log Analytics

### Resource Group

```bash
az group create \
  --name rg-capstone-dev \
  --location eastus2 \
  --tags environment=dev owner=platform-team costCenter=capstone
```

Expected output:

```json
{
  "id": "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/rg-capstone-dev",
  "location": "eastus2",
  "managedBy": null,
  "name": "rg-capstone-dev",
  "properties": { "provisioningState": "Succeeded" },
  "tags": { "costCenter": "capstone", "environment": "dev", "owner": "platform-team" },
  "type": "Microsoft.Resources/resourceGroups"
}
```

Note the resource ID format. Every Azure resource has a fully qualified ID of the form `/subscriptions/<sub>/resourceGroups/<rg>/providers/<namespace>/<type>/<name>`. This ID is the *scope* used in RBAC and Policy, which makes it the Azure counterpart of an ARN.

### Budget alert (Golden Rule 4: keep lab spend under control)

```bash
az consumption budget create \
  --budget-name capstone-monthly \
  --amount 15 \
  --category Cost \
  --time-grain Monthly \
  --start-date "$(date -u +%Y-%m-01)" \
  --end-date "2027-12-01" \
  --resource-group rg-capstone-dev
```

Budgets are evaluated several times a day. For notifications to email or an Action Group, the `az consumption budget` command set is limited; use the portal or Bicep (`Microsoft.Consumption/budgets` with a `notifications` block). A Bicep example is in [05-production-patterns.md](05-production-patterns.md#cost-management).

### Log Analytics workspace (every lab sends diagnostics here)

```bash
az monitor log-analytics workspace create \
  --resource-group rg-capstone-dev \
  --workspace-name law-capstone-dev \
  --location eastus2 \
  --retention-time 30 \
  --sku PerGB2018

LAW_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-capstone-dev \
  --workspace-name law-capstone-dev \
  --query id --output tsv)
echo "$LAW_ID"
```

Enable Activity Log export to the workspace (the Azure equivalent of a CloudTrail trail):

```bash
az monitor diagnostic-settings subscription create \
  --name activity-to-law \
  --location eastus2 \
  --workspace "$LAW_ID" \
  --logs '[{"category":"Administrative","enabled":true},{"category":"Security","enabled":true},{"category":"Policy","enabled":true},{"category":"Alert","enabled":true},{"category":"Autoscale","enabled":true},{"category":"ResourceHealth","enabled":true},{"category":"ServiceHealth","enabled":true},{"category":"Recommendation","enabled":true}]'
```

## 9. Verification Checklist

Run each; all should succeed before starting [04-hands-on-labs.md](04-hands-on-labs.md).

```bash
az account show --query "{sub:name, tenant:tenantId, user:user.name}" --output table
az group show --name rg-capstone-dev --query properties.provisioningState --output tsv
az provider show --namespace Microsoft.ContainerService --query registrationState --output tsv
az bicep version
kubectl version --client --output yaml | head -n 5
az role assignment list --assignee "$(az ad signed-in-user show --query id --output tsv)" \
  --scope "/subscriptions/$(az account show --query id --output tsv)" \
  --query "[].roleDefinitionName" --output tsv
```

The last command should show `Owner` or `Contributor` plus `User Access Administrator` on a personal subscription. If you only see `Reader`, you cannot complete the labs.

Query your tenant-wide estate with Resource Graph (the Azure analogue of AWS Config advanced queries, but instantaneous and free):

```bash
az graph query -q "Resources | summarize count() by type | order by count_ desc | take 10" --output table
```

## 10. Common Setup Problems

| Symptom | Cause | Fix |
|---|---|---|
| `az: command not found` after Windows install | Terminal opened before install finished | Open a new terminal; check `where az` |
| `AADSTS50076: Due to a configuration change made by your administrator ... multi-factor authentication` | Tenant requires MFA for CLI | Use `az login` interactively; for CI use federated identity, not a user |
| `AADSTS700016: Application with identifier ... was not found in the directory` | Wrong tenant for the service principal | Add `--tenant <TENANT_ID>` |
| `MissingSubscriptionRegistration` | Provider not registered | Section 6 |
| `(AuthorizationFailed) The client ... does not have authorization to perform action` | Role not assigned at this scope | `az role assignment list --assignee ... --all`; ask for Contributor on the RG |
| `az login` hangs in WSL | No browser bridge | `az login --use-device-code` |
| Extension conflicts (`aks-preview` shadows GA commands) | Preview extension installed | `az extension remove --name aks-preview` when not needed |
| Slow commands | Long-running ARM operations polled synchronously | Add `--no-wait`, then `az <resource> wait --created` |

## 11. Clean Up Setup Resources (optional)

If you want to tear down after verifying setup, everything created above lives in one Resource Group, which is the whole point:

```bash
az group delete --name rg-capstone-dev --yes --no-wait
```

The subscription-level diagnostic setting is not in the RG:

```bash
az monitor diagnostic-settings subscription delete --name activity-to-law --yes
```

Keep the RG if you are continuing straight into the labs.

## Next

Read [02-core-concepts.md](02-core-concepts.md) before running any further commands. The hierarchy and identity sections explain why the labs are structured the way they are.
