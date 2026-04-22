# Post-Deployment Validation

Run after deployment succeeds. Steps 1-3 can run from anywhere (management plane). Steps 4-5 require VNet access.

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

### 1.3 Public Network Access Audit

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

> **T10 (Private Basic):** Steps 2-5 below do not apply — T10 has no agents, no capability host, and no BYO resources. Setup is complete after Step 1.

## 2. RBAC Role Assignment (no VNet required)

The template does not assign data-plane roles automatically.

Assign `Azure AI Developer` at the **account** scope (management-plane):

```bash
az role assignment create \
  --role "Azure AI Developer" \
  --assignee <your-object-id-or-email> \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<ai-account-name>
```

Assign `Azure AI User` at the **project** scope (data-plane — required for `agents/read`, `agents/write`):

```bash
az role assignment create \
  --role "Azure AI User" \
  --assignee <your-object-id-or-email> \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<ai-account-name>/projects/<project-name>
```

> ⚠️ RBAC propagation can take 1–5 minutes.

## 3. Deploy a Model (no VNet required)

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

---

## 4. VNet Access Setup

> ⚠️ The remaining tests (DNS resolution, agent lifecycle, isolation proof) require connectivity to the VNet. Steps 1-3 above are complete.

Use `AskUserQuestion`:

**"Steps 1-3 are done. The remaining tests need VNet access to verify DNS resolution and agent connectivity. How do you want to proceed?"**
Options:
- `I have a Bastion VM / jump box` — generate the test scripts as files, provide instructions to upload and run them on the VM
- `Set up a point-to-site VPN for me` — read [vpn-dns-setup.md](vpn-dns-setup.md) to deploy VPN + DNS resolver. After VPN connects, return here for Step 5
- `I have VPN / ExpressRoute already` — proceed to Step 5 directly
- `Skip testing for now` — inform user that DNS, agent lifecycle, and isolation tests are pending

**If user chose Bastion VM:**

The Bastion VM is inside the VNet — it has direct access to all private endpoints with no DNS or CORS issues. Inform the user:

- **Open `ai.azure.com` in the VM's browser** → full portal access to create and test agents
- **Run `az` CLI or Python SDK** directly on the VM for programmatic testing
- All private endpoints (AI Services, Cosmos DB, Storage, AI Search) are reachable natively from the VM

> **STOP here for Bastion users.** Do NOT proceed to Step 5. The agent cannot run tests on the Bastion VM — the user explores and validates directly on the VM. Setup is complete.

---

## 5. End-to-End Test (VPN users only)

> Skip this step if user chose Bastion VM (they can explore via portal directly) or skipped testing.\n\nPresent this test plan to the user using `AskUserQuestion`:

**"Ready to validate. The test covers 3 phases:"**
1. **Network** — DNS resolution + port 443 reachability for all private endpoints
2. **Agent Lifecycle** — Create agent, thread, run, verify response, cleanup (exercises all 4 PEs)
3. **Isolation Proof** — Same test with VPN off — should fail with 403

Options: `Run all phases` | `Skip for now`

> ⚠️ **VPN users — browser access:** Chromium-based browsers (Edge, Chrome) may bypass VPN DNS via Secure DNS (DNS-over-HTTPS). If the Foundry portal shows "Error loading agents" but CLI works, disable Secure DNS in browser settings or use an InPrivate/Incognito window.

### 5.0 Requirements

```bash
pip show azure-ai-agents azure-ai-projects azure-identity 2>/dev/null || \
  pip install azure-ai-projects azure-identity azure-ai-agents
```

### Phase 1: Network Validation

**What this proves:** Private DNS zones are correctly linked to the VNet and all private endpoints are reachable over the private network.

```powershell
$endpoints = @(
  '<ai-account>.services.ai.azure.com',
  '<ai-account>.openai.azure.com',
  '<ai-account>.cognitiveservices.azure.com',
  '<cosmos-account>.documents.azure.com',
  '<storage-account>.blob.core.windows.net',
  '<search-service>.search.windows.net'
)
foreach ($h in $endpoints) {
  $ip = (Resolve-DnsName $h | Where-Object {$_.IPAddress}).IPAddress
  $reach = Test-NetConnection $h -Port 443 -WarningAction SilentlyContinue
  Write-Host "$h -> $ip (reachable: $($reach.TcpTestSucceeded))"
}
```

