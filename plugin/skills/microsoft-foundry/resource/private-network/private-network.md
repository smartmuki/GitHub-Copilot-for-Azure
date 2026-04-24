---
name: private-network
description: "Answer questions about and deploy Microsoft Foundry with network isolation. Covers BYO VNet, Managed VNet, hybrid patterns, private endpoints, and Bicep deployment. WHEN: 'Foundry networking', 'BYO VNet vs managed VNet', 'deploy Foundry in private VNet', 'private endpoints for Foundry'. DO NOT USE FOR: generic Azure networking without Foundry (use azure-enterprise-infra-planner)."
license: MIT
allowed-tools: Read, Write, Bash, AskUserQuestion, microsoft_docs_search, microsoft_docs_fetch
metadata:
  author: Microsoft
  version: "1.1.0"
---

# Microsoft Foundry Private Networking

Answer questions about and deploy Microsoft Foundry with network isolation. Covers architecture concepts, deployment patterns (BYO VNet, Managed VNet, Hub-Spoke, hybrid), and end-to-end Bicep deployment.

## Quick Reference

| Property | Value |
|----------|-------|
| **Best for** | Foundry with VNet isolation, private endpoints, subnet delegation, APIM + Foundry, VPN/Bastion access |
| **Tools** | Azure CLI (`az deployment group create`, `az deployment group what-if`), `AskUserQuestion`, `microsoft_docs_search`, `microsoft_docs_fetch` |
| **Workflow** | Ground in Learn → Gather → Plan → Scaffold → Validate → Deploy → Test |

## When to Use

- User has general questions regarding Foundry in isolated network
- User needs Foundry with VNet isolation or private endpoints
- User wants to deploy Foundry agents in a private network
- User asks about BYO VNet, Managed VNet, or hybrid private networking
- User needs private endpoints for Cosmos DB, Storage, AI Search behind a VNet
- User needs APIM integration with private Foundry agents

**Do NOT use for:**
- Public Foundry setup without VNet → use [project/create](../../project/create/create-foundry-project.md)
- Bare Foundry resource without networking → use [resource/create](../create/create-foundry-resource.md)

## Workflow

---

## Step 0 — Ground Every Answer in Microsoft Learn

To answer user's questions about VNET configuration, private endpoints, NSP, or managed VNET for Microsoft Foundry:

- `microsoft_docs_search` — for targeted queries (returns up to 10 relevant excerpts, ~500 tokens each)
- `microsoft_docs_fetch` — for full pages when you need complete procedures or detailed reference

**Rules:** Cite the Learn URL in your answer. If Learn does not cover the question, say so explicitly — do not invent facts, limits, flags, or compatibility claims.

---

## End-to-End Deployment Workflow

> **Important:** When the user decides to deploy Foundry in an isolated network, all following steps are mandatory. Communicate the steps and plan with the user before acting.

## Step 1 — Information Gathering

Read [references/information-gathering.md](references/information-gathering.md). It covers subscription verification, architecture questions (VNet management, agents, MCP, APIM, identity), deployment inputs (region, resource group, VNet, BYO resources), region validation, and template matching.

After the user confirms the template match, load [requirement-gathering.md](requirement-gathering.md) to complete enterprise requirement intake before Plan Generation.

---

## Step 2 — Plan Generation

Use the confirmed Requirements Summary from [requirement-gathering.md](requirement-gathering.md) as the input for the plan. Incorporate VNet topology, DNS strategy, NSG/firewall rules, client access method, IaC path, and any blockers from the Requirements Summary into the plan.

Read the selected template's reference doc:

| Template | Reference |
|----------|-----------|
| T10 | [private-network-basic.md](../../references/private-network-basic.md) |
| T15 | [private-network-standard-agent-setup.md](../../references/private-network-standard-agent-setup.md) |
| T16 | [private-network-standard-agent-apim-setup.md](../../references/private-network-standard-agent-apim-setup.md) |
| T17 | [private-network-uai-agent-setup.md](../../references/private-network-uai-agent-setup.md) |
| T18 | [managed-virtual-network-agent-setup.md](../../references/managed-virtual-network-agent-setup.md) |
| T19 | [hybrid-private-resources-agent-setup.md](../../references/hybrid-private-resources-agent-setup.md) |

Always present to the user:
1. A text-based architecture overview showing VNet, subnets, private endpoints, resources, DNS zones, and access method — use the user's actual values
2. Resources that will be created
3. Deployment order: VNet + subnets → dependent resources → PEs + DNS → capability host → model
4. RBAC prerequisites (Owner, or Contributor + UAA)
5. DNS zones that will be created
6. Any known caveats (preview status, feature flag requirements)
7. Estimated deployment time

---

## Step 3 — Scaffold & Parameterize

Read [references/scaffold.md](references/scaffold.md). It covers template, parameter wiring in `main.bicepparam`, and adapt-vs-fetch handling based on IaC path and GitHub-access answers.

---

## Step 4 — Pre-Deployment Validation

Before deploying, check all of these. Catch blockers **before** investing in a deployment.

**Sovereign cloud:** Run `az cloud show --query name -o tsv`. If it returns `AzureUSGovernment` or `AzureChinaCloud`, warn the user only if using official templates: the official private-network Bicep templates hardcode `core.windows.net` and Azure Public AAD endpoints, and do not support sovereign clouds today. Stop and escalate.

**RBAC:** Verify deploying identity has Owner, or Contributor + User Access Administrator on the resource group.

**Azure Policy:** Run `az deployment group what-if` to catch policy violations (e.g., `disableLocalAuth`, `defaultOutboundAccess`) before actual deployment.

**Quota & Resource Availability:** Check resource availability in the target region before deploying (commands below work in both Bash and PowerShell):

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

**Feature Flags:** For Managed VNet — verify `AI.ManagedVnetPreview` is registered.

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
