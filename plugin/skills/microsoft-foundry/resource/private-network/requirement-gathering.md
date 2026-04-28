# Requirement Gathering — Private Network Foundry Setup

> **When to read this file:** After template is confirmed in `private-network.md` Step 1.5, before Plan Generation (Step 2). Supplements the inputs already gathered (region, RG, VNet) with enterprise requirements.

Use `AskUserQuestion` for every unanswered item. Batch related questions.

---

## Path Decision

```
Network ownership (R0)
├── BYO VNet  (Standard / APIM Integration / User-Assigned Identity / Hybrid MCP)
│   ├── Section 1  — Network Architecture
│   └── Section 2  — Ownership & Access Boundaries
└── Managed VNet  (Preview — Azure-managed isolation)
    └── Section 1M — Managed VNet Requirements

Both paths → Section 3 → 4 → 5 → 6 → 7 → Confirmation
```

---

## Step 0 — BYO VNet vs Managed VNet

> ⚠️ **Warning:** This is the primary decision. It drives template selection and which sections below apply.

**R0 — Network ownership:**

| Option | What it means | Deployment patterns |
|--------|---------------|-----------|
| **BYO VNet** | You own and manage the VNet, subnets, DNS, NSG, and peering | Standard, APIM Integration, User-Assigned Identity, Hybrid MCP |
| **Managed VNet** | Microsoft manages VNet isolation; you do not configure a VNet | Managed VNet (Preview) |

> 💡 **Tip:** If you have an existing VNet or on-prem connectivity, BYO VNet is the right path. Managed VNet is simpler but is in preview and does not support on-prem inbound.

**If BYO VNet → go to Section 1**
**If Managed VNet → skip to Section 1M**

---

## Section 1 — BYO VNet Network Architecture

*Skip if Managed VNet (R0).*

**R1 — VNet topology:**
Options: `Standalone` | `Hub-and-spoke (peering required)` | `Azure vWAN` | `Unknown`

**R2 — On-prem / VPN / ExpressRoute connectivity:**
Options: `No` | `Yes — VPN Gateway` | `Yes — ExpressRoute` | `Not yet`

**R3 — DNS:**
Options: `Azure-provided DNS (default)` | `Custom DNS resolver` | `On-prem DNS forwarding`

> ⚠️ **Warning:** Custom or on-prem DNS requires private DNS zones linked to the resolver's VNet. Confirm this is possible before deploying.

**R4 — Address space:** Is `192.168.0.0/16` (default) available, or must a specific range be used?

**R5 — NSG / Firewall:** Existing NSG or Azure Firewall rules on the subnets?
Options: `No` | `Yes — customer provides rules` | `Yes — need guidance`

→ Continue to Section 2.

---

## Section 1M — Managed VNet Requirements

*Only for Managed VNet (Preview). Skip if BYO VNet (R0).*

**R1M-a — Feature flag:** Verify `AI.ManagedVnetPreview` is registered:

```bash
az feature show --namespace Microsoft.MachineLearningServices --name AI.ManagedVnetPreview
```

> ⚠️ **Warning:** If not registered, register it and wait 15–30 min before deploying.

```bash
az feature register --namespace Microsoft.MachineLearningServices --name AI.ManagedVnetPreview
az provider register --namespace Microsoft.MachineLearningServices
```

**R1M-b — Outbound access mode:**
Options: `Internet outbound (default)` | `Approved outbound only (enterprise recommended)`

**R1M-c — MCP servers:** *(Ask only if agents use MCP)*
Managed VNet supports public MCP endpoints only. If MCP servers must run on a private VNet, use BYO VNet Hybrid instead.

**R1M-d — Client access:** Managed VNet disables public access to the agent endpoint. All clients must connect from within Azure (same VNet, peered VNet, or private link from an Azure-hosted service).
Options: `Client is Azure-hosted — confirmed` | `Unknown — need guidance`

> ⚠️ **Warning:** If clients are on-prem and cannot connect via a peered Azure network, Managed VNet is not appropriate. Use BYO VNet instead.

→ Skip Section 2, continue to Section 3.

---

## Section 2 — Ownership & Access Boundaries

*BYO VNet only. Skip if Managed VNet (R0).*

**R6 — Deployment executor:** Where will `az` CLI and SDK validation commands run after deployment? Private endpoints block all management plane access from outside the VNet.
Options: `VM inside VNet` | `Azure Bastion jump box` | `VPN/ExpressRoute connected machine` | `Azure Cloud Shell (VNet integrated)` | `Unknown — need guidance`

**R7 — Subscription / tenant scope:**
Options: `Same subscription and tenant` | `Cross-subscription` | `Cross-tenant`

> ⚠️ **Warning:** Cross-tenant is not supported by any template. Cross-subscription requires additional role assignments for private DNS zone linking.

**R8 — Team ownership:** Who can modify the VNet, private DNS zones, NSG, and subscription policy — is it the same team making this request?
Options: `Same team` | `Different team — approvals needed` | `Unknown`

> ⚠️ **Warning:** If DNS or NSG are owned by a different team, block progress until changes are pre-approved. Deployment may succeed but endpoints will be unreachable without those changes.

