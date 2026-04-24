# Information Gathering

Use this reference to collect requirements for a Foundry private-network deployment. Run sequentially — do not skip sub-steps.

## 1.0 Verify Subscription

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

## 1.1 Extract Known Answers

Before asking questions, scan the user's message for implicit answers:

| User Says | Inferred Answer |
|-----------|----------------|
| "my existing VNet" / "my VNet" | BYO VNet |
| "managed virtual network" | Managed VNet |
| "user-assigned identity" / "UAI" | User-assigned identity |
| "APIM" / "API Management" | Needs APIM |
| "MCP servers on the VNet" | Needs MCP subnet |

## 1.2 Ask Architecture Questions

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

## 1.3 Collect Deployment Inputs

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

## 1.4 Validate Region Consistency

All Foundry resources must be in the same region as the VNet. If existing resources were provided (VNet, resource group, Cosmos DB, Storage, AI Search), verify they are all in the same region using `az` CLI. If there's a mismatch, inform the user and resolve before proceeding.

## 1.5 Match to Template

Map the collected requirements to the best available deployment pattern:

| Deployment Pattern | Internal ID | Match When |
|-------------------|-------------|-----------|
| **Private Basic** (no agents) | T10 | No agents, private endpoints only |
| **Private Network Standard** | T15 | BYO VNet, agents, system-assigned identity |
| **Private Network + APIM Integration** | T16 | BYO VNet, agents, APIM integration (**Preview**) |
| **Private Network + User-Assigned Identity** | T17 | BYO VNet, agents, user-assigned identity |
| **Managed Virtual Network** | T18 | Managed VNet, agents (**Preview**) |
| **Hybrid Private Resources** | T19 | BYO VNet, agents, MCP servers on VNet |

> 💡 **User-facing language:** When presenting the match to the user, use the **Deployment Pattern** name (e.g., "Private Network Standard"), not the internal ID (e.g., T15). The IDs are for agent reference only.

Two possible outcomes:
1. **Exact match** — deploy the official template
2. **No match** — inform the user that no official template covers this combination and suggest the closest alternative

Present the match to the user using the deployment pattern name and explain why it was selected based on their answers. Ask: **"Does this match your requirements?"**

> Do NOT proceed until the user confirms.
