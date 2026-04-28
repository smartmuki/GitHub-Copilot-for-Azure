# Findings: Microsoft Foundry Skill — VNet Setup Gap Analysis

> **Date:** April 15, 2026
> **Method:** Static analysis of skill documents against test plan criteria (Phases 1–3 deep, Phases 4–5 doc-level assessment)
> **Skill version:** 1.0.10

---

## Executive Summary

The skill has a **solid routing tree** for selecting the correct VNet template (Phase 1) but lacks structured workflows for **requirement gathering** (Phase 2), **architecture planning** (Phase 3), and **post-deployment validation** (Phase 5). Code generation (Phase 4) is template-referral-based and mostly sufficient, but has gaps around existing IaC and Managed VNet CLI steps.

| Phase | Score | Verdict |
|-------|-------|---------|
| Phase 1 — Network Model Selection | 7/8 PASS | Strong — routing tree works, minor UX gaps |
| Phase 2 — Requirement Gathering | 1/8 PASS | Weak — no structured VNet requirement workflow |
| Phase 3 — Architecture Planning | 2/7 PASS | Weak — no planning sub-skill, missing resource enumeration |
| Phase 4 — IaC Generation | N/A (doc assessment) | Moderate — template refs exist, parameterization guidance missing |
| Phase 5 — Validation | N/A (doc assessment) | Missing — no validation workflow exists |

---

## Phase 1: Network Model Selection — Detailed Findings

### Test Case Results

| TC | Prompt Scenario | Result | Notes |
|----|----------------|--------|-------|
| P1-TC01 | Ambiguous "private network" | **PASS** | Routing Q3 distinguishes BYO vs Managed |
| P1-TC02 | Explicit "my existing VNet" | **PARTIAL** | Routing exists but no instruction to detect implicit intent and skip questions |
| P1-TC03 | Explicit "managed VNet" | **PASS** | Preview flag + limitations documented |
| P1-TC04 | "Default networking" | **PASS** | Q2→No routes to standard-agent-setup.md |
| P1-TC05 | "Private, no agents" | **PASS** | Q1→No routes to private-network-basic.md |
| P1-TC06 | "BYO VNet + APIM" | **PASS** | Q5→Yes routes to template 16 |
| P1-TC07 | "BYO VNet + UAI" | **PASS** | Q6→Yes routes to template 17 |
| P1-TC08 | "MCP on VNet" | **PASS** | Q4→Yes routes to template 19 |

### Gaps Found

| ID | Gap | Severity | Details |
|----|-----|----------|---------|
| P1-G1 | No conversation workflow for routing questions | **High** | Routing tree says "ask these questions in order" but gives no guidance on phrasing, expected response formats, or how to handle ambiguous answers. Agent must improvise the conversation. |
| P1-G2 | No confirmation gate after template selection | **High** | No instruction to summarize the chosen template and ask user "Is this what you want?" before proceeding. User could end up on wrong path silently. |
| P1-G3 | No implicit-intent detection | **Medium** | If user says "my existing VNet" the agent should skip the BYO-vs-Managed question. No rules for recognizing implied answers. |
| P1-G4 | No tradeoff presentation strategy | **Medium** | Each template doc has "When to Use / When NOT to Use" but no guidance on WHEN to surface tradeoffs (proactively? only if asked?). Risk of info-dumping. |
| P1-G5 | Preview status not in routing question text | **Low** | Templates 16 and 18 are Preview but the routing questions (Q3, Q5) don't mention that. User could select a Preview template without being warned. |

---

## Phase 2: Requirement Gathering — Detailed Findings

### Test Case Results

| TC | Prompt Scenario | Result | Notes |
|----|----------------|--------|-------|
| P2-TC01 | Minimal "Deploy Foundry with BYO VNet" | **PARTIAL** | Routing questions exist, but no structured list of VNet-specific inputs to collect |
| P2-TC02 | Region validation | **PARTIAL** | create-foundry-resource.md says "always ask for region"; quota skill has region checks; but no VNet-specific region validation |
| P2-TC03 | Complete spec — skip redundant Qs | **PARTIAL** | Generic "collect only missing values" principle in SKILL.md but not applied to VNet context |
| P2-TC04 | Subnet details | **PASS** | Template docs clearly specify /24 CIDRs, delegation, and same-region constraint |
| P2-TC05 | Existing IaC | **GAP** | All templates say "Always use the official Bicep template" — no guidance for modifying existing customer IaC |
| P2-TC06 | Agent workload types | **PARTIAL** | Agent types documented (Prompt vs Hosted) but not collected during Phase 2; hosted agents need extra infra (ACR, capability host) |
| P2-TC07 | DNS and on-prem connectivity | **GAP** | DNS zones documented as template outputs, never as discovery questions. On-prem/VPN mentioned only in template access notes, not systematically gathered |
| P2-TC08 | APIM + MCP conflict | **PARTIAL** | Routing handles them as sequential questions (Q4 then Q5) but no guidance when user needs BOTH — no combined template exists |

