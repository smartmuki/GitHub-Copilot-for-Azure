# Test Plan: Microsoft Foundry Skill — VNet Setup End-to-End

## Objective

Evaluate the existing `microsoft-foundry` skill for its ability to guide, plan, deploy, and validate an Azure AI Foundry setup in a Virtual Network (VNet). The end goal is to ensure this skill is production-ready for customer self-service, enabling customers to confidently deploy Foundry in their own Azure environments with minimal human intervention.

## Scope

This test plan validates five phases of the skill's VNet setup workflow:

| Phase | Focus |
|-------|-------|
| 1 — Requirement Discovery | Network model selection (BYO VNet vs Managed VNet) |
| 2 — Requirement Gathering | Resources, services, security constraints |
| 3 — Planning & Architecture | Deployment plan, resource graph, pattern mapping |
| 4 — IaC Code Generation | Bicep/Terraform output aligned with the plan |
| 5 — Validation | Post-deployment correctness checks |

### Non-Goals (Out of Scope)

- Cost/SKU optimization beyond correctness
- Non-Azure cloud providers
- Long-term operational monitoring or autoscaling

## Skill References Under Test

The following skill documents are the authoritative source material the skill must reference:

| Reference | Path |
|-----------|------|
| SKILL.md (root) | `plugin/skills/microsoft-foundry/SKILL.md` |
| Standard Agent Setup | `references/standard-agent-setup.md` |
| Private Network Basic | `references/private-network-basic.md` |
| Private Network Standard Agent Setup | `references/private-network-standard-agent-setup.md` |
| Private Network Standard + APIM | `references/private-network-standard-agent-apim-setup.md` |
| Private Network UAI | `references/private-network-uai-agent-setup.md` |
| Managed Virtual Network Agent Setup | `references/managed-virtual-network-agent-setup.md` |
| Hybrid Private Resources Agent Setup | `references/hybrid-private-resources-agent-setup.md` |

---

## Phase 1: Requirement Discovery — Network Model Selection

### Goal

Validate that the skill asks the right clarifying questions to determine the customer's preferred networking model.

### Skill Responsibilities

- Identify whether the customer wants **BYO VNet** or **Managed VNet**
- Explain tradeoffs at a high level only when needed (no upfront information dump)
- Follow the routing logic defined in `SKILL.md` § "Agent: Setup References"

### Test Cases

#### P1-TC01: Ambiguous VNet request triggers clarification

| Field | Value |
|-------|-------|
| **Prompt** | `"I want to deploy Foundry in a private network"` |
| **Expected behavior** | Skill asks whether the user wants to bring their own VNet or use an Azure-managed VNet. Does **not** assume either. |
| **Validation** | Response contains a question distinguishing BYO VNet vs Managed VNet. |

#### P1-TC02: Explicit BYO VNet — no needless follow-up on network model

| Field | Value |
|-------|-------|
| **Prompt** | `"Set up Foundry with my existing VNet in eastus2"` |
| **Expected behavior** | Skill accepts BYO VNet, does **not** re-ask about network model, and proceeds to resource gathering. |
| **Validation** | No redundant clarification about Managed vs BYO. Moves to Phase 2. |

#### P1-TC03: Explicit Managed VNet selection

| Field | Value |
|-------|-------|
| **Prompt** | `"Deploy Foundry using a managed virtual network"` |
| **Expected behavior** | Skill acknowledges Managed VNet, mentions preview status and `AI.ManagedVnetPreview` feature flag requirement, and proceeds. |
| **Validation** | Response references the managed VNet template and its preview limitations. |

#### P1-TC04: No VNet requirement — routes to public standard

| Field | Value |
|-------|-------|
| **Prompt** | `"Set up Foundry agents with default networking"` |
| **Expected behavior** | Skill identifies no VNet isolation is needed and routes to the Standard Agent Setup template. |
| **Validation** | Response references `standard-agent-setup.md` or its Bicep template. |

