# Microsoft Foundry Skill — Live BYO VNet Deployment Findings

> **Date:** April 15–16, 2026
> **Scope:** End-to-end live deployment of Template 15 (Private Network Standard Agent Setup) against a real Azure subscription — followed skill docs exactly, ran actual `az deployment group create` commands.
> **Skill version tested:** 1.0.10
> **Subscription:** Microsoft internal (restricted Azure Policy environment)
> **Location:** westus | **Resource Group:** rayankhoury
> **Deployments run:** gpt-4.1 ✅ | Mistral-Large-3 ✅ | Llama-3.3-70B-Instruct ✅

---

## Expected Workflow

The skill should guide a customer through these 5 phases in order, from first prompt to a working deployed agent:

| Phase | Description | Expected Behavior |
|-------|-------------|-------------------|
| **1. Information Gathering** | Understand user requirements | Ask routing questions, collect VNet/model/identity inputs in one exchange |
| **2. Plan Generation** | Present what resources will be created | Show template selection with reasoning, get confirmation, proceed |
| **3. Validation** | Check RBAC, policies, quotas, provider registrations | Catch all blockers BEFORE deployment. No preventable failures. |
| **4. Deploy & Track** | Deploy the template and track progress | Deploy, poll with smart intervals, surface errors clearly |
| **5. Test & Validate** | Post-deployment verification | Verify resources, deploy model, assign roles, validate agent access |

**Status after live testing:** Phases 4 and 5 required manual intervention at every run due to template bugs and missing skill guidance. No deployment succeeded on first attempt without fixes.

---

## Phase 1 & 2: Information Gathering and Plan Generation

### What should happen
Skill asks routing questions, identifies Template 15 (BYO VNet), collects model/VNet/identity params, and presents a deployment plan before running anything.

### What actually happened
Routing to Template 15 worked correctly once invoked. However, the skill did not surface the following inputs as required before deployment — they were discovered only at deploy-time failure:

- `disableLocalAuth` policy requirement
- `defaultOutboundAccess` policy requirement
- `modelVersion` format differences by vendor
- VNet name reuse risk on retry

### Issues

**P1-1. No pre-flight policy check before deployment** (Critical)
- Findings #6 and #7 below (policy blocks on `disableLocalAuth` and `defaultOutboundAccess`) are predictable in any enterprise subscription with Azure Policy enabled.
- A `az deployment group what-if` step, or explicit pre-flight questions ("Does your subscription enforce `disableLocalAuth`? Does it require subnets to disable default outbound access?"), would catch these before a failed deployment.
- **Status:** Not present in skill. Aligns with Proposal Phase 1 Fix #4.

**P1-2. No model catalog verification step** (Medium)
- The skill provides no guidance on how to find the correct `modelVersion` for non-OpenAI models before parameterizing.
- Mistral-Large-3 uses integer versions (`1`, `2`) while OpenAI uses date strings (`2025-04-14`). This is not documented anywhere in the skill.
- **Status:** Not present in skill. See Finding #12.

---

## Phase 3: Validation

### What should happen
Skill checks RBAC roles, quota, Azure Policy compliance, and provider registrations before touching any resource.

