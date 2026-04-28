# Private Network Standard Agent Setup

> **MANDATORY:** Read [Standard Agent Setup with Network Isolation docs](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-private-link?tabs=azure-portal&pivots=fdp-project) before proceeding. It covers RBAC requirements, resource provider registration, and role assignments.

## Overview

Extends [standard agent setup](standard-agent-setup.md) with full VNet isolation using private endpoints and subnet delegation. All resources communicate over private network only.

> ⚠️ **Critical:** With `publicNetworkAccess` disabled, the agent endpoint is **not reachable from outside the VNet** — even with correct credentials and roles. All agent management (creation, testing, SDK calls) must happen from within the VNet or a connected network (VPN Gateway, ExpressRoute, Azure Bastion VM).

## Networking Constraints

Two subnets required:

| Subnet | CIDR | Purpose | Delegation |
|--------|------|---------|------------|
| Agent Subnet | /24 (e.g., 192.168.0.0/24) | Agent workloads | `Microsoft.App/environments` (exclusive) |
| Private Endpoint Subnet | /24 (e.g., 192.168.1.0/24) | Private endpoints | None |

- All Foundry resources **must be in the same region as the VNet**.
- Agent subnet must be exclusive to one Foundry account.
- VNet address space must not overlap with existing networks or reserved ranges.

> ⚠️ **Warning:** If providing an existing VNet, ensure both subnets exist before deployment. Otherwise the template creates a new VNet with default address spaces.

## Pre-Deployment Checklist

Complete all items before running the template. Skipping these causes preventable deployment failures.

### 1. RBAC

The deploying identity must have **Owner**, or **Contributor + User Access Administrator** (both roles) on the resource group. UAA alone is not sufficient — it lacks `Microsoft.Resources/deployments/validate/action`.

### 2. Model Version

Verify the correct `modelVersion` for your chosen model **before** parameterizing. OpenAI models use date strings; Mistral AI and Meta models use integers. Mixing these formats causes `DeploymentModelNotSupported`.

```bash
az cognitiveservices model list --location <location> \
  --query "[?model.name=='<modelName>'].{version:model.version}" -o table
```

Reference:

| Vendor | Example Model | Version Format | Example |
|--------|--------------|----------------|---------|
| OpenAI | gpt-4.1 | Date string | `2025-04-14` |
| OpenAI | gpt-4o | Date string | `2024-11-20` |
| Mistral AI | Mistral-Large-3 | Integer | `1` |
| Meta | Llama-3.3-70B-Instruct | Integer | `9` |

### 3. VNet Name — Never Reuse on Retry

If a prior deployment reached the capability host step and failed, Azure Container Apps leaves a `legionservicelink` service association on the agent subnet. This association **cannot be removed by the user**. Reusing the same VNet name on retry will fail immediately.

> ⛔ **Always use a new `vnetName` in `main.bicepparam` on every retry.** Never reuse a VNet name from a deployment that reached the capability host provisioning step.

## Deployment

**Always use the official Bicep template:**
[Private Network Standard Agent Setup Bicep](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/15-private-network-standard-agent-setup)

```bash
az deployment group create \
  --resource-group <rg> \
  --template-file main.bicep \
  --parameters main.bicepparam \
  --name <deployment-name>
```

> ⚠️ **Warning:** Capability host provisioning is **asynchronous** (10–20 minutes). The CLI produces no output during this phase. Do not assume the deployment is hung — poll for status.

## Deployment Status Monitoring

Poll during deployment. Run in a separate terminal or periodically during the wait.

```bash
# Overall deployment state
az deployment group show \
  --resource-group <rg> \
  --name <deployment-name> \
  --query "{state:properties.provisioningState,error:properties.error}" \
  -o json

# Per-resource progress (run every 2–3 minutes)
az deployment operation group list \
  --resource-group <rg> \
  --name <deployment-name> \
  --query "[].{resource:properties.targetResource.resourceType,state:properties.provisioningState}" \
  -o table

# Wait with timeout (blocks until succeeded or failed)
az deployment group wait \
  --resource-group <rg> \
  --name <deployment-name> \
  --created --timeout 1800
```

Expected resource progression:

1. `Microsoft.Network/virtualNetworks` → Succeeded
2. `Microsoft.CognitiveServices/accounts` → Succeeded
3. `Microsoft.Search/searchServices` → Succeeded
4. `Microsoft.Storage/storageAccounts` → Succeeded
5. `Microsoft.DocumentDB/databaseAccounts` → Succeeded
6. `Microsoft.Network/privateEndpoints` (×5) → Succeeded
7. `Microsoft.MachineLearningServices/workspaces` → Succeeded
8. `Microsoft.MachineLearningServices/workspaces/capabilityHosts` → Succeeded (async — takes longest)

## Error Recovery

When a deployment fails, follow this workflow:

### Step 1 — Identify the error

```bash
az deployment operation group list \
  --resource-group <rg> \
  --name <deployment-name> \
  --query "[?properties.provisioningState=='Failed'].{resource:properties.targetResource.resourceType,error:properties.statusMessage}" \
  -o json
```

### Step 2 — Map to known fix

| Error | Root Cause | Fix |
|-------|-----------|-----|
| `DeploymentModelNotSupported` | Wrong `modelVersion` format for vendor | Query model catalog; use integer version for Mistral AI / Meta |
| `legionservicelink` / subnet in use | Orphaned service link from prior attempt | Use a new `vnetName` — do not reuse the prior VNet |
| `AuthorizationFailed` on `validate/action` | Missing Contributor role | Assign Contributor + User Access Administrator to deploying identity |
| `SubnetDelegationAlreadyExists` | Agent subnet already delegated to another resource | Use a new VNet or open a support ticket to remove the delegation |

### Step 3 — Present fix to user and get approval

Before re-deploying, show the user:
- What failed and why
- What file/parameter will be changed
- The new `vnetName` to use (must be different from the failed run)

### Step 4 — Re-deploy with a new deployment name

```bash
# Update main.bicepparam: change vnetName to a new unique name
az deployment group create \
  --resource-group <rg> \
  --template-file main.bicep \
  --parameters main.bicepparam \
  --name <deployment-name>-retry
```

## Post-Deployment

### 1. Assign Azure AI Developer role

The template does not assign this role automatically. Without it, the deploying user cannot create or manage agents (lacks `Microsoft.CognitiveServices/accounts/AIServices/agents/write`).

```bash
az role assignment create \
  --role "Azure AI Developer" \
  --assignee <your-object-id-or-email> \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<ai-account-name>
```

### 2. Deploy a model

Verify model version format before running (see Pre-Deployment Checklist step 2):

```bash
az cognitiveservices account deployment create \
  --resource-group <rg> \
  --name <ai-account-name> \
  --deployment-name <deployment-name> \
  --model-name <modelName> \
  --model-version <modelVersion> \
  --model-format <format> \
  --sku-name GlobalStandard \
  --sku-capacity 50
```

Fall back to `Standard` SKU if `GlobalStandard` quota is exhausted.

### 3. Create the agent

Agent management requires connectivity to the private endpoint. Use the correct token audience:

- Token audience: `https://ai.azure.com` (not `https://cognitiveservices.azure.com/`)
- Agent endpoint: `https://<account>.services.ai.azure.com/api/projects/<project>/assistants`

To reach the private endpoint from outside the VNet, you need one of:
- Azure Bastion + jump VM inside the VNet
- VPN Gateway or ExpressRoute connected to the VNet
- Azure Cloud Shell with VNet integration

## Known Issues

| Issue | Impact | Status |
|-------|--------|--------|
| `legionservicelink` on agent-subnet cannot be removed by user after partial deployment | Subnet permanently blocked; must use new VNet on retry | Open — requires Container Apps service team |
| Bicep warning BCP037: `capabilityHostKind` not in type library | Cosmetic — ARM accepts the property, deploy succeeds | Pending Bicep type update |
| Hardcoded `core.windows.net` in template | Blocks sovereign cloud deployments (Azure Gov, China) | Open — should use `environment().suffixes.storage` |

## References

- [Azure AI Foundry Networking](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-private-link?tabs=azure-portal&pivots=fdp-project)
- [Azure AI Foundry RBAC](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/rbac-azure-ai-foundry?pivots=fdp-project)
- [Standard Agent Setup (public network)](standard-agent-setup.md)