#### P1-TC05: No agents — routes to Private Network Basic

| Field | Value |
|-------|-------|
| **Prompt** | `"I need a private Foundry account for hosting models, no agents"` |
| **Expected behavior** | Skill routes to Private Network Basic template (template 10). |
| **Validation** | Response references `private-network-basic.md`. |

#### P1-TC06: Routing tree completeness — APIM requirement

| Field | Value |
|-------|-------|
| **Prompt** | `"Deploy Foundry agents in a private VNet with APIM integration"` |
| **Expected behavior** | Skill routes to Private Network Standard + APIM template (template 16). |
| **Validation** | Response references `private-network-standard-agent-apim-setup.md`. |

#### P1-TC07: Routing tree completeness — User-assigned identity

| Field | Value |
|-------|-------|
| **Prompt** | `"Set up Foundry with private networking and a user-assigned managed identity"` |
| **Expected behavior** | Skill routes to Private Network UAI Agent Setup (template 17). |
| **Validation** | Response references `private-network-uai-agent-setup.md`. |

#### P1-TC08: Routing tree completeness — MCP subnet / Hybrid

| Field | Value |
|-------|-------|
| **Prompt** | `"Deploy Foundry with private backend resources and MCP servers on the VNet"` |
| **Expected behavior** | Skill routes to Hybrid Private Resources Agent Setup (template 19). |
| **Validation** | Response references `hybrid-private-resources-agent-setup.md`. |

### Success Criterion

A clear, explicitly acknowledged network choice captured as structured input before the skill proceeds.

---

## Phase 2: Requirement Gathering — Resources and Constraints

### Goal

Validate that the skill can systematically gather all required inputs needed to provision Foundry correctly.

### Skill Responsibilities

Collect:
- Azure region (and validate availability constraints)
- Foundry components: AI Search, Storage, Cosmos DB, Key Vault, Application Insights, Agents (excluding Hosted Agents unless explicitly requested), APIM / MCP (if required), Foundry Evals (with or without agents)
- Security & networking constraints: private endpoints, DNS, on-prem/VPN access
- Existing infrastructure context: existing VNet or IaC, subscription & resource group scoping

### Test Cases

#### P2-TC01: Minimal prompt — skill elicits missing fields

| Field | Value |
|-------|-------|
| **Prompt** | `"Deploy Foundry with BYO VNet"` |
| **Expected behavior** | Skill asks for: region, resource group, VNet details (address space, subnets), which dependent services to provision (Cosmos DB, Storage, AI Search). |
| **Validation** | Response contains follow-up questions for at least region, resource group, and VNet configuration. |

#### P2-TC02: Region validation — unavailable region

| Field | Value |
|-------|-------|
| **Prompt** | `"Deploy Foundry agents privately in brazilsouth"` |
| **Expected behavior** | Skill flags potential availability concerns for the requested region and suggests alternatives or asks user to confirm. |
| **Validation** | Response mentions region availability or suggests fallback regions. |

#### P2-TC03: Complete specification — no extraneous questions

| Field | Value |
|-------|-------|
| **Prompt** | `"Deploy Foundry in my existing VNet 10.0.0.0/16 in eastus2, resource group rg-foundry, with AI Search, Cosmos DB, Storage, and private endpoints. System-assigned identity is fine."` |
| **Expected behavior** | Skill acknowledges the complete specification and proceeds to planning without unnecessary follow-up. |
| **Validation** | Skill does not ask for already-provided values. Moves to Phase 3. |

#### P2-TC04: Missing subnet details for BYO VNet

| Field | Value |
|-------|-------|
| **Prompt** | `"I have a VNet at 10.0.0.0/16 in eastus2. Deploy Foundry agents privately."` |
| **Expected behavior** | Skill asks about subnet configuration: agent subnet (/24), PE subnet (/24), whether subnets already exist or should be created. |
| **Validation** | Response asks for subnet CIDR details and delegation requirements. |

