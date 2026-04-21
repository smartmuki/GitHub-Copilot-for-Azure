# Post-Deployment Validation

Run after deployment succeeds. Verifies infrastructure, sets up access, and confirms end-to-end connectivity.

## 1. Infrastructure Verification

### 1.1 Resource State

Verify all resources are in `Succeeded` state:

```bash
az deployment operation group list \
  --resource-group <rg> --name <deployment-name> \
  --query "[].{resource:properties.targetResource.resourceType,state:properties.provisioningState}" -o table
```

### 1.2 Private Endpoint Connections

Verify all PE connections are `Approved`:

```bash
az network private-endpoint list \
  --resource-group <rg> \
  --query "[].{name:name,status:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status,resource:privateLinkServiceConnections[0].groupIds[0]}" -o table
```

### 1.3 DNS Resolution

From a machine connected to the VNet, verify each endpoint resolves to a private IP:

```bash
nslookup <ai-account-name>.services.ai.azure.com
nslookup <cosmos-account>.documents.azure.com
nslookup <storage-account>.blob.core.windows.net
nslookup <search-service>.search.windows.net
```

Each should resolve to a `10.x`, `172.16-31.x`, or `192.168.x` address — not a public IP.

### 1.4 Public Network Access Audit

Verify all resources have public access disabled:

```bash
az cognitiveservices account show --name <ai-account> --resource-group <rg> \
  --query "properties.publicNetworkAccess" -o tsv

az cosmosdb show --name <cosmos-account> --resource-group <rg> \
  --query "publicNetworkAccess" -o tsv

az storage account show --name <storage-account> --resource-group <rg> \
  --query "publicNetworkAccess" -o tsv

az search service show --name <search-service> --resource-group <rg> \
  --query "publicNetworkAccess" -o tsv
```

All should return `Disabled`.

> **T10 (Private Basic):** Steps 2–5 below do not apply — T10 has no agents, no capability host, and no BYO resources (Cosmos/Storage/Search). After infrastructure verification, the setup is complete.

## 2. Access Setup

> ⚠️ With `publicNetworkAccess: Disabled`, the agent endpoint is **not reachable from outside the VNet** — even with correct credentials. All management must happen from within the VNet.

Use `AskUserQuestion` to determine how the user will access private resources:

**"How do you want to access and test your private Foundry resources?"**
Options:
- `I already have VNet access` — user has Bastion, VPN, or ExpressRoute set up. Proceed to Step 3.
- `Set up a point-to-site VPN for me` — create a VPN Gateway with point-to-site configuration so the user can connect from their machine.
- `I'll set up access myself later` — skip connectivity testing. Complete Steps 3-4 (RBAC + model deploy) and inform user that Step 5 (end-to-end test) requires VNet access.

If user chose **"Set up a point-to-site VPN for me"**, create a VPN Gateway with point-to-site on the VNet before proceeding. This enables the user to connect directly from their dev machine to test the deployment.

## 3. RBAC Role Assignment

The template does not assign this automatically. Required to create or manage agents:

```bash
az role assignment create \
  --role "Azure AI Developer" \
  --assignee <your-object-id-or-email> \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<ai-account-name>
```

## 4. Deploy a Model

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

## 5. End-to-End Test

Create a test agent and invoke it to confirm the full setup works.

- Token audience: `https://ai.azure.com` (not `https://cognitiveservices.azure.com/`)
- Agent endpoint: `https://<account>.services.ai.azure.com/api/projects/<project>/assistants`
- Must be run from within the VNet (via the access method set up in step 2)
