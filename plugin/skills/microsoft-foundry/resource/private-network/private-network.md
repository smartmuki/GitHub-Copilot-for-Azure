---
name: private-network
description: "Set up Microsoft Foundry with network isolation using official Bicep templates. Covers BYO VNet, Managed VNet, hybrid. WHEN: 'deploy Foundry in a private VNet', 'private endpoints for Foundry', 'managed virtual network Foundry', 'network isolation Foundry', 'BYO VNet Foundry'."
license: MIT
allowed-tools: Read, Write, Bash, AskUserQuestion
metadata:
  author: Microsoft
  version: "1.0.0"
---

# Private Network Infrastructure Setup

Deploy Microsoft Foundry with network isolation using official Bicep templates. Covers BYO VNet, Managed VNet, and hybrid configurations.

## Quick Reference

| Property | Value |
|----------|-------|
| **Best for** | Deploying Foundry with VNet isolation, private endpoints, subnet delegation |
| **Templates** | T10 (Basic), T15 (Standard), T16 (APIM), T17 (UAI), T18 (Managed), T19 (Hybrid) |
| **Tools** | Azure CLI (`az deployment group create`, `az deployment group what-if`), `AskUserQuestion` |
| **Steps** | Information Gathering → Plan → Scaffold → Validation → Deploy → Test |

## When to Use

- User needs Foundry with VNet isolation or private endpoints
- User wants to deploy Foundry agents in a private network
- User asks about BYO VNet, Managed VNet, or hybrid private networking
- User needs private endpoints for Cosmos DB, Storage, AI Search behind a VNet
- User needs APIM integration with private Foundry agents

**Do NOT use for:**
- Public Foundry setup without VNet → use [project/create](../../project/create/create-foundry-project.md)
- Bare Foundry resource without networking → use [resource/create](../create/create-foundry-resource.md)

## Workflow

Execute steps in order.

---

## Step 1 — Information Gathering

### 1.0 Verify Subscription

Verify the active Azure subscription before proceeding:

```bash
az account show --query "{Name:name, Id:id, State:state}" -o table
```

Use `AskUserQuestion`: **"Is this the correct subscription for deployment? If not, type the subscription name or ID to switch to."**
Options: `Yes, this is correct`

If the user provides a different subscription name or ID:

```bash
az account set --subscription "<name-or-id-provided-by-user>"
```

### 1.1 Extract Known Answers

Before asking questions, scan the user's message for implicit answers:

| User Says | Inferred Answer |
|-----------|----------------|
| "my existing VNet" / "my VNet" | BYO VNet |
| "managed virtual network" | Managed VNet |
| "user-assigned identity" / "UAI" | User-assigned identity |
| "APIM" / "API Management" | Needs APIM |
| "MCP servers on the VNet" | Needs MCP subnet |

### 1.2 Ask Architecture Questions

Use `AskUserQuestion` for unanswered items:

**Q1 — VNet Management:** "Who should manage the virtual network? BYO VNet = you provide and manage your own. Managed VNet = Azure creates and manages it (preview, requires feature flag)."
Options: `BYO VNet` | `Managed VNet (Preview)`

**Q2 — Agents:** "Do you need to run AI agent workloads (container-based agents that execute custom code), or just a private Foundry resource for models and projects?"
Options: `Yes - agent workloads` | `No - just models and projects`

**Q3 — MCP Servers:** "Do you need MCP servers on the VNet or optional public Foundry portal access alongside private backend?"
Options: `No` | `Yes`

**Q4 — APIM:** "Do you need Azure API Management (APIM) integration?"
Options: `No` | `Yes`

**Q5 — Identity:** "System-assigned managed identity (default) or user-assigned managed identity?"
Options: `System-assigned` | `User-assigned`

### 1.3 Collect Deployment Inputs

Ask for any of these not already provided:

**Q6 — Region:** "Which Azure region?"

**Q7 — Resource Group:** "Use an existing resource group or create a new one?"
Options: `Create new` | `Use existing` → ask for name

**Q8 — VNet:** "Create a new VNet or use an existing one?"
Options: `Create new` | `Use existing` → ask for VNet name/resource ID
- If new: VNet address space (default: `192.168.0.0/16`)
- Subnet CIDRs: agent subnet `/24`, PE subnet `/24`, MCP subnet `/24` (if Q3=Yes)

**Q9 — BYO Resources:** "Do you have existing Cosmos DB, Storage, or AI Search resources to reuse, or should the template create new ones?"
Options: `Create new` (recommended) | `Use existing` → ask for resource IDs

### 1.4 Validate Region Consistency

All Foundry resources must be in the same region as the VNet. If existing resources were provided (VNet, resource group, Cosmos DB, Storage, AI Search), verify they are all in the same region using `az` CLI. If there's a mismatch, inform the user and resolve before proceeding.

### 1.5 Match to Template

Map the collected requirements to the best available template:

| Template | Match When |
|----------|-----------|
| **T10** Private Basic | No agents, private endpoints only |
| **T15** Private Standard | BYO VNet, agents, system-assigned identity |
| **T16** Private + APIM | BYO VNet, agents, APIM integration (**Preview**) |
| **T17** Private + UAI | BYO VNet, agents, user-assigned identity |
| **T18** Managed VNet | Managed VNet, agents (**Preview**) |
| **T19** Hybrid Private | BYO VNet, agents, MCP servers on VNet |

Two possible outcomes:
1. **Exact match** — deploy the official template
2. **No match** — inform the user that no official template covers this combination and suggest the closest alternative

Present the match result to the user. Ask: **"Does this match your requirements?"**

> Do NOT proceed until the user confirms.

After template is confirmed, load [requirement-gathering.md](requirement-gathering.md) to complete enterprise requirement intake before Plan Generation.