#### P2-TC05: Existing IaC — skill adapts its approach

| Field | Value |
|-------|-------|
| **Prompt** | `"I already have a Bicep template for my VNet and storage. Help me add Foundry to it."` |
| **Expected behavior** | Skill asks for the existing template or references, and plans to modify rather than generate net-new infrastructure. |
| **Validation** | Response acknowledges existing IaC and adjusts the approach to modification. |

#### P2-TC06: Agent workload exclusion — no Hosted Agents

| Field | Value |
|-------|-------|
| **Prompt** | `"Set up private Foundry — I need standard agents, not hosted agents"` |
| **Expected behavior** | Skill notes the standard agent preference and does not include hosted agent infrastructure. |
| **Validation** | Output plan does not include hosted agent components. |

#### P2-TC07: DNS and on-prem access implications

| Field | Value |
|-------|-------|
| **Prompt** | `"Deploy Foundry privately in our existing hub-spoke network. We connect to Azure via ExpressRoute."` |
| **Expected behavior** | Skill asks about DNS resolution (Azure Private DNS vs custom DNS forwarding), VPN/ExpressRoute connectivity, and how on-prem clients will resolve private endpoints. |
| **Validation** | Response mentions DNS zones and on-prem connectivity considerations. |

#### P2-TC08: APIM and MCP combined requirements

| Field | Value |
|-------|-------|
| **Prompt** | `"We need Foundry agents with APIM and MCP server support on the VNet"` |
| **Expected behavior** | Skill identifies a potential conflict (APIM template and Hybrid/MCP template are separate) and asks clarifying questions to resolve the architecture choice. |
| **Validation** | Skill does not silently ignore one of the two requirements. |

### Success Criterion

A complete, internally consistent requirement set suitable for Azure deployment. No critical dependencies missed.

---

## Phase 3: Planning & Architecture Generation

### Goal

Validate the skill's ability to reason holistically and produce a deployment-ready plan that maps to a known Foundry VNet template.

### Skill Responsibilities

- Generate a step-by-step deployment plan
- Identify: new resources, resource relationships/dependencies, required private endpoints, DNS zones, identity & RBAC considerations
- Optionally produce a high-level architecture description
- Map to known Foundry Bicep templates (templates 10, 15, 16, 17, 18, 19)

### Test Cases

#### P3-TC01: BYO VNet standard plan — correct template mapping

| Field | Value |
|-------|-------|
| **Scenario** | Requirements gathered: BYO VNet, eastus2, agents, Cosmos DB, Storage, AI Search, system-assigned identity, no APIM. |
| **Expected behavior** | Skill produces a plan referencing template 15 (`private-network-standard-agent-setup`). Plan lists: VNet + 2 subnets, AI Services account, Cosmos DB, Storage, AI Search, private endpoints for each, DNS zones, capability host, RBAC assignments. |
| **Validation** | Plan contains all expected resources. References the correct Bicep template URL. |

#### P3-TC02: Managed VNet plan — preview considerations

| Field | Value |
|-------|-------|
| **Scenario** | Requirements gathered: Managed VNet, westus2, agents. |
| **Expected behavior** | Plan references template 18 (`managed-virtual-network-preview`). Notes: preview feature flag `AI.ManagedVnetPreview`, async capability host provisioning (10–20 min), no upgrade path from custom VNet. |
| **Validation** | Plan includes preview warnings and feature flag registration step. |

#### P3-TC03: Hybrid plan — MCP subnet included

| Field | Value |
|-------|-------|
| **Scenario** | Requirements gathered: BYO VNet, agents, MCP servers on VNet, optional public Foundry access. |
| **Expected behavior** | Plan references template 19 (`hybrid-private-resources-agent-setup`). Includes three subnets: agent, PE, MCP. |
| **Validation** | Plan lists 3 subnets. MCP subnet is present. |

