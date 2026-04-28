# Customer Questions & Concerns on VNet Setup

## 1. Foundry Agent Runtime & VNet Injection

- Does the Foundry Agent runtime actually run inside the customer VNet, or is it accessed purely via Private Link?
- If agents are VNet-injected, does that mean Private Link is no longer required for agent execution?
- Can some agents run public while others are private, or is it an all-or-nothing model once VNet injection is enabled?
- Does the Foundry service (PaaS endpoint) still require a Private Endpoint even if agents run inside the VNet?

## 2. Subnet Design & Delegation

- Do Foundry Agents require a dedicated subnet, or can they share a subnet with Azure Container Apps?
- What subnet delegation is required for Foundry Agent runtime (and is it the same as Container Apps)?
- Once a subnet is delegated to Foundry Agents, can anything else run on that subnet, or is it exclusive?
- If customers have multiple Foundry projects, do they need one subnet per project or can projects share a runtime/subnet?

## 3. Subnet Sizing & Scaling

- What is the recommended subnet size for Foundry Agent runtime?
- How should subnet sizing account for:
  - Number of agents
  - Agent runtimes / capability host
  - Scaling behavior (IPs consumed per runtime)?
- Is there a hard limit on:
  - Number of agents per project
  - Number of projects per Foundry account?

## 4. Identity, Runtime, and Project Boundaries

- What is the relationship between Foundry Project, Capability Host, and Agent Runtime?
- Is there a one-to-one or one-to-many relationship between:
  - Project ↔ runtime
  - Runtime ↔ agents?
- Which identity is used by agents by default (project identity vs agent identity)?

## 5. Firewall, NSG, and Egress Rules

- Are there official NSG / firewall rule requirements published specifically for Foundry Agent subnets (similar to Container Apps)?
- Should customers reuse Container Apps firewall guidance, or are Foundry agents different?
- What outbound destinations must be explicitly allowed (AAD, model endpoints, storage, registry, etc.)?
- Are there service tags or FQDN allowlists customers should use for Foundry Agent egress?
- Does VNet injection assign a NIC/IP per runtime, and how does that impact firewall rules?

## 6. Private DNS & Name Resolution

- Do all Foundry-related endpoints resolve to private IPs when private networking is enabled?
- Are there any DNS zones customers must explicitly manage for Foundry Private Endpoints?

## 7. Traffic Flow & Architecture Clarity

- What is the exact call flow in a private setup:
  - Container Apps → Foundry service → Agent runtime in subnet?
- Is all traffic strictly HTTPS over 443, or are other ports required?

## 8. Teams / Bot Service Integration (Separate but Related)

- For architectures where agents are invoked from Microsoft Teams:
  - How can customers securely bridge public Teams traffic to a private Foundry Agent?
  - Is use of Azure Bot Service + APIM + private backend a supported or recommended pattern?
- Are there examples of Teams → Foundry (private) reference architectures?

## 9. Operational Constraints & Limitations

- Are there known limitations around:
  - Firewall TLS inspection
  - IP range restrictions (RFC1918 only)
  - Region alignment between VNet and Foundry resources?
- Once deployed, can the delegated subnet be changed, or is redeployment required?
