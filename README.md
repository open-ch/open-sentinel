# Sentinel Integration Package

This package deploys Open Systems' ASIM (Advanced Security Information Model) log parsers and the Sentinel ingestion infrastructure they need into your Microsoft Sentinel workspace.

| File | Purpose |
| --- | --- |
| `mainTemplate.json` | The ARM template you deploy. Creates a Data Collection Endpoint, one Data Collection Rule per product, and the ASIM log parsers as Sentinel saved-search functions. |
| `manifest.json` | The authoritative inventory of this package: its version, the literal Azure resources it provisions (Data Collection Endpoint, Data Collection Rules, role assignments), every deployed parser function (under `resources.savedSearches`), and a checksum of `mainTemplate.json`. |
| [`RELEASE_TEMPLATE.md`](RELEASE_TEMPLATE.md) | The template Open Systems uses to prepare consistent customer-facing GitHub release notes. It is not part of the Azure deployment. |
| `README.md` | This file. |

## Overview of the process

Deploying this package is a three-step flow; each step has its own section below.

1. **Deploy the template** — run `mainTemplate.json` into your Sentinel resource group.
2. **Share the outputs with Open Systems** — the deployment prints the ingestion endpoint and Data Collection Rule IDs; send these back, along with the Sender Principal credentials (separately and securely).
3. **Verify the deployment** — confirm the ASIM parsers resolve in your workspace.

## What this package deploys

When deployed successfully, this package creates and configures the following resources in your Azure environment:

- One Data Collection Endpoint: `opensystems-otlp-dce`.
- One Data Collection Rule per supported Open Systems product:
  - `opensystems-auth-dcr`
  - `opensystems-dhcp-dcr`
  - `opensystems-dns_inspection-dcr`
  - `opensystems-firewall-dcr`
  - `opensystems-mep-dcr`
  - `opensystems-ndr-dcr`
  - `opensystems-proxy-dcr`
  - `opensystems-zeek-dcr`
  - `opensystems-ztna-dcr`
- Role assignments on those Data Collection Rules (Monitoring Metrics Publisher) for the Sender Principal.
- Sentinel saved-search functions for ASIM parsing:
  - Product-specific source parsers (`vim<Schema>OpenSystems<Product>`)
  - Per-schema Open Systems unifying parsers (`vim<Schema>OpenSystems`)
  - Per-schema ASIM customization seam functions (`vim<Schema>Custom`)

For the authoritative, versioned inventory of all deployed functions and resources, see `manifest.json`.

## Before you deploy

Make sure all prerequisites below are met before deploying this package:

- **A target resource group must already exist.** This package is deployed with `az deployment group ...` and the ARM template does not create resource groups.
- **A Microsoft Sentinel-enabled Log Analytics workspace must already exist** in the same Azure subscription you will deploy this package into. This package does not create a Log Analytics workspace or enable Sentinel.
  - The resource group that contains that workspace must also already exist, and you must know both the workspace name and its resource group name (`workspaceName` and `workspaceResourceGroupName`).
- **An Entra ID service principal that you create** (the *Sender Principal*) must already exist. You need its **object ID** (not its application/client ID) to deploy this package.
- **Deployment permissions for the deployer identity:** the user or automation identity that runs the `az deployment group ...` commands needs `Contributor` on the target resource group(s), plus permission to create role assignments (`Microsoft.Authorization/roleAssignments`) scoped to the Data Collection Rules this package creates. This is separate from the Sender Principal used for ingestion.

## Deploying the package

This package deploys at the **resource group** scope (`az deployment group ...`). By default, everything is created in the same resource group as your Sentinel workspace; run the commands below with `--resource-group <your-workspace-resource-group>`.

The Data Collection Endpoint created by this package supports only **public network access**; there is no private-link option today.

```bash
az deployment group validate \
  --resource-group <your-workspace-resource-group> \
  --template-file mainTemplate.json \
  --parameters \
    workspaceName=<your-log-analytics-workspace> \
    workspaceResourceGroupName=<your-workspace-resource-group> \
    senderPrincipalId=<your-sender-principal-object-id>
```

Review the exact change set before applying it:

```bash
az deployment group what-if \
  --resource-group <your-workspace-resource-group> \
  --template-file mainTemplate.json \
  --parameters \
    workspaceName=<your-log-analytics-workspace> \
    workspaceResourceGroupName=<your-workspace-resource-group> \
    senderPrincipalId=<your-sender-principal-object-id>
```