#### P3-TC04: Plan includes RBAC prerequisites

| Field | Value |
|-------|-------|
| **Scenario** | Any private network plan. |
| **Expected behavior** | Plan includes RBAC requirements: Owner or User Access Administrator on the resource group, role assignments for managed identity (Storage Blob Data Contributor, Cosmos DB Operator, AI Search roles). |
| **Validation** | RBAC section is present in the plan. |

#### P3-TC05: Plan includes DNS zones

| Field | Value |
|-------|-------|
| **Scenario** | BYO VNet standard plan (template 15). |
| **Expected behavior** | Plan lists required private DNS zones: `privatelink.services.ai.azure.com`, `privatelink.openai.azure.com`, `privatelink.cognitiveservices.azure.com`, and zones for Cosmos DB, Storage, and AI Search. |
| **Validation** | DNS zones for all private-endpoint-enabled resources are enumerated. |

#### P3-TC06: Plan identifies deployment order

| Field | Value |
|-------|-------|
| **Scenario** | Any VNet plan. |
| **Expected behavior** | Plan describes deployment order: (1) VNet + subnets, (2) dependent resources, (3) private endpoints + DNS, (4) Foundry account + capability host, (5) model deployment, (6) agent creation. |
| **Validation** | Steps are ordered and dependency relationships are clear. |

#### P3-TC07: Customer can review plan before code generation

| Field | Value |
|-------|-------|
| **Scenario** | After plan generation. |
| **Expected behavior** | Skill explicitly asks the user to review and confirm the plan before proceeding to code generation. |
| **Validation** | Response includes a confirmation prompt (e.g., "Does this plan look correct?" or equivalent). |

### Success Criterion

Customer can review the plan and explicitly confirm alignment before code generation begins.

---

## Phase 4: Infrastructure Code Generation & Modification

### Goal

Validate the skill's ability to translate the confirmed plan into deployable Bicep/Terraform code.

### Skill Responsibilities

- Generate new Bicep/Terraform code, or modify existing customer-provided code
- Ensure: idempotent deployment, correct parameterization, no hard-coded environment values
- Indicate manual steps (if unavoidable) and deployment order

### Test Cases

#### P4-TC01: New Bicep generation — references official template

| Field | Value |
|-------|-------|
| **Scenario** | BYO VNet standard (template 15), no existing IaC. |
| **Expected behavior** | Skill references the official Bicep template at `https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/15-private-network-standard-agent-setup`. Generates or guides user to download and parameterize it. |
| **Validation** | Output references the correct template URL. Deployment command uses `az deployment group create`. |

#### P4-TC02: Parameterization — no hard-coded values

| Field | Value |
|-------|-------|
| **Scenario** | Any generated Bicep output. |
| **Expected behavior** | Subscription ID, resource group name, region, VNet CIDR, subnet CIDRs, resource names are all parameterized (either as Bicep `param` or CLI `--parameters`). |
| **Validation** | No hard-coded GUIDs, resource names, or CIDRs in the template body. |

#### P4-TC03: Existing IaC modification

| Field | Value |
|-------|-------|
| **Scenario** | User provides an existing Bicep template for their VNet. |
| **Expected behavior** | Skill modifies the existing template to add Foundry resources, private endpoints, and DNS zones rather than generating a separate template. |
| **Validation** | Output is a diff or updated version of the user's template, not a wholly separate deployment. |

#### P4-TC04: Managed VNet — outbound rules via CLI

| Field | Value |
|-------|-------|
| **Scenario** | Managed VNet plan (template 18). |
| **Expected behavior** | Skill includes the post-deployment `az rest` commands for outbound rule configuration, since these cannot be defined in Bicep. |
| **Validation** | Output includes `az rest` commands for outbound rules. Documents manual steps clearly. |

#### P4-TC05: APIM integration — existing APIM resource ID parameter