### Gaps Found

| ID | Gap | Severity | Details |
|----|-----|----------|---------|
| P2-G1 | No VNet requirement gathering workflow | **Critical** | No single document defines "here are all the inputs you must collect for a VNet deployment." Agent must read 8+ template docs and infer what to ask. This is the biggest gap in the entire skill. |
| P2-G2 | No structured input schema | **Critical** | No checklist, table, or JSON schema defining required vs. optional fields per template. Compare to `agent-metadata.yaml` schema (well-defined) — VNet setup has no equivalent. |
| P2-G3 | No existing IaC handler | **High** | Enterprise customers with existing Bicep/Terraform have no documented path. Skill says "use official template" with no modification workflow. |
| P2-G4 | DNS discovery missing | **High** | Private DNS zones are created by the template but the skill never asks: "Do you have custom DNS?" or "How does on-prem resolve private endpoints?" This will break hub-spoke customers. |
| P2-G5 | On-prem/VPN connectivity not asked | **High** | VPN Gateway, ExpressRoute, and Azure Bastion are mentioned as access methods but never collected during requirements. Customer discovers the gap post-deployment. |
| P2-G6 | Region validation not VNet-aware | **Medium** | Quota skill checks model quotas per region but doesn't validate VNet-related constraints (e.g., Container Apps availability for agent subnet delegation). |
| P2-G7 | No "stop before planning" gate | **Medium** | No instruction telling the agent "do NOT generate Bicep until all requirements are collected." Agent may jump ahead. |
| P2-G8 | APIM + MCP combination unhandled | **Medium** | Routing treats Q4 (MCP) and Q5 (APIM) as sequential, but if both are Yes, there's no combined template. Agent has no guidance for this edge case. |

---

## Phase 3: Architecture Planning — Detailed Findings

### Test Case Results

| TC | Prompt Scenario | Result | Notes |
|----|----------------|--------|-------|
| P3-TC01 | BYO VNet plan — resource enumeration | **PARTIAL** | Template 15 doc references Bicep template and describes subnets but does NOT have a "What It Creates" section (only template 10 has one) |
| P3-TC02 | Managed VNet — preview warnings | **PASS** | Template 18 doc includes feature flag, no-upgrade-path, async provisioning warnings |
| P3-TC03 | Hybrid — MCP subnet | **PASS** | Template 19 doc lists all 3 subnets correctly |
| P3-TC04 | RBAC prerequisites | **GAP** | Only standard-agent-setup.md mentions RBAC prereqs. Templates 15–19 reference docs do NOT include RBAC sections. rbac.md exists but is a separate skill, not integrated into planning. |
| P3-TC05 | DNS zones | **PARTIAL** | Template 10 lists 3 DNS zones. Templates 15–19 do NOT list which DNS zones are created. |
| P3-TC06 | Deployment order | **GAP** | No template doc describes resource deployment order or dependency graph. All say "use the Bicep template" and assume it handles ordering. |
| P3-TC07 | Confirm before code generation | **GAP** | No workflow step exists between plan and code generation. No documented "review this plan" checkpoint. |

### Gaps Found

| ID | Gap | Severity | Details |
|----|-----|----------|---------|
| P3-G1 | No planning sub-skill or workflow | **Critical** | No `plan.md` or planning phase exists. SKILL.md routes directly from template selection to deployment. The entire concept of "generate and review a plan" is undocumented. |
| P3-G2 | "What It Creates" section missing from T15–T19 | **Critical** | Only template 10 (Basic) enumerates resources. Templates 15–19 say "use the Bicep template" without listing what will be provisioned. Agent cannot tell the user what resources will be created. |
| P3-G3 | No deployment ordering | **High** | No doc describes: VNet first → resources → PEs → DNS → capability host → model → agent. Agent must infer dependency order or rely on Bicep's built-in ordering (which users can't review). |
| P3-G4 | No user confirmation checkpoint | **High** | No instruction to present plan and ask "Does this look correct?" before proceeding. Phase 3 → Phase 4 transition is seamless but uncontrolled. |
| P3-G5 | RBAC not integrated into planning | **High** | RBAC role requirements (Owner/UAA, Storage Blob Data Contributor, etc.) exist in rbac.md but are not referenced in any template's planning section. Customer hits permission errors at deploy time. |
| P3-G6 | DNS zones incomplete across templates | **Medium** | Template 10 lists DNS zones. T15–T19 don't list them. Agent can't enumerate: `privatelink.documents.azure.com` (Cosmos), `privatelink.blob.core.windows.net` (Storage), `privatelink.search.windows.net` (AI Search), etc. |
| P3-G7 | No architecture diagram guidance | **Low** | Test plan mentions "optionally produce a high-level architecture diagram." No skill doc provides ASCII diagrams, Mermaid templates, or textual architecture descriptions per template. |