### What actually happened
No validation step was performed by the skill. All three blockers below (Findings #1, #6, #7) were discovered at deployment failure, not pre-deployment.

### Issues

**P3-1. RBAC prerequisite is wrong** (Critical)
- Skill states "Owner OR User Access Administrator" is sufficient. UAA alone fails with `AuthorizationFailed` on `Microsoft.Resources/deployments/validate/action`.
- Correct minimum: **Owner, or Contributor + UAA** (both roles when not Owner).
- **Source:** `references/standard-agent-setup.md`
- **Status:** Fixed locally in our fork. Must be upstreamed.

**P3-2. Azure AI Developer role not listed as post-deployment requirement** (High)
- The `Azure AI Developer` role (`64702f94-c441-49e6-a78b-ef80e0188fee`) on the AI Account is required to create or manage agents. The template does not assign it. The skill does not mention it.
- After a successful deployment, the deploying user cannot create agents without this role.
- **Status:** Added manually post-deployment each run. Not in template or skill prereqs.

---

## Phase 4: Deploy & Track

### What should happen
Skill runs the deployment, polls for status at smart intervals, surfaces errors with clear descriptions and proposed fixes when they occur.

### What actually happened
`az deployment group create` runs silently for 15–20 minutes. On first attempt, deployment fails at ARM validation due to policy blocks. On retry, customer has no VNet name reuse warning and may re-hit the orphaned subnet problem.

### Issues

**P4-1. Template defaults `disableLocalAuth: false` — blocked by enterprise policy** (Critical)
- `ai-account-identity.bicep` hardcodes `disableLocalAuth: false`. Enterprise policies (common in Microsoft and many large customers) require `disableLocalAuth: true`.
- Deployment fails at ARM validation immediately. A customer would have no way to know this was coming.
- **Source:** `modules-network-secured/ai-account-identity.bicep`
- **Fix:** Change default to `true`. Expose as an explicit template parameter.

**P4-2. Subnets missing `defaultOutboundAccess: false` — blocked by enterprise policy** (Critical)
- Both `agent-subnet` and `pe-subnet` are created without `defaultOutboundAccess: false`.
- Azure deprecated default outbound access in September 2025. Enterprise policies block subnets without this property.
- **Source:** `modules-network-secured/vnet.bicep`, `modules-network-secured/subnet.bicep`
- **Fix:** Add `defaultOutboundAccess: false` to both subnet property blocks.

**P4-3. Orphaned subnet service link on retry — permanent VNet lockout** (High)
- If a deployment reaches the capability host step before failing, Azure Container Apps delegates `agent-subnet` with a `legionservicelink` service association that **cannot be removed by the user**.
- Reusing the same VNet name on retry causes immediate failure. The subnet is permanently stuck.
- The skill does not warn about this.
- **Fix:** Explicitly document: "Always use a new `vnetName` on every retry. Never reuse a VNet name from a deployment that reached the capability host step."

**P4-4. No deployment progress monitoring guidance** (High)
- The skill notes the capability host step is async (10–20 min) but provides no polling guidance. `az deployment group create` runs silently — no output for 15+ minutes.
- Customers have no way to distinguish "still running" from "stalled" or "failed silently."
- **Aligns with:** Proposal Phase 1 Fix #6, Proposal Phase 2 Deploy & Track gap.
- **Proposed additions:**

  ```bash
  # Check overall deployment state
  az deployment group show \
    --resource-group <rg> \
    --name <deployment-name> \
    --query "{state:properties.provisioningState, error:properties.error}" \
    -o json

  # Check resource-level progress
  az deployment operation group list \
    --resource-group <rg> \
    --name <deployment-name> \
    --query "[].{resource:properties.targetResource.resourceType, state:properties.provisioningState}" \
    -o table

  # Or wait with timeout
  az deployment group wait \
    --resource-group <rg> \
    --name <deployment-name> \
    --created --timeout 1800
  ```

**P4-5. NRMS auto-NSG deployments appear as alarming failures in portal** (Medium — internal subscriptions only)
- In Microsoft-managed subscriptions, the Network Resource Management Service (NRMS) automatically tries to create NSGs on new VNets. NRMS is itself blocked by subscription policy, resulting in multiple `NRMS-VNet-NSG-*` failed deployments in the portal alongside the main deployment.
- These look alarming but are unrelated to the main deployment. Customer subscriptions do not have NRMS.
- **Fix:** Document in playbook for internal testing environments.

**P4-6. Non-OpenAI model versions use integer format, not date strings** (Medium)
- OpenAI models use calendar-date versioning (e.g., `gpt-4.1` → `2025-04-14`, `gpt-4o` → `2024-11-20`). This is the convention shown in all skill examples and docs.
- Mistral AI and Meta models use simple integer versioning (e.g., `Mistral-Large-3` → `1`, `2`; `Llama-3.3-70B-Instruct` → `9`).
- **What happened:** Following the skill's OpenAI-style convention, we set `modelVersion = '2405'` for Mistral-Large-3. Deployment failed immediately with:
  ```
  DeploymentModelNotSupported: The model 'Format:Mistral AI,Name:Mistral-Large-3,Version:2405' is not supported.
  ```
  The correct version was `1`, discovered by querying the model catalog. A second deployment (`mistral-deployment-v2`) with `modelVersion = '1'` succeeded.
- This is a silent trap: the skill's examples all use OpenAI date strings, leading users to infer the same format applies to all vendors. There is no warning that Mistral AI and Meta use a completely different convention.
- **Fix:** Skill should add a vendor version reference table and instruct users to always verify before parameterizing:
  ```bash
  az cognitiveservices model list --location <location> \
    --query "[?model.name=='<modelName>'].{version:model.version}" -o table
  ```
  Example reference table:

  | Vendor | Example Model | Version Format | Example |
  |--------|--------------|----------------|---------|
  | OpenAI | gpt-4.1 | Date string | `2025-04-14` |
  | OpenAI | gpt-4o | Date string | `2024-11-20` |
  | Mistral AI | Mistral-Large-3 | Integer | `1` |
  | Meta | Llama-3.3-70B-Instruct | Integer | `9` |

---

## Phase 5: Test & Validate

### What should happen
Skill verifies all resources deployed correctly, deploys a model, assigns the Azure AI Developer role, and validates the agent endpoint is reachable.

### What actually happened
No post-deployment validation guidance exists in the skill. Role assignment had to be done manually. Agent endpoint is not reachable from outside the VNet (by design, but not documented).

### Issues

**P5-1. Azure AI Developer role must be manually assigned post-deployment** (High)
- See P3-2. The template does not assign this role. The skill does not include it as a post-deployment step.
- **Fix:** Add explicit post-deployment step to assign `Azure AI Developer` role on the AI Account.

**P5-2. Agent endpoint is private — no skill guidance on how to reach it** (High)
- When `publicNetworkAccess` is `Disabled`, the Agents API (`https://{account}.services.ai.azure.com/api/projects/{project}/assistants`) is unreachable from outside the VNet — even with correct Entra ID credentials and correct roles.
- The correct token audience is `https://ai.azure.com` (not `https://cognitiveservices.azure.com/`).
- The skill does not explain this or provide options for accessing the private endpoint (VM via Azure Bastion, VPN Gateway, ExpressRoute, Cloud Shell with VNet integration).
- **Fix:** Add explicit callout in post-deployment section: "This deployment disables public network access. All agent management must happen from within the VNet or via a connected network."

---

## Bicep Compile Warnings (Non-Blocking)

These appear on every deployment but do not cause failures:

| Warning | Source | Impact |
|---------|--------|--------|
| BCP037: `capabilityHostKind` not in type library | `add-project-capability-host.bicep:26` | None — ARM accepts it |
| BCP318: Null-access on conditional module outputs | `network-agent-vnet.bicep:60-67` | Risk on BYO VNet path |
| BCP321: DNS zone VNet links typed as null | `private-endpoint-and-dns.bicep:365-403` | None — cosmetic |
| `no-hardcoded-env-urls`: `core.windows.net` hardcoded | `main.bicep:107,117` | Sovereign cloud blocker |

---

## Model Compatibility Summary

Template 15 successfully deployed with all three model families tested:

| Model | Format | Version | Deployment | Notes |
|-------|--------|---------|------------|-------|
| gpt-4.1 | OpenAI | 2025-04-14 | ✅ First attempt | Baseline — all fixes applied |
| Mistral-Large-3 | Mistral AI | 1 | ✅ Second attempt | First attempt failed: wrong version `2405` |
| Llama-3.3-70B-Instruct | Meta | 9 | ✅ First attempt | No issues after prior fixes in place |

---