---

## Step 2 — Plan Generation

Use the confirmed Requirements Summary from [requirement-gathering.md](requirement-gathering.md) as the input for the plan. Incorporate topology (R1), DNS (R3), NSG (R5), client access (R14-R15), IaC path (R17), and any blockers from Section 7 into the plan.

Read the selected template's reference doc:

| Template | Reference |
|----------|-----------|
| T10 | [private-network-basic.md](../../references/private-network-basic.md) |
| T15 | [private-network-standard-agent-setup.md](../../references/private-network-standard-agent-setup.md) |
| T16 | [private-network-standard-agent-apim-setup.md](../../references/private-network-standard-agent-apim-setup.md) |
| T17 | [private-network-uai-agent-setup.md](../../references/private-network-uai-agent-setup.md) |
| T18 | [managed-virtual-network-agent-setup.md](../../references/managed-virtual-network-agent-setup.md) |
| T19 | [hybrid-private-resources-agent-setup.md](../../references/hybrid-private-resources-agent-setup.md) |

Present to the user:
1. A text-based architecture overview showing VNet, subnets, private endpoints, resources, DNS zones, and access method — use the user's actual values
2. Resources that will be created
3. Deployment order: VNet + subnets → dependent resources → PEs + DNS → capability host → model
4. RBAC prerequisites (Owner, or Contributor + UAA)
5. DNS zones that will be created
6. Any known caveats (preview status, feature flag requirements)
7. Estimated deployment time

---

## Step 3 — Scaffold & Parameterize

Fetch the Bicep template from the GitHub URL in the template reference doc (loaded in Step 2). Fetch the **entire template folder** including subdirectories (e.g., `modules-network-secured/`). Create the files in the user's workspace (e.g., `infra/` folder).

> ⚠️ Do NOT download only `main.bicep` and `main.bicepparam` — the template depends on module files in subdirectories. Use the GitHub API contents endpoint to list and download all files recursively.

Set parameter values in `main.bicepparam` using answers from Step 1 and the Requirements Summary:

| Parameter | Source |
|-----------|--------|
| Location | Q6 (or inferred from existing VNet) |
| VNet name / resource ID | Q8 |
| VNet address space | R4 (or Q8 default `192.168.0.0/16`) |
| Subnet CIDRs | Q8 |
| Existing Cosmos DB / Storage / AI Search IDs | Q9 / R10 (if BYO) |
| Isolation mode (T18 only) | R1M-b (`AllowOnlyApprovedOutbound` or `AllowInternetOutbound`) |
| Model name, version, format | R12, R13 |
| `disableLocalAuth` | R19 (set to `true` if Azure Policy requires it) |

If R17 = `Adapt Bicep into existing repo`, copy the relevant resources into the customer's existing templates instead of using the official template directly.

If R18 = `No GitHub access`, the template must already be present in the workspace. Do NOT attempt to fetch from GitHub.

> Do NOT run `az deployment group create` yet — validate first.

---

## Step 4 — Pre-Deployment Validation

Before deploying, check all of these. Catch blockers **before** investing in a deployment.

**RBAC:** Verify deploying identity has Owner, or Contributor + User Access Administrator on the resource group.

**Azure Policy:** Run `az deployment group what-if` to catch policy violations (e.g., `disableLocalAuth`, `defaultOutboundAccess`) before actual deployment.

**Quota & Resource Availability:** Check resource availability in the target region before deploying:

```bash
# Model quota
az cognitiveservices usage list --location <region> -o table

# AI Search availability
az search service list-sku --location <region> -o table

# Check if region has capacity for Cognitive Services
az cognitiveservices account list-skus --location <region> --kind AIServices -o table
```

If you get `ResourcesForSkuUnavailable` or `InsufficientResourcesAvailable`, the region lacks capacity — suggest a different region to the user. Run `az deployment group what-if` to catch these before the actual deployment.

**Provider Registrations:** Verify `Microsoft.CognitiveServices`, `Microsoft.DocumentDB`, `Microsoft.Search`, `Microsoft.Network` are registered.

**Feature Flags:** For T18 (Managed VNet) — verify `AI.ManagedVnetPreview` is registered.

> Do NOT deploy until all pre-flight checks pass.

---

## Step 5 — Deploy & Track

Read [references/deploy.md](references/deploy.md) for deployment command, polling strategy, and error recovery. For expected resource progression specific to your template, refer to the template reference doc loaded in Step 2.

---

## Step 6 — Test & Validate

> **Detailed reference:** Read [references/post-deployment-validation.md](references/post-deployment-validation.md) for PE verification, DNS resolution checks, RBAC audit, `publicNetworkAccess` audit, requirements cross-check, and end-to-end agent test.

---

## Error Handling

| Error | Step | Remediation |
|-------|------|-------------|
| No matching template | 1 | Suggest closest template and explain what's different |
| Missing inputs | 1 | Ask missing questions before proceeding |
| `what-if` shows policy violation | 4 | Fix Bicep params (e.g., `disableLocalAuth: true`), re-run |
| Deployment fails mid-way | 5 | Use new VNet name on retry, see deploy.md error recovery |
| PE/DNS verification fails | 6 | Check resource provisioning state, re-run failed resources |

---

## Documentation Resources

| Topic | Link |
|-------|------|
| Network isolation overview | [Configure network isolation for Foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-private-link) |
| Agent Service private networking | [Set up private networking for Agent Service](https://learn.microsoft.com/en-us/azure/ai-services/agents/how-to/virtual-networks) |
| Managed VNet (classic) | [Configure managed virtual network](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-managed-network) |
| Agent Service FAQ — VNet | [Agent FAQ — Virtual Networking](https://learn.microsoft.com/en-us/azure/foundry/agents/faq#virtual-networking) |
