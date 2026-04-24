# Private Network Standard Agent Setup

> **MANDATORY:** Read [Standard Agent Setup with Network Isolation docs](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-private-link?tabs=azure-portal&pivots=fdp-project) before proceeding. It covers RBAC requirements, resource provider registration, and role assignments.

>  **Reachability:** With `publicNetworkAccess` disabled, agent endpoints are **not reachable from outside the VNet**. See [post-deployment-validation](../resource/private-network/references/post-deployment-validation.md) for access setup (Bastion, VPN, or ExpressRoute).

## Overview

Extends [standard agent setup](standard-agent-setup.md) with full VNet isolation using private endpoints and subnet delegation. All resources communicate over private network only.

## Networking Constraints

Two subnets required:

| Subnet | CIDR | Purpose | Delegation |
|--------|------|---------|------------|
| Agent Subnet | /24 (e.g., 192.168.0.0/24) | Agent workloads | `Microsoft.App/environments` (exclusive) |
| Private Endpoint Subnet | /24 (e.g., 192.168.1.0/24) | Private endpoints | None |

- All Foundry resources **must be in the same region as the VNet**.
- Agent subnet must be exclusive to one Foundry account.
- VNet address space must not overlap with existing networks or reserved ranges.

> ⚠️ **Warning:** If providing an existing VNet, ensure both subnets exist before deployment. Otherwise the template creates a new VNet with default address spaces.

## Retry Rule — Never Reuse VNet Name

If a prior deployment reached the capability host step and failed, Azure Container Apps leaves a `legionservicelink` service association on the agent subnet that **cannot be removed by the user**. Reusing the same VNet name on retry will fail immediately.

> **Always use a new `vnetName` in `main.bicepparam` on every retry.**

## Deployment

**Always use the official Bicep template:**
[Private Network Standard Agent Setup Bicep](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/15-private-network-standard-agent-setup)

> ⚠️ **Warning:** Capability host provisioning is **asynchronous** (10–20 minutes). Poll deployment status until success before proceeding.

## Expected Resource Progression

1. `Microsoft.Network/virtualNetworks` → Succeeded
2. `Microsoft.CognitiveServices/accounts` → Succeeded
3. `Microsoft.Search/searchServices` → Succeeded
4. `Microsoft.Storage/storageAccounts` → Succeeded
5. `Microsoft.DocumentDB/databaseAccounts` → Succeeded
6. `Microsoft.Network/privateEndpoints` (×5) → Succeeded
7. `Microsoft.MachineLearningServices/workspaces` → Succeeded
8. `Microsoft.MachineLearningServices/workspaces/capabilityHosts` → Succeeded (async, longest)

## Post-Deployment

1. **Deploy a model** to the new AI Services account (e.g., `gpt-4o`). Fall back to `Standard` SKU if `GlobalStandard` quota is exhausted.
2. **Create the agent** using MCP tools (`agent_update`) or the Python SDK.


## Known Issues

| Issue | Impact | Status |
|-------|--------|--------|
| `legionservicelink` on agent subnet cannot be removed by user after partial deployment | Subnet permanently blocked; must use new VNet on retry | Open — requires Container Apps service team |
| Bicep warning BCP037: `capabilityHostKind` not in type library | Cosmetic — ARM accepts the property, deploy succeeds | Pending Bicep type update |
| Hardcoded `core.windows.net` in template | Blocks sovereign cloud deployments (Azure Gov, China) | Open — should use `environment().suffixes.storage` |

## References

- [Azure AI Foundry Networking](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-private-link?tabs=azure-portal&pivots=fdp-project)
- [Azure AI Foundry RBAC](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/rbac-azure-ai-foundry?pivots=fdp-project)
- [Standard Agent Setup (public network)](standard-agent-setup.md)