All should resolve to `192.168.x.x` and be reachable.

**Report to user after running:**
- ✅ or ❌ per endpoint: hostname → IP (private or public?) + reachable (yes/no)
- If any resolve to public IPs → DNS zone linking issue
- If private IP but not reachable → NSG or firewall blocking

### Phase 2: Agent Lifecycle Test

**What this proves:** All 4 private endpoints work end-to-end for agent operations — AI Services (inference), Cosmos DB (thread storage), Storage (file uploads), AI Search (vector store). This is the definitive test that the deployment is functional.

```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

endpoint = "https://<ai-account>.services.ai.azure.com"
project = "<project-name>"
client = AIProjectClient(endpoint=endpoint, credential=DefaultAzureCredential(), project_name=project)
agents = client.agents

# 1. Create agent (AI Services PE)
agent = agents.create_agent(model="<deployment-name>", name="vnet-test", instructions="Reply with 'OK'")
# 2. Create thread (Cosmos DB PE)
thread = agents.threads.create()
# 3. Send message + run (AI Services PE)
agents.messages.create(thread_id=thread.id, role="user", content="test")
run = agents.runs.create_and_process(thread_id=thread.id, agent_id=agent.id)
# 4. Verify response
msgs = agents.messages.list(thread_id=thread.id)
print(f"Response: {msgs.data[0].content[0].text.value}")
# 5. Cleanup
agents.threads.delete(thread.id)
agents.delete_agent(agent.id)
```

**Report to user after each step:**
- ✅ `create_agent` — AI Services PE working
- ✅ `create_thread` — Cosmos DB PE working
- ✅ `send_message + run` — AI Services inference PE working
- ✅ `verify_response` — Model returned expected output
- ✅ `cleanup` — Resources deleted

If any step fails, report which PE is affected and the error message.

### Phase 3: Isolation Proof (VPN OFF)

**What this proves:** Network isolation is correctly enforced — even with valid credentials and correct RBAC, resources are unreachable from outside the VNet.

Ask user to disconnect VPN, then repeat Phase 2. It should **fail** on step 1 with 403 `PublicNetworkAccess disabled`.

**Report to user:**
- ✅ `create_agent` failed with 403 — **this is the expected result**. Network isolation is working.
- ❌ If it succeeds — `publicNetworkAccess` may not be disabled. Re-check Step 1.3.

### Summary

After all phases, present a final summary:

```
Deployment Validation Report
─────────────────────────────
Step 1: Infrastructure    ✅ Resources succeeded, PEs approved, public access disabled
Step 2: RBAC              ✅ Azure AI Developer (account) + Azure AI User (project)
Step 3: Model             ✅ <model-name> deployed (<sku>, <capacity> TPM)
Step 4: VNet Access       ✅ Connected via <method>
Step 5: Tests
  Phase 1 (Network)       ✅ All 6 FQDNs resolve to private IPs + reachable
  Phase 2 (Agent)         ✅ All 4 PEs exercised, agent responded correctly
  Phase 3 (Isolation)     ✅ Access correctly blocked without VPN
```

### Cleanup (VPN users only)

If a VPN Gateway and DNS Resolver were deployed in Step 4, ask:

Use `AskUserQuestion`: **"Testing is complete. The VPN Gateway (~$140/month) and DNS Resolver (~$180/month) are no longer needed for testing. Would you like me to delete them, or keep them for ongoing access?"**
Options: `Delete VPN + DNS resolver` | `Keep for ongoing access`

If delete:

```bash
az network vnet-gateway delete --resource-group <rg> --name vpn-gateway-<suffix> --no-wait
az network dns-resolver delete --resource-group <rg> --name dns-resolver-<suffix> --yes
az network public-ip delete --resource-group <rg> --name vpn-gateway-pip-<suffix>
```