| Field | Value |
|-------|-------|
| **Scenario** | Private Network + APIM plan (template 16). |
| **Expected behavior** | Skill instructs user to pass their existing APIM resource ID as a Bicep parameter (`apiManagementResourceId`). |
| **Validation** | Parameter for APIM resource ID is present in the deployment command. |

#### P4-TC06: Deployment command includes async polling

| Field | Value |
|-------|-------|
| **Scenario** | Any VNet template deployment. |
| **Expected behavior** | Skill warns about asynchronous capability host provisioning (10–20 min) and includes or recommends polling the deployment status. |
| **Validation** | Output mentions async provisioning and polling requirement. |

#### P4-TC07: Model deployment step included

| Field | Value |
|-------|-------|
| **Scenario** | Any VNet plan. |
| **Expected behavior** | After infrastructure deployment, skill includes the model deployment step (e.g., deploy `gpt-4o` or `gpt-5.3`), with fallback to `Standard` SKU if `GlobalStandard` quota is exhausted. |
| **Validation** | Post-deployment steps include model deployment with SKU fallback guidance. |

### Success Criterion

Infrastructure can be deployed successfully using the generated or updated code with minimal human adjustment.

---

## Phase 5: Validation & Post-Deployment Testing

### Goal

Validate that the skill can verify the deployed environment meets all stated requirements.

### Skill Responsibilities

- Verify: Foundry resource accessibility, private endpoint configuration, dependent service reachability, networking boundary behavior
- Cross-check deployment against original requirements
- Flag: missing components, misconfigurations, deviations from plan

### Test Cases

#### P5-TC01: Foundry resource accessibility check

| Field | Value |
|-------|-------|
| **Scenario** | Post-deployment of any private setup. |
| **Expected behavior** | Skill suggests commands to verify the Foundry resource is accessible (e.g., `az cognitiveservices account show`, connectivity test from within the VNet). |
| **Validation** | Output includes a verification command for Foundry resource status. |

#### P5-TC02: Private endpoint verification

| Field | Value |
|-------|-------|
| **Scenario** | Post-deployment of BYO VNet standard. |
| **Expected behavior** | Skill provides commands to list and verify private endpoints for all resources (AI Services, Cosmos DB, Storage, AI Search). |
| **Validation** | Output includes `az network private-endpoint list` or equivalent verification. |

#### P5-TC03: DNS resolution validation

| Field | Value |
|-------|-------|
| **Scenario** | Post-deployment. |
| **Expected behavior** | Skill provides steps to verify DNS resolution of private endpoints (e.g., `nslookup` from a VM on the VNet, checking private DNS zone records). |
| **Validation** | Output includes DNS resolution verification steps. |

#### P5-TC04: Dependent service reachability

| Field | Value |
|-------|-------|
| **Scenario** | Post-deployment of standard setup with Cosmos DB, Storage, AI Search. |
| **Expected behavior** | Skill suggests verifying connectivity to each dependent service from the Foundry account (e.g., capability host can reach Cosmos DB, Storage, and AI Search over private network). |
| **Validation** | Output includes reachability checks for each dependent service. |

#### P5-TC05: Cross-check against original requirements

| Field | Value |
|-------|-------|
| **Scenario** | Completed deployment. |
| **Expected behavior** | Skill produces a structured summary comparing what was requested (Phase 2 output) against what was deployed, noting any deviations. |
| **Validation** | Output includes a comparison table or checklist: requirement → deployed status (pass/fail/partial). |

#### P5-TC06: Agent creation validation

| Field | Value |
|-------|-------|
| **Scenario** | Post-deployment with agent workloads. |
| **Expected behavior** | Skill guides user to create a test agent and invoke it to confirm the end-to-end setup is functional (model deployed, capability host ready, agent reachable). |
| **Validation** | Output includes agent creation and invocation test steps. |

#### P5-TC07: Misconfiguration detection — public access on private resource

