---
name: microsoft-foundry
description: "Deploy, evaluate, and manage Foundry agents end-to-end: Docker build, ACR push, hosted/prompt agent create, container start, batch eval, prompt optimization, prompt optimizer workflows, agent.yaml, dataset curation from traces. USE FOR: deploy agent to Foundry, hosted agent, create agent, invoke agent, evaluate agent, run batch eval, optimize prompt, improve prompt, prompt optimization, prompt optimizer, improve agent instructions, optimize agent instructions, optimize system prompt, deploy model, Foundry project, RBAC, role assignment, permissions, quota, capacity, region, troubleshoot agent, deployment failure, create dataset from traces, dataset versioning, eval trending, create AI Services, Cognitive Services, create Foundry resource, provision resource, knowledge index, agent monitoring, customize deployment, onboard, availability. DO NOT USE FOR: Azure Functions, App Service, general Azure deploy (use azure-deploy), general Azure prep (use azure-prepare)."
license: MIT
metadata:
  author: Microsoft
  version: "1.0.10"
---

# Microsoft Foundry Skill

This skill helps developers work with Microsoft Foundry resources, covering model discovery and deployment, complete dev lifecycle of AI agent, evaluation workflows, and troubleshooting.

## Sub-Skills

> **MANDATORY: Before executing ANY workflow, you MUST read the corresponding sub-skill document.** Do not call MCP tools for a workflow without reading its skill document. This applies even if you already know the MCP tool parameters — the skill document contains required workflow steps, pre-checks, and validation logic that must be followed. This rule applies on every new user message that triggers a different workflow, even if the skill is already loaded.

