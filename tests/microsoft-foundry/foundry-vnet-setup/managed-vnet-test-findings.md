# Microsoft Foundry Skill — Identified Issues

> **Date:** 2026-04-15  
> **Scope:** Skill review + 5 chat session test analysis (focus: Managed VNet setup)  
> **Skill version:** 1.0.10  
> **Test sessions:** `309fcc19`, `84547530`, `846e396a`, `c97cc191`, `9b7640ec`

---

## Expected Workflow

The skill should execute these 5 phases in order, smoothly, with minimal user intervention:

| Phase | Description | Expected Behavior |
|-------|-------------|-------------------|
| **1. Information Gathering** | Understand user requirements and needs | Ask the right questions, once, in a single batch. No irrelevant questions. |
| **2. Plan Generation** | Generate a plan showing what resources will be created | Present template selection with reasoning, get one confirmation, proceed. |
| **3. Validation** | Check RBAC, quotas, policies, feature flags, provider registrations | Catch all blockers BEFORE deployment. No preventable failures. |
| **4. Deploy & Track** | Deploy the template and poll to track progress | Fetch template, deploy, poll with smart intervals. No wasted turns. |
| **5. Test & Validate** | Post-deployment verification, model deployment, agent creation | Verify resources, deploy model, create agent, validate end-to-end. |

**No session has completed all 5 phases.** The best session (`9b7640ec`) reached Phase 4 but was still polling when the transcript ended. Phases 4 and 5 remain untested end-to-end.

---

## Phase 1: Information Gathering

### What should happen
Single batch of questions covering: agent workloads (yes/no), network isolation (public/private), VNet management (BYO/Managed), APIM, MCP servers, identity model, region, existing infrastructure. No irrelevant questions. One exchange.

### What actually happened

| Session | BYO vs Managed asked? | Irrelevant Qs? | Missing Qs? | Exchanges | Outcome |
|---------|----------------------|----------------|-------------|-----------|---------|
| `309fcc19` | ❌ Never asked | Region, RG, model only | All routing Qs | 1 | Wrong template (15 instead of 18) |
| `c97cc191` | ❌ Asked wrong Q ("new vs existing VNet") | "Hosted vs Prompt agent" | BYO/Managed, APIM, MCP, Identity | 1 | Wrong template (15), session abandoned |
| `84547530` | ✅ (via different skill) | None | None | 1 | Correct template (18) |
| `9b7640ec` | ✅ Correctly asked | None | None | 1 | Correct template (18) |

### Issues

**P1-1. Agent bypasses routing and pre-selects a template before asking questions** (Critical)
- **Sessions:** `309fcc19`, `c97cc191`
- In `309fcc19`, the agent read `private-network-standard-agent-setup.md` as its very first action (Turn 0) and locked onto template 15 before any user interaction.
- In `c97cc191`, the agent searched for `private-network-standard-agent-setup.md` in Turn 0, read it in Turn 1, then asked questions in Turn 2 — but the template was already chosen.
- **Root cause:** The agent sees template doc names in the sub-skills table or file system and reads one before consulting the routing logic. The `⛔ STOP` instruction in Step 0 was not effective at preventing this.
- **Status:** SKILL.md has been updated with stronger guard rails (⛔ banner at top of sub-skills section, guard rails in each template doc). Needs re-testing.

**P1-2. BYO VNet vs Managed VNet question missing or asked wrong** (Critical)
- **Sessions:** `309fcc19` (never asked), `c97cc191` (asked "create new VNet vs use existing" — wrong question, conflates infrastructure state with architecture decision)
- This is THE critical routing question — determines template 15 (BYO) vs 18 (Managed). Getting it wrong wastes the entire session.
- **Status:** SKILL.md Step 0 now has prescriptive Q3 with exact wording. Needs re-testing.

**P1-3. "Hosted vs Prompt agent" question asked — irrelevant for infra** (Medium)
- **Session:** `c97cc191`
- Agent type (hosted/prompt) has zero bearing on infrastructure template selection. All templates support both types.
- **Status:** SKILL.md Q1 now explicitly says "Do NOT ask about agent type (hosted vs prompt)." Needs re-testing.

**P1-4. Silent exploration before asking questions wastes turns** (Low)
- **Session:** `9b7640ec` — Agent spent 5 turns silently reading workspace files before asking the user anything. Adds latency but doesn't affect correctness.

---

## Phase 2: Plan Generation

### What should happen
After requirements are collected, present the selected template with reasoning, list what resources will be created, note trade-offs/limitations, get ONE confirmation, proceed.

### What actually happened

| Session | Correct template? | Plan presented? | Confirmations needed | Unnecessary back-and-forth |
|---------|-------------------|-----------------|---------------------|---------------------------|
| `309fcc19` | ❌ Template 15 (should be 18) | Yes (for wrong template) | 1 | N/A (wrong template) |
| `c97cc191` | ❌ Template 15 (should be 18) | Partial todo list | 0 (abandoned) | Session abandoned |
| `84547530` | ✅ Template 18 | Yes | 1 | None |
| `9b7640ec` | ✅ Template 18 | Yes | **2** (asked twice) | 1 extra confirmation |