---

## Section 3 — Agent Workloads & Tools

**R9 — Agent tools:** (Select all that apply)
Options: `Azure AI Search` | `Cosmos DB` | `Azure Storage` | `MCP servers` | `External REST APIs` | `Bing grounding` | `Code Interpreter` | `Not yet defined`

**R10 — Existing resources:** Reuse existing Cosmos DB / Storage / AI Search, or create new?
Options: `Create new (recommended)` | `Use existing — collect resource IDs`

If existing: confirm all resources are in the same region as the VNet.

**R11 — MCP location:** *(Ask only if R9 includes MCP)*
Options: `Public MCP endpoint` | `Private VNet MCP — requires BYO VNet Hybrid`

---

## Section 4 — Model Requirements

**R12 — Model name and vendor:** Ask for model name and vendor (OpenAI / Mistral AI / Meta / other).

**R13 — Model version:** Version formats differ by vendor:

| Vendor | Format | Example |
|--------|--------|---------|
| OpenAI | Date string | `2025-04-14` |
| Mistral AI | Integer | `1` |
| Meta (Llama) | Integer | `9` |

> ⚠️ **Warning:** Using an OpenAI-style date version for Mistral or Meta causes `DeploymentModelNotSupported`. Verify the exact version with `az cognitiveservices model list` before parameterizing.

---

## Section 5 — Consumer & Frontend Access

**R14 — Client type:** What will call the deployed agent?
Options: `SDK / Python client` | `Web application` | `Teams bot` | `Other service`

**R15 — Client network access:** With `publicNetworkAccess: Disabled`, clients must reach the agent via private network. Confirm the client's path:
Options: `Inside same VNet` | `Peered VNet` | `VPN/ExpressRoute connected network` | `Not confirmed — need guidance`

**R16 — Authentication:**
Options: `Entra ID (recommended)` | `API key`

> 💡 **Tip:** Entra ID token audience for the Foundry Agents API is `https://ai.azure.com` — not `https://cognitiveservices.azure.com`.

---

## Section 6 — Delivery & Operations

**R17 — IaC path:**
- `Official Bicep as-is` — fully supported, recommended
- `Adapt Bicep into existing repo` — supported, requires manual steps
- `Terraform` — no official template; customer must own the translation

**R18 — GitHub access:** Can the deployment environment reach `github.com` to fetch the Bicep template?
Options: `Yes` | `No — pre-stage template before this session`

**R19 — Azure Policy:** Known subscription policies (e.g., `disableLocalAuth: true`, `defaultOutboundAccess: false`)?
Options: `Unknown — run what-if to check` | `Yes — list them` | `No known policies`

**R20 — Monitoring:** Existing Log Analytics workspace to connect to?
Options: `Yes — provide workspace ID` | `No — create new` | `Not needed`

---

## Section 7 — Compatibility Gate

| Scenario | Status | Action |
|----------|--------|--------|
| APIM Integration + private MCP on same VNet | ❌ No supported pattern | Use BYO VNet + APIM Integration (public MCP only) or BYO VNet Hybrid (private MCP, no APIM) |
| Managed VNet without `AI.ManagedVnetPreview` registered | ❌ Blocked | Register feature flag and wait for `Registered` state |
| Cross-tenant resources | ❌ Not supported | Stop; inform user and halt |
| Terraform delivery | ⚠️ No official template | Proceed only if customer owns the translation |
| Air-gapped + no pre-staged template | ⚠️ Blocked | Pre-stage Bicep template before proceeding |

> ⚠️ **Warning:** If a ❌ blocker is found, stop here. Present the issue and options before continuing.

---

## Confirmation Checkpoint

Present this summary and ask: **"Confirm this is accurate before I generate a deployment plan."**

```
Requirements Summary
─────────────────────────────────────────────
Network path:        [BYO VNet / Managed VNet]
Deployment pattern:  [BYO VNet Standard / BYO VNet APIM / BYO VNet UAI / BYO VNet Hybrid MCP / Managed VNet]
Region:              <region>
Resource group:      <resource-group>

── BYO VNet ──────────────────────────────────
VNet:                [new / existing: <vnet-name>]
Address space:       <cidr>
DNS:                 [Azure-provided / custom]
On-prem:             [none / VPN / ExpressRoute]
Topology:            [standalone / hub-spoke / vWAN]
Deployment executor: <host>
Ownership:           [same team / approvals needed]

── Managed VNet ──────────────────────────────
Feature flag:        [registered / pending]
Outbound mode:       [internet / approved-only]

── Shared ────────────────────────────────────
Agent tools:         <list>
Model:               <vendor>, <name>, version <version>
Client type:         <client-type>
Client network path: <path>
Auth:                [Entra ID / API key]
IaC path:            [Bicep as-is / adapt / Terraform]
GitHub access:       [yes / no]
Policy constraints:  <list or none>
Monitoring:          [workspace: <workspace-id> / new / none]
Blockers:            <list or none>
─────────────────────────────────────────────
```

> Do NOT proceed to Plan Generation until the user confirms.