> **⛔ INFRASTRUCTURE REQUESTS — READ THIS FIRST:** If the user asks to **set up, provision, deploy, or create Foundry infrastructure** (including any mention of VNet, private network, isolated network, network security, or private endpoints), you **MUST** go to the **[Infrastructure Setup](#agent-infrastructure-setup)** section and complete **Step 0 (requirements gathering)** BEFORE reading ANY setup reference or template document. Do **NOT** read, open, search for, or reference any file under `references/` (e.g., `private-network-standard-agent-setup.md`, `standard-agent-setup.md`, `managed-virtual-network-agent-setup.md`, etc.) until Step 0 is complete and the routing logic in Step 1–2 has selected a specific template. This is **non-negotiable** — reading a template doc early biases your recommendation and causes wrong template selection.

This skill includes specialized sub-skills for specific workflows. **Use these instead of the main skill when they match your task:**

| Sub-Skill | When to Use | Reference |
|-----------|-------------|-----------|
| **deploy** | Containerize, build, push to ACR, create/update/start/stop/clone agent deployments | [deploy](foundry-agent/deploy/deploy.md) |
| **invoke** | Send messages to an agent, single or multi-turn conversations | [invoke](foundry-agent/invoke/invoke.md) |
| **observe** | Evaluate agent quality, run batch evals, analyze failures, optimize prompts, improve agent instructions, compare versions, and set up CI/CD monitoring | [observe](foundry-agent/observe/observe.md) |
| **trace** | Query traces, analyze latency/failures, correlate eval results to specific responses via App Insights `customEvents` | [trace](foundry-agent/trace/trace.md) |
| **troubleshoot** | View container logs, query telemetry, diagnose failures | [troubleshoot](foundry-agent/troubleshoot/troubleshoot.md) |
| **create** | Create new hosted agent applications. Supports Microsoft Agent Framework, LangGraph, or custom frameworks in Python or C#. Downloads starter samples from foundry-samples repo. | [create](foundry-agent/create/create.md) |
| **eval-datasets** | Harvest production traces into evaluation datasets, manage dataset versions and splits, track evaluation metrics over time, detect regressions, and maintain full lineage from trace to deployment. Use for: create dataset from traces, dataset versioning, evaluation trending, regression detection, dataset comparison, eval lineage. | [eval-datasets](foundry-agent/eval-datasets/eval-datasets.md) |
| **project/create** | Creating a new Azure AI Foundry project for hosting agents and models. Use when onboarding to Foundry or setting up new infrastructure. | [project/create/create-foundry-project.md](project/create/create-foundry-project.md) |
| **resource/create** | Creating Azure AI Services multi-service resource (Foundry resource) using Azure CLI. Use when manually provisioning AI Services resources with granular control. | [resource/create/create-foundry-resource.md](resource/create/create-foundry-resource.md) |
| **models/deploy-model** | Unified model deployment with intelligent routing. Handles quick preset deployments, fully customized deployments (version/SKU/capacity/RAI), and capacity discovery across regions. Routes to sub-skills: `preset` (quick deploy), `customize` (full control), `capacity` (find availability). | [models/deploy-model/SKILL.md](models/deploy-model/SKILL.md) |
| **quota** | Managing quotas and capacity for Microsoft Foundry resources. Use when checking quota usage, troubleshooting deployment failures due to insufficient quota, requesting quota increases, or planning capacity. | [quota/quota.md](quota/quota.md) |
| **rbac** | Managing RBAC permissions, role assignments, managed identities, and service principals for Microsoft Foundry resources. Use for access control, auditing permissions, and CI/CD setup. | [rbac/rbac.md](rbac/rbac.md) |
| **Infrastructure Setup** | Setting up Foundry infrastructure: creating projects, provisioning resources, deploying Bicep templates (public or private networking). **Requires mandatory requirements gathering — see section below.** | [Infrastructure Setup](#agent-infrastructure-setup) (this file) |

> 💡 **Tip:** For a complete onboarding flow: Infrastructure Setup (Step 0-3) → `models/deploy-model` → agent workflows (`create` → `deploy` → `invoke`).

> 💡 **Model Deployment:** Use `models/deploy-model` for all deployment scenarios — it intelligently routes between quick preset deployment, customized deployment with full control, and capacity discovery across regions.

> 💡 **Prompt Optimization:** For requests like "optimize my prompt" or "improve my agent instructions," load [observe](foundry-agent/observe/observe.md) and use the `prompt_optimize` MCP tool through that eval-driven workflow.

## Agent Development Lifecycle

Match user intent to the correct workflow. Read each sub-skill in order before executing.

| User Intent | Workflow (read in order) |
|-------------|------------------------|
| Set up Foundry infrastructure from scratch | Infrastructure Setup (Step 0-3 above) → [models/deploy-model](models/deploy-model/SKILL.md) → [create](foundry-agent/create/create.md) → [deploy](foundry-agent/deploy/deploy.md) → [invoke](foundry-agent/invoke/invoke.md) |
| Set up Foundry with VNet isolation | Infrastructure Setup (Step 0-3 above — **must** gather networking requirements first) |
| Create a new agent from scratch | [create](foundry-agent/create/create.md) → [deploy](foundry-agent/deploy/deploy.md) → [invoke](foundry-agent/invoke/invoke.md) |
| Deploy an agent (code already exists) | deploy → invoke |
| Update/redeploy an agent after code changes | deploy → invoke |
| Invoke/test/chat with an agent | invoke |
| Optimize / improve agent prompt or instructions | observe (Step 4: Optimize) |
| Evaluate and optimize agent (full loop) | observe |
| Troubleshoot an agent issue | invoke → troubleshoot |
| Fix a broken agent (troubleshoot + redeploy) | invoke → troubleshoot → apply fixes → deploy → invoke |
| Start/stop agent container | deploy |

## Agent: .foundry Workspace Standard

Every agent source folder should keep Foundry-specific state under `.foundry/`:

```text
<agent-root>/
  .foundry/
    agent-metadata.yaml
    datasets/
    evaluators/
    results/
```

- `agent-metadata.yaml` is the required source of truth for environment-specific project settings, agent names, registry details, and evaluation test cases.
- `datasets/` and `evaluators/` are local cache folders. Reuse them when they are current, and ask before refreshing or overwriting them.
- See [Agent Metadata Contract](references/agent-metadata-contract.md) for the canonical schema and workflow rules.

## Agent: Infrastructure Setup

When a user asks to **set up, provision, or deploy Foundry infrastructure** (not an agent application — that's the `deploy` sub-skill), follow the steps below **in order**. Do NOT skip to a specific template or reference doc before completing requirements gathering.

### Step 0: Gather Requirements (MANDATORY — do NOT skip)

> **⛔ STOP: You MUST complete this step before reading ANY setup reference document.** Do not open, read, search for, or reference any file under `references/` (e.g., `standard-agent-setup.md`, `private-network-standard-agent-setup.md`, `managed-virtual-network-agent-setup.md`, etc.) until all requirements below are collected and the routing logic has selected a template. Reading a template doc before routing biases your recommendation and leads to wrong template selection.

Use `askQuestions` to collect the following from the user in a **single call**. Ask **exactly these questions with exactly these option labels** — do not rephrase, merge, or skip any:

**Question 1 — Agent workloads (header: "Agent Workloads")**
> "Do you need to run AI agent workloads, or do you only need a private Foundry resource for hosting models and projects?"
> Options: `Yes - I need agent workloads` (recommended) | `No - just models and projects`
> *Why: Determines whether capability host and BYO resources (Cosmos DB, Storage, AI Search) are needed. Do NOT ask about agent type (hosted vs prompt) — that is irrelevant for infrastructure template selection.*

**Question 2 — Network isolation (header: "Network Isolation")**
> "Do you need network isolation (VNet, private endpoints), or is public internet access acceptable?"
> Options: `Private networking (VNet isolation)` | `Public access is fine` (recommended)

**Question 3 — VNet management (header: "VNet Management")**
> "Who should manage the virtual network? **BYO VNet** means you provide and manage your own VNet. **Managed VNet** means Azure creates and manages the VNet for you (preview feature, requires feature flag registration)."
> Options: `BYO VNet - I manage the VNet myself` (recommended) | `Managed VNet - Azure manages it for me (Preview)`
> *Why: This is the critical routing question — BYO VNet routes to templates 15/16/17/19, Managed VNet routes to template 18. These are completely different architectures. The question "existing VNet vs create new VNet" is a DIFFERENT question (about infrastructure state, not architecture).*

**Question 4 — APIM integration (header: "APIM Integration")**
> "Do you need Azure API Management (APIM) integration for your agents?"
> Options: `No` (recommended) | `Yes - I need APIM`

**Question 5 — MCP servers (header: "MCP Servers")**
> "Do you need to deploy MCP servers on the VNet, or need optional public Foundry portal access alongside private backend resources?"
> Options: `No` (recommended) | `Yes - I need MCP servers on VNet or hybrid access`

**Question 6 — Identity model (header: "Identity Model")**
> "What identity model? System-assigned managed identity is the default. User-assigned is for enterprises that require pre-provisioned identities."
> Options: `System-assigned (default)` (recommended) | `User-assigned managed identity`

**Question 7 — Region (header: "Azure Region")**
> "Which Azure region should resources be deployed in?"
> *(free text, no options)*

**Question 8 — Existing infrastructure (header: "Existing Infrastructure")**
> "Do you have an existing resource group, VNet, or subnets to use, or should everything be created from scratch?"
> Options: `Create everything from scratch` (recommended) | `I have existing infrastructure`

> 💡 **Shortcut:** If the user's message already answers some questions (e.g., "I need VNet isolation with BYO VNet in Sweden Central"), extract those answers and only ask the **remaining unanswered** questions. But you must still ask questions 1-6 if not answered — do not assume defaults for routing-critical decisions.

### Step 1: Route to the correct path

**Use answers from Step 0 to decide the path. Do NOT read any template doc yet.**

| Condition (from Step 0 answers) | Path |
|-----------|------|
| User chose **"Public access is fine"** (Q2) AND no special requirements | → [project/create](project/create/create-foundry-project.md) (uses `azd`, fastest path for public setup) |
| User chose **"Public access is fine"** (Q2) BUT needs custom Bicep parameters (specific SKUs, BYO resources) | → [Standard Agent Setup](references/standard-agent-setup.md) (Bicep) |
| User chose **"Private networking"** (Q2) | → Continue to Step 2 (Bicep templates only) |
| User chose **"No - just models and projects"** (Q1) AND needs private networking | → [Private Network Basic](references/private-network-basic.md) |
| User only needs a **bare AI Services resource** (no project, no agents, no infra) | → [resource/create](resource/create/create-foundry-resource.md) |

> ⚠️ **`project/create` only supports public infrastructure.** If the user chose private networking in Q2, do NOT use `project/create` — continue to Step 2.

### Step 2: Select the correct Bicep template (private networking only)

Route using **the answers already collected in Step 0**. Do not re-ask any question. Walk through in order — stop at the first match:

| # | Routing Question (from Step 0) | If Yes | If No |
|---|-------------------------------|--------|-------|
| 1 | Did user choose **"Managed VNet"** in Q3? | → **[Managed Virtual Network Agent Setup](references/managed-virtual-network-agent-setup.md)** (template 18) | → continue |
| 2 | Did user choose **"Yes - MCP servers on VNet or hybrid"** in Q5? | → **[Hybrid Private Resources Agent Setup](references/hybrid-private-resources-agent-setup.md)** (template 19) | → continue |
| 3 | Did user choose **"Yes - I need APIM"** in Q4? | → **[Private Network APIM Agent Setup](references/private-network-standard-agent-apim-setup.md)** (template 16, Preview) | → continue |
| 4 | Did user choose **"User-assigned managed identity"** in Q6? | → **[Private Network UAI Agent Setup](references/private-network-uai-agent-setup.md)** (template 17) | → **[Private Network Standard Agent Setup](references/private-network-standard-agent-setup.md)** (template 15) |

> 💡 **Multiple features:** If the user needs a combination not covered by a single template (e.g., BYO VNet + APIM + UAI), note that templates are not composable — recommend the template that covers the most critical requirement and explain what's not included.

### Step 3: Present the plan and confirm

Before reading the selected template's reference doc, **present your recommendation to the user**:
1. State which template was selected and why
2. Summarize what it will create
3. List any limitations or trade-offs
4. Get explicit user confirmation before proceeding

**Only after user confirms** → read the selected setup reference document and begin execution.

### Setup Reference Templates

| Template | Networking | Identity | APIM | MCP Subnet | Status | Reference |
|----------|-----------|----------|------|------------|--------|-----------|
| Standard (public) | Public | System-assigned | ❌ | ❌ | GA | [Standard Agent Setup](references/standard-agent-setup.md) |
| Private Basic (no agents) | Private (VNet + PE) | System-assigned | ❌ | ❌ | GA | [Private Network Basic](references/private-network-basic.md) |
| Private Standard | Private (BYO VNet) | System-assigned | ❌ | ❌ | GA | [Private Network Standard Agent Setup](references/private-network-standard-agent-setup.md) |
| Private Standard + APIM | Private (BYO VNet) | System-assigned | ✅ | ❌ | **Preview** | [Private Network APIM Agent Setup](references/private-network-standard-agent-apim-setup.md) |
| Private Standard + UAI | Private (BYO VNet) | **User-assigned** | ❌ | ❌ | GA | [Private Network UAI Agent Setup](references/private-network-uai-agent-setup.md) |
| Managed VNet | **Azure-managed VNet** | System-assigned | ❌ | ❌ | **Preview** | [Managed Virtual Network Agent Setup](references/managed-virtual-network-agent-setup.md) |
| Hybrid Private | Private backend, optional public Foundry | System-assigned | ❌ | ✅ | GA | [Hybrid Private Resources Agent Setup](references/hybrid-private-resources-agent-setup.md) |

## Agent: Project Context Resolution

Agent skills should run this step **only when they need configuration values they don't already have**. If a value (for example, agent root, environment, project endpoint, or agent name) is already known from the user's message or a previous skill in the same session, skip resolution for that value.

### Step 1: Discover Agent Roots

Search the workspace for `.foundry/agent-metadata.yaml`.

- **One match** → use that agent root.
- **Multiple matches** → require the user to choose the target agent folder.
- **No matches** → for create/deploy workflows, seed a new `.foundry/` folder during setup; for all other workflows, stop and ask the user which agent source folder to initialize.

### Step 2: Resolve Environment

Read `.foundry/agent-metadata.yaml` and resolve the environment in this order:
1. Environment explicitly named by the user
2. Environment already selected earlier in the session
3. `defaultEnvironment` from metadata

If the metadata contains multiple environments and none of the rules above selects one, prompt the user to choose. Keep the selected agent root and environment visible in every workflow summary.

### Step 3: Resolve Common Configuration

Use the selected environment in `agent-metadata.yaml` as the primary source:

| Metadata Field | Resolves To | Used By |
|----------------|-------------|---------|
| `environments.<env>.projectEndpoint` | Project endpoint | deploy, invoke, observe, trace, troubleshoot |
| `environments.<env>.agentName` | Agent name | invoke, observe, trace, troubleshoot |
| `environments.<env>.azureContainerRegistry` | ACR registry name / image URL prefix | deploy |
| `environments.<env>.testCases[]` | Dataset + evaluator + threshold bundles | observe, eval-datasets |

### Step 4: Bootstrap Missing Metadata (Create/Deploy Only)

If create/deploy is initializing a new `.foundry` workspace and metadata fields are still missing, check if `azure.yaml` exists in the project root. If found, run `azd env get-values` and use it to seed `agent-metadata.yaml` before continuing.

| azd Variable | Seeds |
|-------------|-------|
| `AZURE_AI_PROJECT_ENDPOINT` or `AZURE_AIPROJECT_ENDPOINT` | `environments.<env>.projectEndpoint` |
| `AZURE_CONTAINER_REGISTRY_NAME` or `AZURE_CONTAINER_REGISTRY_ENDPOINT` | `environments.<env>.azureContainerRegistry` |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription for trace/troubleshoot lookups |

### Step 5: Collect Missing Values

Use the `ask_user` or `askQuestions` tool **only for values not resolved** from the user's message, session context, metadata, or azd bootstrap. Common values skills may need:
- **Agent root** — Target folder containing `.foundry/agent-metadata.yaml`
- **Environment** — `dev`, `prod`, or another environment key from metadata
- **Project endpoint** — AI Foundry project endpoint URL
- **Agent name** — Name of the target agent

> 💡 **Tip:** If the user already provides the agent path, environment, project endpoint, or agent name, extract it directly — do not ask again.

## Agent: Agent Types

All agent skills support two agent types:

| Type | Kind | Description |
|------|------|-------------|
| **Prompt** | `"prompt"` | LLM-based agents backed by a model deployment |
| **Hosted** | `"hosted"` | Container-based agents running custom code |

Use `agent_get` MCP tool to determine an agent's type when needed.

## Tool Usage Conventions

- Use the `ask_user` or `askQuestions` tool whenever collecting information from the user
- Use the `task` or `runSubagent` tool to delegate long-running or independent sub-tasks (e.g., env var scanning, status polling, Dockerfile generation)
- Prefer Azure MCP tools over direct CLI commands when available
- Reference official Microsoft documentation URLs instead of embedding CLI command syntax

## Additional Resources

- [Foundry Hosted Agents](https://learn.microsoft.com/azure/ai-foundry/agents/concepts/hosted-agents?view=foundry)
- [Foundry Agent Runtime Components](https://learn.microsoft.com/azure/ai-foundry/agents/concepts/runtime-components?view=foundry)
- [Foundry Samples](https://github.com/azure-ai-foundry/foundry-samples)

## SDK Quick Reference

- [Python](references/sdk/foundry-sdk-py.md)