| Field | Value |
|-------|-------|
| **Scenario** | User intended fully private but a resource has `publicNetworkAccess: Enabled`. |
| **Expected behavior** | Skill detects the mismatch and flags it as a misconfiguration. |
| **Validation** | Output includes a check for `publicNetworkAccess` settings on all resources and raises a warning when it conflicts with the private requirement. |

### Success Criterion

Clear validation report (pass/fail/partial per requirement) confirming whether the deployed environment matches customer intent.

---

## Test Matrix — Template Coverage

Each networking template should be exercised by at least one end-to-end scenario across all five phases:

| Template | ID | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Phase 5 |
|----------|----|---------|---------|---------|---------|---------|
| Standard (public) | — | P1-TC04 | — | — | — | — |
| Private Basic (no agents) | 10 | P1-TC05 | — | — | — | — |
| Private Standard (BYO VNet) | 15 | P1-TC01/02 | P2-TC01/03/04 | P3-TC01/04/05 | P4-TC01/02 | P5-TC01–07 |
| Private Standard + APIM | 16 | P1-TC06 | P2-TC08 | — | P4-TC05 | — |
| Private Standard + UAI | 17 | P1-TC07 | — | — | — | — |
| Managed VNet | 18 | P1-TC03 | — | P3-TC02 | P4-TC04 | — |
| Hybrid (MCP subnet) | 19 | P1-TC08 | — | P3-TC03 | — | — |

> **Priority:** Template 15 (Private Standard BYO VNet) is the primary end-to-end scenario and should be the first fully automated test path.

---

## Integration Test Prompts (for `integration.test.ts`)

These prompts are designed for automated integration tests using the existing test harness:

### Skill Invocation Rate Tests

| Test ID | Prompt | Expected Skill |
|---------|--------|----------------|
| VNet-INV-01 | `"Deploy Azure AI Foundry in a private virtual network"` | `microsoft-foundry` |
| VNet-INV-02 | `"Set up Foundry with BYO VNet and private endpoints"` | `microsoft-foundry` |
| VNet-INV-03 | `"Configure managed virtual network for Foundry agents"` | `microsoft-foundry` |
| VNet-INV-04 | `"Deploy Foundry privately with Cosmos DB and AI Search behind private endpoints"` | `microsoft-foundry` |
| VNet-INV-05 | `"Help me set up network isolation for my Azure AI Foundry project"` | `microsoft-foundry` |

### Trigger Tests (positive — should trigger)

```text
"Deploy Azure AI Foundry in a private VNet"
"Set up Foundry with BYO VNet and private endpoints"
"Configure managed virtual network for Foundry agents"
"Help me set up network isolation for my Azure AI Foundry project"
"Deploy Foundry with private endpoints for Cosmos DB, Storage, and AI Search"
"I need a private Foundry setup with subnet delegation"
"Set up Foundry agents in an isolated network environment"
"Deploy Foundry privately with APIM integration"
"Configure Foundry with user-assigned identity in a private network"
"Deploy Foundry with MCP servers on a private VNet"
```

### Trigger Tests (negative — should NOT trigger)

```text
"Deploy a web app to Azure App Service"
"Help me configure an AWS VPC"
"Set up a Kubernetes cluster with network policies"
"How do I create an Azure Virtual Network without Foundry?"
```

---

## Final Output Expected from the Skill (for Test Pass)

For a complete end-to-end test pass, the skill must produce:

1. **Captured requirements** — Structured set of inputs including network model, region, resource group, services, security constraints, and existing infrastructure context
2. **Confirmed architecture plan** — Step-by-step deployment plan aligned with a known Foundry VNet template, reviewed and confirmed by the user
3. **Generated or modified IaC** — Parameterized Bicep/Terraform code (or reference to official template with customized parameters) that deploys without errors
4. **Deployment validation summary** — Structured pass/fail report for each requirement, covering resource accessibility, private endpoints, DNS, dependent services, and agent functionality