### Issues

**P2-1. Double confirmation before proceeding** (Low)
- **Session:** `9b7640ec` — Agent presented the plan (Turn 7), user said "yes," then agent presented detailed steps (Turn 11) and asked "Shall we start?" again. One confirmation is sufficient.

**P2-2. No decision logic between `project/create` (azd) vs Bicep templates** (High)
- A user saying "set up Foundry" has no clear routing between azd (public-only, fast) and Bicep templates (full control, VNet support). The agent may send users to `project/create` when they need private networking.
- **Status:** SKILL.md Step 1 now routes between azd and Bicep based on Q2 answer. Needs re-testing.

**P2-3. `project/create` has no warning it creates public-only infrastructure** (High)
- **File:** `project/create/create-foundry-project.md`
- Users needing VNet isolation will waste time before discovering `project/create` can't help.

---

## Phase 3: Validation

### What should happen
Before deploying, validate ALL prerequisites: RBAC roles, quota availability, Azure policy compliance, feature flag registration, resource provider registrations, region availability. Catch every blocker before investing in a deployment.

### What actually happened

| Session | RBAC checked? | Quota checked? | Policies checked? | Feature flag checked? | Providers checked? | Deployment failure? |
|---------|--------------|----------------|-------------------|-----------------------|--------------------|--------------------|
| `309fcc19` | ✅ Thorough | ❌ | ❌ | ❌ | ❌ | N/A (never deployed) |
| `c97cc191` | ❌ | ❌ | ❌ | ❌ | ❌ | N/A (abandoned) |
| `84547530` | ❌ | ❌ | ❌ | ❌ | ❌ | N/A (never deployed) |
| `9b7640ec` | ❌ | ❌ | ❌ | ✅ | ❌ | **Yes — Azure Policy** |

### Issues

**P3-1. No Azure Policy validation — caused preventable deployment failure** (Critical)
- **Session:** `9b7640ec` — First deployment failed because the Bicep template had `disableLocalAuth: false` but the organization's Azure Policy required `disableLocalAuth: true` on Cognitive Services resources.
- A pre-deployment `az deployment group what-if` or `az policy assignment list --resource-group <rg>` would have caught this before wasting the deployment attempt.
- **Cost:** 1 failed deployment + 3 turns to diagnose and fix = ~2 minutes wasted.

