# Proposal: Microsoft Foundry Skill — VNet Setup Improvements

> **Date:** April 15, 2026  
> **Skill version tested:** 1.0.10  

---

## Context

We tested the VNet setup skill from multiple angles to get a comprehensive picture:

- **BYO VNet — end-to-end flow:** Tested the full journey for setting up Foundry with bringing own VNet through deployment. Focused on whether the skill correctly routes to BYO templates and handles the complete lifecycle.
- **Managed VNet — end-to-end flow:** Ran 5 chat sessions specifically targeting the Managed VNet path (template 18). Tested routing, plan generation, validation, and deployment tracking.
- **Breadth — static analysis:** Analyzed all skill documents against a structured test plan covering requirement gathering, architecture planning (resource enumeration, deployment ordering, RBAC), validation checklists, and post-deployment verification.

---

## Phase 1: Quick Wins

Targeted fixes that improve reliability without restructuring the skill. These address issues we hit directly in testing.

| # | Fix | Why | Severity |
|---|-----|-----|----------|
| 1 | **Fix GitHub repo URL** — 14 files reference `microsoft-foundry/foundry-samples` (wrong org), should be `azure-ai-foundry/foundry-samples` | Caused fetch failure in session `84547530`. Agent retried with wrong URL before finding the correct one. | High |
| 2 | **Remove fictitious model name `gpt-5.3`** in `managed-virtual-network-agent-setup.md` (line 57) and `hybrid-private-resources-agent-setup.md` (line 72) | Model doesn't exist. Replace with a real model like `gpt-4o`. | Medium |
| 3 | **Add prerequisite sections to 5 templates** — `private-network-basic.md`, `private-network-standard-agent-setup.md`, `private-network-standard-agent-apim-setup.md`, `private-network-uai-agent-setup.md`, `hybrid-private-resources-agent-setup.md` | These templates jump straight to deployment with no prereqs. Only `standard-agent-setup.md` and `managed-virtual-network-agent-setup.md` have them. RBAC, feature flags, and provider registrations should be listed. | Critical |
| 4 | **Add `az deployment group what-if` guidance** to all templates | Session `9b7640ec` hit an Azure Policy failure (`disableLocalAuth`) that a dry run would have caught. Zero sessions used `what-if`. | High |
| 5 | **Link post-deployment steps to sub-skills** — "Deploy a model" → `models/deploy-model`, "Create the agent" → `foundry-agent/deploy` | All 7 setup templates mention these steps but don't link to the sub-skills. Users hit a dead end. | High |
| 6 | **Add polling backoff guidance** — recommend exponential backoff (1min, 3min, 5min) or `az deployment group wait` | Session `9b7640ec`: 20 polls in 31 minutes, 75% showed no change. 15 wasted turns. | Critical |
| 7 | **Add `project/create` public-only warning** | Users needing VNet isolation may start with `project/create` and waste time before discovering it can't help. | High |

---

## Phase 2: Structured Flow

The skill can *select* the right template but has no structured workflow between "pick a template" and "deploy it." This phase introduces the flow we discussed across all 5 phases.

### What's Missing Today

| Phase | Current State | Gap |
|-------|--------------|-----|
| **1. Information Gathering** | Routing questions exist (improved in latest commit) | No structured collection of VNet-specific inputs: DNS, on-prem/VPN, existing IaC, subnet details, agent workload types. Agent must infer from 8+ template docs. |
| **2. Plan Generation** | Template selection works (when questions are asked correctly) | Templates 15–19 don't enumerate what resources they create. No RBAC prereqs, no deployment ordering, no "review this plan" checkpoint. |
| **3. Validation** | Essentially absent | RBAC checked in 1/5 sessions, quota in 0/5, Azure Policy in 0/5, provider registrations in 0/5. No enforcement mechanism. |
| **4. Deploy & Track** | Works but is wasteful | No local template detection (re-cloned from GitHub when already in workspace), no polling strategy, no parameter documentation. |
| **5. Test & Validate** | Completely absent | No PE verification, no DNS resolution checks, no requirements cross-check, no end-to-end agent test workflow. |

## Findings

Detailed test plan, static analysis, prompt testing results, and managed VNet session analysis:  
[`tests/microsoft-foundry/foundry-vnet-setup/`](https://github.com/smartmuki/GitHub-Copilot-for-Azure/tree/manni/test_branch/tests/microsoft-foundry/foundry-vnet-setup)

| Document | Contents |
|----------|----------|
| `TEST_PLAN.md` | 5-phase test plan with test cases |
| `FINDINGS_STATIC_ANALYSIS.md` | Static analysis of skill docs against test plan (Phases 1–5) |
| `FINDINGS_VIA_PROMPT_TESTING.md` | Findings from live prompt testing sessions |
| `managed-vnet-test-findings.md` | Detailed session-by-session analysis (5 sessions, Managed VNet focus) |
| `updated_skills.md` | Current state of SKILL.md with latest guardrails |