Then deploy:

```bash
az deployment group create \
  --name sentinel-package-$(date +%Y%m%d%H%M%S) \
  --resource-group <your-workspace-resource-group> \
  --template-file mainTemplate.json \
  --parameters \
    workspaceName=<your-log-analytics-workspace> \
    workspaceResourceGroupName=<your-workspace-resource-group> \
    senderPrincipalId=<your-sender-principal-object-id>
```

## After deployment: share the output with Open Systems

Retrieve the deployment's outputs (use the same `--resource-group` you deployed with):

```bash
az deployment group show \
  --resource-group <your-workspace-resource-group> \
  --name <deployment-name> \
  --query properties.outputs
```

```json
{
  "collectorConnection": {
    "value": {
      "DCE_URL": "https://....ingest.monitor.azure.com",
      "DCR_NDR_ID": "dcr-...",
      "DCR_ZEEK_ID": "dcr-..."
    }
  }
}
```

This output is **non-secret** — it contains only the DCE ingestion URL and immutable DCR IDs, one `DCR_<PRODUCT>_ID` entry per product this package deploys. It does not contain your Sender Principal's credentials. Share this output, along with your Sender Principal's credentials (through a secure channel), with your Open Systems contact — they will use it to configure log ingestion into the correct Data Collection Rules.

## How the parsers connect to ASIM

This package deploys three layers of ASIM parser functions that chain together so your Open Systems logs become reachable through Microsoft's standard ASIM entry points:

- **Source parser** (`vim<Schema>OpenSystems<Product>`) — normalizes one Open Systems product's logs into a specific ASIM schema.
- **Open Systems union** (`vim<Schema>OpenSystems`) — unions every Open Systems source parser for that schema into one function.
- **Customization seam** (`vim<Schema>Custom`) — the ASIM extension point Microsoft's built-in unifier looks for. Microsoft's `_Im_<Schema>` parser **automatically** unions `vim<Schema>Custom` when it exists, so deploying this seam is what wires Open Systems data into the standard ASIM parser. **There is no manual registration step.**

```text
_Im_<Schema>()                             <- Microsoft built-in unifier (do not author or redeploy)
  |__ vim<Schema>Custom                    <- customization seam (this package deploys it)
        |__ vim<Schema>OpenSystems         <- Open Systems union (this package deploys it)
              |__ vim<Schema>OpenSystems<Product>   <- per-source parsers (this package deploys them)
```

> **Overwrite caveat:** the customization seam is a single function per schema (`vim<Schema>Custom`). If you already maintain your own `vim<Schema>Custom` for a schema, deploying this package **overwrites** it. Contact Open Systems **before** deploying so we can merge your existing customizations into the deployed function.

## Verify the deployment

After deploying, confirm each ASIM schema resolves through Microsoft's built-in unifier. Run each query in **Logs**; a result — or an empty result with no error — means the parser chain is wired correctly. (Data only appears once Open Systems begins sending logs.)

```kusto
_Im_AlertEvent() | take 10
_Im_Authentication() | take 10
_Im_DhcpEvent() | take 10
_Im_Dns() | take 10
_Im_NetworkSession() | take 10
_Im_WebSession() | take 10
```

To confirm the individual functions were created, check the deployed function names listed in `manifest.json` under `resources.savedSearches`, or open **Log Analytics → Functions** and filter for `vim`. Every function listed there should exist in your workspace after a successful deployment.

## Redeploying, rolling back, and cleaning up

- **Redeploying is safe.** Running `az deployment group create` again with a newer version of this package updates only what changed; anything else in your resource group(s) is left alone.
- **Your Sender Principal should not change on a routine redeploy.** If it needs to be rotated or replaced, that is a deliberate migration — talk to your Open Systems contact before changing `senderPrincipalId`.
- **To roll back**, redeploy the previous version of this package with the same parameters.
- **Cleaning up**: if a newer version of this package no longer includes a product you had before, its Data Collection Rule or parser is not deleted automatically — redeploying only creates or updates what the current template declares. Remove anything no longer needed yourself, or ask your Open Systems contact for help.

## Questions or issues

Contact your Open Systems representative for help with deployment, migrating from a previous integration, or anything else in this package.