**P3-2. No RBAC validation in 4 of 5 sessions** (Critical)
- Only `309fcc19` checked RBAC (and found the user was missing Contributor — would have caused deployment failure). All other sessions skipped it entirely.
- **Inconsistency:** The one session that did check RBAC hit a CLI error (`az role assignment create --resource-group` doesn't exist; needs `--scope`).

**P3-3. No quota validation in any session** (High)
- Zero sessions checked quota before deployment. If AI Services, Cosmos DB, or Search quotas were exhausted, the deployment would fail mid-way after already creating some resources — a partial deployment that's hard to clean up.

**P3-4. No resource provider registration checks** (Medium)
- `Microsoft.CognitiveServices`, `Microsoft.DocumentDB`, `Microsoft.Search`, `Microsoft.Network` must be registered. Never validated.

**P3-5. No `what-if` deployment** (High)
- `az deployment group what-if` would catch policy violations, quota issues, and parameter errors in a single dry run. Never used in any session.

**P3-6. 5 of 7 setup reference templates have no prerequisites section** (Critical)
- **Files:** `private-network-basic.md`, `private-network-standard-agent-setup.md`, `private-network-standard-agent-apim-setup.md`, `private-network-uai-agent-setup.md`, `hybrid-private-resources-agent-setup.md`
- Only `standard-agent-setup.md` and `managed-virtual-network-agent-setup.md` have prerequisite sections. The others jump straight to deployment.

---

## Phase 4: Deploy & Track

### What should happen
Fetch the correct Bicep template, deploy with appropriate parameters, poll with smart intervals (exponential backoff: 1min, 3min, 5min), report status changes, handle errors gracefully. Minimal wasted turns.

### What actually happened (Session `9b7640ec` — only session to reach this phase)

| Step | What happened | Turns | Issue? |
|------|---------------|-------|--------|
| Template fetch | Cloned from GitHub via git sparse-checkout | 6 | Template was already in workspace (`C:\work\engagements\vNet\skills\18-managed-virtual-network-preview\`) — clone was unnecessary |
| Parameter update | Changed `eastus2` → `swedencentral` | 1 | OK |
| Deploy attempt 1 | `az deployment group create` | 1 | **FAILED** — Azure Policy `disableLocalAuth` |
| Fix | Changed `disableLocalAuth: false` → `true` | 2 | Quick fix, but preventable with Phase 3 |
| Deploy attempt 2 | `az deployment group create` | 1 | Started successfully |
| Polling | 20 `az deployment group list` calls | 20 | **15 of 20 polls showed no change (75% wasted)** |
| Completion | Capability host still running at transcript end | — | Session ended before deployment completed |

### Issues

**P4-1. Excessive polling — 75% of polls wasted** (Critical)
- **Session:** `9b7640ec` — 20 polling calls over 31 minutes, only 5 showed status changes. The agent polled every 15-30 seconds during the first 3 minutes when the deployment takes 10-20 minutes.
- **Cost:** 15 wasted turns, significant token consumption, cluttered conversation.
- **Expected:** Exponential backoff — poll at 1min, 3min, 5min, then every 5min. Or use `az deployment group wait` with a timeout.

**P4-2. Template already in workspace but re-cloned from GitHub** (Medium)
- **Session:** `9b7640ec` — The `18-managed-virtual-network-preview/` folder already existed at `C:\work\engagements\vNet\skills\18-managed-virtual-network-preview\`. The agent cloned it from GitHub anyway, taking 6 turns.
- The skill should check for local templates before fetching from GitHub.

**P4-3. GitHub repo URL inconsistency** (High)
- 14 files use `microsoft-foundry/foundry-samples` (wrong), while 3 files use `azure-ai-foundry/foundry-samples` (correct). Wrong URLs cause fetch failures and retries.
- **Session `84547530`:** First GitHub fetch failed due to wrong org name.
- **Affected files:** All setup reference docs, `deploy.md` (2×), `create.md` (7×)

**P4-4. Fictitious model name `gpt-5.3` in post-deployment guidance** (Medium)
- **Files:** `managed-virtual-network-agent-setup.md` (line 57), `hybrid-private-resources-agent-setup.md` (line 72)
- Post-deployment says "Deploy a model (e.g., `gpt-5.3`)" — this model doesn't exist.

**P4-5. No Bicep parameter documentation in setup templates** (Medium)
- Users can't discover what parameters the Bicep template accepts (e.g., existing VNet/subnet IDs). The agent must read the `.bicepparam` file to figure it out.

---

## Phase 5: Test & Validate

### What should happen
After deployment completes: verify all resources are provisioned correctly, deploy a model, create an agent, invoke the agent to test, validate the end-to-end flow.

### What actually happened

**No session has reached this phase.** All sessions either abandoned, stalled in Phase 1-2 (wrong template), or stalled in Phase 4 (still polling).

### Issues

**P5-1. Post-deployment in setup templates doesn't chain to correct sub-skills** (High)
- All 7 setup reference templates say "Deploy a model" and "Create the agent" but don't link to `models/deploy-model` or `foundry-agent/deploy` sub-skills.

**P5-2. No model deployment validation before agent creation** (High)
- **File:** `foundry-agent/deploy/deploy.md`
- The deploy skill creates an agent referencing a model (e.g., `gpt-4o`) without verifying the deployment exists.

**P5-3. No ACR pre-validation before image build** (High)
- **File:** `foundry-agent/deploy/deploy.md`
- No check that ACR exists and user has `AcrPush` role before `az acr build`.

**P5-4. vNext agent readiness check contradiction** (Medium)
- `invoke.md` says "Ready immediately after deployment (no container status check needed)" but `troubleshoot.md` says to check version status.

---

## Cross-Cutting Issues

**X1. No session completed end-to-end** (Critical)
- 5 sessions, 0 completions. The skill cannot demonstrate a working end-to-end flow.
- Best session (`9b7640ec`): reached Phase 4 polling but didn't finish deployment, never reached Phase 5.

**X2. Inconsistent behavior across sessions** (High)
- RBAC checked in 1 session, skipped in 4. Feature flag checked in 1 session, skipped in 4. The skill doesn't enforce consistent validation.

**X3. User needed to abandon and restart** (High)
- Session `c97cc191` was abandoned because the wrong template was selected (missing BYO vs Managed question). User had to start session `9b7640ec` from scratch.

---

## Session Timeline Summary

| Session | Phase reached | User exchanges | Wasted turns | Root cause of stall |
|---------|--------------|----------------|--------------|---------------------|
| `309fcc19` | Phase 2 | 2 | ~3 (RBAC retries) | Wrong template, then template fetch stalled |
| `84547530` | Phase 2 | 3 | ~2 (GitHub URL retry) | Correct template found but session ended |
| `846e396a` | Phase 1 | 1 | 0 | Advisory only — no execution attempted |
| `c97cc191` | Phase 1 | 1 | 1 (wrong Qs) | **Abandoned** — wrong template due to missing BYO/Managed Q |
| `9b7640ec` | Phase 4 | 5 | **~18** (15 polls + 1 dup Q + 2 confirms) | Policy failure + excessive polling |

---