---

## Phase 4: IaC Generation — Document-Level Assessment

> **Note:** Cannot fully validate without deploying to actual Azure resources. Assessment is based on what the skill docs provide.

### What the Skill DOES Cover

| Capability | Status | Evidence |
|------------|--------|----------|
| Official Bicep template references | ✅ | Every template doc links to `github.com/microsoft-foundry/foundry-samples/...` |
| Deployment command (`az deployment group create`) | ✅ | private-network-basic.md includes full command |
| Async capability host warning | ✅ | All private templates mention 10–20 min async provisioning |
| Post-deployment model deployment | ✅ | All templates include "Deploy a model" + GlobalStandard→Standard fallback |
| Post-deployment agent creation | ✅ | All templates reference `agent_update` MCP tool or Python SDK |

### Gaps Found

| ID | Gap | Severity | Details |
|----|-----|----------|---------|
| P4-G1 | No parameterization guidance | **High** | Template docs link to Bicep repos but don't document which parameters to customize (VNet CIDR, subnet names, resource names, region). User must read the raw Bicep to figure out params. |
| P4-G2 | No existing IaC modification workflow | **High** | (Same as P2-G3) All docs say "use the official template." No guidance for merging Foundry infra into customer's existing Bicep/Terraform. |
| P4-G3 | Managed VNet `az rest` commands not inline | **Medium** | Template 18 doc says "see `update-outbound-rules-cli/` in the template folder" for outbound rules but doesn't include the actual commands. Agent must fetch from GitHub. |
| P4-G4 | No idempotency guidance | **Medium** | No doc mentions whether re-running the Bicep template is safe (it should be, but this isn't documented for customer confidence). |
| P4-G5 | APIM parameter not documented inline | **Medium** | Template 16 doc mentions `apiManagementResourceId` param but doesn't show the full deployment command with it. |

---

## Phase 5: Validation — Document-Level Assessment

> **Note:** This is the weakest phase. No validation sub-skill or workflow exists anywhere in the skill.

### What's Present

| Item | Status |
|------|--------|
| Resource verification commands | ❌ None in template docs |
| Private endpoint verification | ❌ None |
| DNS resolution checks | ❌ None |
| Dependent service reachability | ❌ None |
| Requirements cross-check | ❌ None |
| Misconfiguration detection | ❌ None |
| End-to-end agent test | ⚠️ Implied by "create the agent" post-deployment step but not structured as validation |

### Gaps Found

| ID | Gap | Severity | Details |
|----|-----|----------|---------|
| P5-G1 | No validation sub-skill | **Critical** | No `validate.md`, no validation workflow, no post-deployment checklist. Entire Phase 5 is absent from the skill. |
| P5-G2 | No PE verification commands | **High** | No `az network private-endpoint list` or `az network private-endpoint-connection list` guidance. |
| P5-G3 | No DNS validation | **High** | No `nslookup` or `Resolve-DnsName` checks for private endpoint DNS resolution. |
| P5-G4 | No requirements cross-check | **High** | No workflow to compare "what was requested" vs "what was deployed." |
| P5-G5 | No `publicNetworkAccess` audit | **Medium** | No check to verify all resources have public access disabled in private setups. |

---

## Consolidated Gap Summary — Prioritized by Impact

### Critical (Blocks customer self-service)

| ID | Gap | Phase | Recommended Fix |
|----|-----|-------|-----------------|
| P2-G1 | No VNet requirement gathering workflow | 2 | Create `vnet-setup/requirements.md` — structured input checklist per template |
| P2-G2 | No structured input schema | 2 | Add requirements table with required/optional fields per template |
| P3-G1 | No planning sub-skill | 3 | Create `vnet-setup/plan.md` — planning workflow with resource enumeration + confirmation gate |
| P3-G2 | "What It Creates" missing from T15–T19 | 3 | Add resource enumeration section to each template doc |
| P5-G1 | No validation sub-skill | 5 | Create `vnet-setup/validate.md` — post-deployment checklist with commands |
| P1-G1 | No conversation workflow for routing | 1 | Add question phrasing, expected responses, and implicit-intent rules to routing section |

### High (Causes deployment failures or rework)

| ID | Gap | Phase | Recommended Fix |
|----|-----|-------|-----------------|
| P1-G2 | No confirmation gate after template selection | 1 | Add "Confirm: [Template X]. Proceed? (Y/N)" step to routing |
| P2-G3 | No existing IaC handler | 2 | Add workflow for modifying customer Bicep/Terraform |
| P2-G4 | DNS discovery missing | 2 | Add DNS questions to requirement gathering |
| P2-G5 | On-prem/VPN not asked | 2 | Add connectivity questions to requirement gathering |
| P3-G3 | No deployment ordering | 3 | Add dependency graph to each template doc |
| P3-G4 | No user confirmation checkpoint | 3 | Add "review plan" step before code generation |
| P3-G5 | RBAC not in planning | 3 | Add RBAC prereqs to each template doc or planning workflow |
| P4-G1 | No parameterization guidance | 4 | Document key parameters per template |
| P4-G2 | No existing IaC workflow | 4 | Guide for merging Foundry infra into customer templates |
| P5-G2 | No PE verification | 5 | Add `az network private-endpoint list` commands |
| P5-G3 | No DNS validation | 5 | Add DNS resolution checks |
| P5-G4 | No requirements cross-check | 5 | Add comparison checklist |

### Medium

| ID | Gap | Phase | Recommended Fix |
|----|-----|-------|-----------------|
| P1-G3 | No implicit-intent detection | 1 | Add patterns (e.g., "my VNet" → skip BYO question) |
| P1-G4 | No tradeoff presentation strategy | 1 | Add rules for when to surface template comparisons |
| P2-G6 | Region validation not VNet-aware | 2 | Add Container Apps region availability check |
| P2-G7 | No "stop before planning" gate | 2 | Add explicit instruction to complete requirements first |
| P2-G8 | APIM + MCP unhandled | 2 | Add conflict resolution path or combined template guidance |
| P3-G6 | DNS zones incomplete across templates | 3 | List all DNS zones per template |
| P4-G3 | Managed VNet `az rest` not inline | 4 | Embed outbound rule commands in template doc |
| P4-G4 | No idempotency guidance | 4 | Add note about safe re-runs |
| P4-G5 | APIM param not fully shown | 4 | Add complete deployment command example |
| P5-G5 | No publicNetworkAccess audit | 5 | Add resource property verification |

### Low

| ID | Gap | Phase |
|----|-----|-------|
| P1-G5 | Preview status not in routing text | 1 |
| P3-G7 | No architecture diagram guidance | 3 |

---

## Recommended Next Steps — Work Items

### Workstream 1: VNet Setup Sub-Skill (addresses 60% of gaps)

Create a new sub-skill folder `plugin/skills/microsoft-foundry/vnet-setup/` with:

| File | Purpose | Addresses |
|------|---------|-----------|
| `requirements.md` | Structured requirement gathering workflow with input schema per template | P2-G1, P2-G2, P2-G4, P2-G5, P2-G7 |
| `plan.md` | Architecture planning workflow with resource enumeration, dependency graph, confirmation gate | P3-G1, P3-G3, P3-G4 |
| `validate.md` | Post-deployment validation checklist with CLI commands | P5-G1, P5-G2, P5-G3, P5-G4, P5-G5 |

### Workstream 2: Template Doc Enrichment

Update each template reference doc (T15, T16, T17, T18, T19) to add:

| Section | Addresses |
|---------|-----------|
| "What It Creates" — full resource inventory | P3-G2 |
| "Required DNS Zones" — enumerated list | P3-G6 |
| "RBAC Prerequisites" — inline (not cross-ref) | P3-G5 |
| "Key Parameters" — params to customize with descriptions | P4-G1 |
| "Deployment Order" — numbered steps | P3-G3 |

### Workstream 3: Routing Logic Enhancement

Update `SKILL.md` § "Agent: Setup References" to add:

| Enhancement | Addresses |
|-------------|-----------|
| Question phrasing and expected responses | P1-G1 |
| Implicit-intent detection rules | P1-G3 |
| Confirmation gate after routing | P1-G2 |
| Preview annotations in routing questions | P1-G5 |
| Tradeoff presentation strategy | P1-G4 |

### Workstream 4: Existing IaC Support

New reference doc covering:

| Content | Addresses |
|---------|-----------|
| Approach for modifying customer Bicep/Terraform | P2-G3, P4-G2 |
| Required parameters to inject for each template | P4-G1 |
| Guidance for BYO resources (existing Cosmos DB, Storage, AI Search) | P2-G3 |

---

## Additional Findings — Networking Deep Dive

The following gaps were identified through a customer call involving deeper analysis of VNet sizing, NSG configuration, and subnet exclusivity constraints. These are not covered in any template doc.

### VNet Sizing — No Guidance Exists

The skill docs specify `/24` per subnet but provide **no guidance on overall VNet sizing based on workload**. Key gaps:

| Gap ID | Gap | Severity | Details |
|--------|-----|----------|---------|
| NET-G1 | No VNet sizing recommendations | **High** | Templates default to `192.168.0.0/16` but enterprise customers with existing networks need to carve address space. No guidance on how many IPs are needed or why `/24` is the minimum. |
| NET-G2 | No multi-account VNet planning | **High** | Each Foundry account needs its own exclusive agent subnet. If a customer plans to run multiple Foundry accounts on the same VNet, they need to plan for multiple `/24` agent subnets — this is never mentioned. |
| NET-G3 | PE subnet oversized with no explanation | **Low** | PE subnet is `/24` (254 usable IPs) but a standard Foundry setup only creates 4–6 private endpoints. A `/27` or `/28` would suffice. No sizing rationale is provided. |

**Sizing reference (not in skill docs):**

| Template | Subnets | Min VNet Size | Breakdown |
|----------|---------|---------------|-----------|
| Private Basic (no agents) | 1 (PE) | `/24` | 1× `/24` PE subnet |
| Private Standard (T15) | 2 | `/23` | 1× `/24` agent + 1× `/24` PE |
| Private + APIM (T16) | 2 | `/23` | Same as T15 |
| Private + UAI (T17) | 2 | `/23` | Same as T15 |
| Hybrid + MCP (T19) | 3 | `/22` | 1× `/24` agent + 1× `/24` PE + 1× `/24` MCP |
| Managed VNet (T18) | 1 (PE) | `/24` | Azure manages agent subnet internally |

### NSG / Security Group Configuration — Almost Entirely Missing

| Gap ID | Gap | Severity | Details |
|--------|-----|----------|---------|
| NET-G4 | No NSG rules documented for agent subnet | **Critical** | The BYO VNet comparison table (T18) says "Customer controls NSGs" for BYO templates, but no BYO template doc specifies what NSG rules are required. Enterprise customers with existing NSG policies will hit connectivity failures post-deployment. |
| NET-G5 | No Container Apps networking requirements referenced | **High** | The agent subnet is delegated to `Microsoft.App/environments`, which has [specific firewall/NSG requirements](https://learn.microsoft.com/en-us/azure/container-apps/firewall-integration). None of the skill docs reference this. |
| NET-G6 | No outbound dependency list for BYO VNet | **High** | For BYO VNet templates, required outbound connectivity (Azure management plane, DNS, Azure services) is never documented. Only Managed VNet (T18) mentions outbound rules (via `az rest`). |
| NET-G7 | No inbound rule guidance | **Medium** | No documentation on what inbound traffic the agent subnet needs (VNet-internal traffic, Azure Load Balancer probes, etc.). |

### Agent Subnet Exclusivity — Constraints Not Fully Explained

| Gap ID | Gap | Severity | Details |
|--------|-----|----------|---------|
| NET-G8 | Exclusivity constraint not explained | **Medium** | Docs say "Agent subnet must be exclusive to one Foundry account" but don't explain that this is a Container Apps delegation constraint — no other container services (ACI, AKS, other Container Apps environments) can run on this subnet. |
| NET-G9 | No guidance for additional container workloads | **Medium** | Customers needing MCP servers, custom APIs, or other container workloads alongside Foundry have no guidance on subnet planning. Only T19 (Hybrid) addresses MCP with a separate subnet. |

### Recommended Additions

These gaps should be addressed in **Workstream 2 (Template Doc Enrichment)** by adding to each BYO VNet template:

| Section | Content |
|---------|---------|
| **"VNet Sizing"** | Minimum VNet size, per-subnet sizing rationale, multi-account planning guidance |
| **"NSG Requirements"** | Required inbound/outbound rules per subnet, link to Container Apps firewall requirements |
| **"Subnet Constraints"** | Explain delegation exclusivity, no shared container workloads, one Foundry account per agent subnet |
