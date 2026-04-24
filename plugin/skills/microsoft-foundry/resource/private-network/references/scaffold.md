# Scaffold & Parameterize

Use this reference to fetch the confirmed template and wire up parameters.

If the user wants to adapt the Bicep into an existing repo, copy the relevant resources into their existing templates instead of using the official template directly.

If the user has no GitHub access, the template must already be present in the workspace. Do NOT attempt to fetch from GitHub.

If using default templates fetch the Bicep template from the GitHub URL in the template reference doc (loaded in Step 2). Fetch the **entire template folder** including subdirectories (e.g., `modules-network-secured/`). Create the files in the user's workspace (e.g., `infra/` folder).

Set parameter values in `main.bicepparam` using the answers collected in [information-gathering.md](information-gathering.md) and [requirement-gathering.md](../requirement-gathering.md):

| Parameter | Source |
|-----------|--------|
| Location | Region (or inferred from existing VNet) |
| VNet name / resource ID | VNet answer (new or existing) |
| VNet address space | Address space from requirements (default `192.168.0.0/16`) |
| Subnet CIDRs | Subnet answers (agent `/24`, PE `/24`, MCP `/24` if needed) |
| Existing Cosmos DB / Storage / AI Search IDs | BYO resource IDs (only if reusing) |
| Isolation mode (T18 only) | Managed VNet outbound mode (`AllowOnlyApprovedOutbound` or `AllowInternetOutbound`) |
| Model name, version, format | Model selection from requirements |
| `disableLocalAuth` | Set `true` if Azure Policy requires it |

> Do NOT run `az deployment group create` yet — validate first (next step).
