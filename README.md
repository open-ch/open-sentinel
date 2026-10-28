# Sentinel Integration Package

This package deploys Open Systems' ASIM (Advanced Security Information Model) log parsers and the Sentinel ingestion infrastructure they need into your Microsoft Sentinel workspace.

| File | Purpose |
| --- | --- |
| `mainTemplate.json` | The ARM template you deploy. Creates a Data Collection Endpoint, one Data Collection Rule per product, and the ASIM log parsers as Sentinel saved-search functions. |
| `manifest.json` | The authoritative inventory of this package: its version, the literal Azure resources it provisions (Data Collection Endpoint, Data Collection Rules, role assignments), every deployed parser function (under `resources.savedSearches`), and a checksum of `mainTemplate.json`. |
| [`asim-registration.md`](asim-registration.md) | Generated, schema-specific union entries for the workspace-managed filtering and non-filtering ASIM custom unifiers. |
| [`RELEASE_TEMPLATE.md`](RELEASE_TEMPLATE.md) | The template Open Systems uses to prepare consistent customer-facing GitHub release notes. It is not part of the Azure deployment. |
| `README.md` | This file. |

## Overview of the process

Deploying this package requires the following steps; each step has its own section below.

1. **Create the Sender Principal** — record its tenant ID, application/client ID, service-principal object ID, and client-secret value.
2. **Deploy the template** — run `mainTemplate.json` into your Sentinel resource group, using the service-principal object ID.
3. **Share the connection details with Open Systems** — provide the deployment outputs, tenant ID, application/client ID, and client-secret value through the agreed secure channel.
4. **Register and verify the ASIM parsers** — add Open Systems to your workspace's custom unifiers without replacing other vendors' registrations, then validate ingestion after Open Systems enables forwarding.

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
  - Non-filtering counterparts for sources and vendor unions (`ASim<Schema>OpenSystems...`)
  - Legacy Open Systems bridge functions (`vim<Schema>Custom`), kept for compatibility and distinct from the workspace-managed `Im_<Schema>Custom` functions

For the authoritative, versioned inventory of all deployed functions and resources, see `manifest.json`.

The template provisions all products listed above; provisioning a Data Collection Rule does not enable delivery or establish a subscription entitlement. Agree the enabled log types and arrange forwarding with your Open Systems contact.

## Before you deploy

Make sure all prerequisites below are met before deploying this package:

- **A target resource group must already exist.** This package is deployed with `az deployment group ...` and the ARM template does not create resource groups.
- **A Microsoft Sentinel-enabled Log Analytics workspace must already exist** in the same Azure subscription you will deploy this package into. This package does not create a Log Analytics workspace or enable Sentinel.
  - The resource group that contains that workspace must also already exist, and you must know both the workspace name and its resource group name (`workspaceName` and `workspaceResourceGroupName`).
- **An Entra ID service principal that you create** (the *Sender Principal*) must already exist in the tenant containing the target Azure subscription. Use its **Enterprise application object ID**, not the App registration object ID or application/client ID, for `senderPrincipalId`.
- **Deployment permissions for the deployer identity:** the user or automation identity that runs the `az deployment group ...` commands needs `Contributor` on the target resource group(s), plus permission to create role assignments (`Microsoft.Authorization/roleAssignments`) scoped to the Data Collection Rules this package creates. This is separate from the Sender Principal used for ingestion.

## Create the Sender Principal

The Sender Principal is a dedicated application identity that Open Systems uses to authenticate when sending logs into your Azure environment. It is not a user account or the identity deploying the template. The template assigns it **Monitoring Metrics Publisher on each deployed Data Collection Rule**; do not grant it subscription-wide Contributor or Owner access for ingestion.

In the Azure portal:

1. Open **Microsoft Entra ID > App registrations > New registration** in the target tenant. Create a dedicated, single-tenant application for Open Systems log ingestion.
2. On the App registration's **Overview** page, record **Directory (tenant) ID** and **Application (client) ID**.
3. Open **Microsoft Entra ID > Enterprise applications** and find the application by its Application ID. Record the **Object ID** shown on this Enterprise application's Overview page. This is the service-principal object ID used as `senderPrincipalId`; the Object ID on the App registration is a different identifier.
4. Return to the **App registration > Certificates & secrets > Client secrets > New client secret**. Choose an expiry that meets your organization's policy and record the expiry date.
5. Copy the new secret's **Value** immediately into your approved secret store. Azure only displays the value when the secret is created. If it was not saved, create a replacement secret.

| Value | Where to find it | Used for |
| --- | --- | --- |
| Directory (tenant) ID | App registrations > your application > Overview | Share as `SP_TENANT_ID` so Open Systems authenticates against the correct tenant |
| Application (client) ID | App registrations > your application > Overview | Share as `SP_CLIENT_ID` to identify the application |
| Service-principal Object ID | Enterprise applications > your application > Overview | ARM deployment parameter `senderPrincipalId` |
| Client-secret **Value** | App registrations > your application > Certificates & secrets | Share securely as `SP_CLIENT_SECRET` to authenticate the application |

**Use the client-secret Value, not the Secret ID.** Do not provide your Azure password, a workspace shared key, or a secret belonging to another application. The tenant ID and client ID must belong to the same application identity as the secret and the service principal receiving the DCR permissions.

The tenant ID, client ID, and client-secret value are needed by Open Systems for forwarding, but are **not ARM template parameters**. Do not put the client secret into the template, deployment parameters, deployment outputs, tickets, or source control. Arrange rotation with your Open Systems contact before expiry: provide the replacement value securely, validate forwarding with it, then revoke the old secret.

## Deploying the package

Use the package from an agreed, versioned [GitHub release](https://github.com/open-ch/open-sentinel/releases), rather than unreleased files on `main`.

This package deploys at the **resource group** scope (`az deployment group ...`). By default, everything is created in the same resource group as your Sentinel workspace; run the commands below with `--resource-group <your-workspace-resource-group>`.

The Data Collection Endpoint created by this package supports only **public network access**; there is no private-link option today.

### Azure portal

Search for **Deploy a custom template** in the Azure portal, select **Build your own template in the editor**, then upload the released `mainTemplate.json` or paste its contents and select **Save**. Select the target subscription and resource group, enter `workspaceName`, `workspaceResourceGroupName`, and the Enterprise application object ID as `senderPrincipalId`, then review the deployment before creating it. Use the same values described in the CLI examples below. After deployment, open the deployment's **Outputs** to retrieve `collectorConnection`.

### Azure CLI

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

This output is **non-secret** — it contains only the DCE ingestion URL and immutable DCR IDs, one `DCR_<PRODUCT>_ID` entry per product this package deploys. It does not contain the tenant ID, client ID, or client-secret value. Send your Open Systems contact the following checklist, using the agreed secure channel for credentials:

| Connection detail | Value to provide |
| --- | --- |
| `SP_TENANT_ID` | Directory (tenant) ID of the Sender Principal |
| `SP_CLIENT_ID` | Application (client) ID, **not** either Object ID |
| `SP_CLIENT_SECRET` | Client-secret **Value**, **not** Secret ID; also communicate its expiry date |
| `DCE_URL` | `collectorConnection.DCE_URL`, the logs ingestion endpoint |
| `DCR_<PRODUCT>_ID` | Immutable `dcr-...` ID for each agreed log type, for example `DCR_FIREWALL_ID` and `DCR_PROXY_ID`; not the Azure resource ID |

Share the full `collectorConnection` output and confirm which products should be enabled. Open Systems will configure forwarding for the agreed log types. Deploying the template alone does not start log delivery.

## How the parsers connect to ASIM

This package deploys source parsers and Open Systems vendor unions. You register the vendor unions in the workspace-managed custom functions so the logs become reachable through Microsoft's standard ASIM entry points:

- **Source parser** (`vim<Schema>OpenSystems<Product>`) — normalizes one Open Systems product's logs into a specific ASIM schema.
- **Open Systems union** (`vim<Schema>OpenSystems`) — unions every Open Systems source parser for that schema into one function.
- **Legacy bridge** (`vim<Schema>Custom`) — references the Open Systems union. Existing registrations keep working; use the vendor-qualified union for new registrations.
- **Non-filtering counterparts** (`ASim<Schema>OpenSystems...`) — call the filtering functions with their unfiltered defaults; these counterparts take no parameters.

```text
_Im_<Schema>()                             <- Microsoft built-in unifier (do not author or redeploy)
  |__ Im_<Schema>Custom                    <- workspace-managed custom unifier (not deployed by this package)
        |__ other vendor parsers          <- preserve existing registrations
        |__ vim<Schema>OpenSystems         <- Open Systems vendor union (this package deploys it)
              |__ vim<Schema>OpenSystems<Product>   <- per-source parsers
```

Follow [Microsoft's ASIM parser-management instructions](https://learn.microsoft.com/en-us/azure/sentinel/normalization-manage-parsers) to create the workspace custom unifiers if needed and edit their union lists. Use the exact schema-specific entries in [asim-registration.md](asim-registration.md), preserving existing vendor references. Register the filtering `vim<Schema>OpenSystems` union in `Im_<Schema>Custom` and the non-filtering `ASim<Schema>OpenSystems` union in `ASim_<Schema>Custom`.

**Upgrading an existing registration:** installing new functions does not change your workspace-managed custom unifiers. Update their calls to forward all filtering parameters by name, using [asim-registration.md](asim-registration.md). If a custom unifier already references `vim<Schema>Custom` or individual Open Systems source parsers, replace that entry rather than adding a second path. Register each schema exactly once per entry-point family to prevent duplicate events. The existing bridge functions and `(starttime, endtime, disabled)` positional calls remain supported.

The filtering parsers support the schema-specific ASIM filters, including `eventresult` for Network Session. `pack` is not implemented and must not be forwarded. Empty lists and wildcard defaults leave filters unrestricted; supplying a filter for a field absent from a source excludes that source's non-matching events. DNS defaults to all event types (`eventtype="*"`) to preserve previous direct-call behavior; Microsoft's unifier can explicitly pass its own event-type default.

> **Overwrite caveat:** deployment reapplies the packaged definitions of all functions listed in `manifest.json`, including `vim<Schema>Custom`. Do not store other vendors' registrations or local customizations in those functions. Keep them in your workspace-managed `Im_<Schema>Custom` functions instead. If you already use a colliding function name, contact Open Systems before deploying.

## Verify the deployment

First, confirm the deployed function names listed in `manifest.json` under `resources.savedSearches` exist under **Log Analytics > Functions**. After Open Systems enables forwarding, check that the new pipeline delivers data for an enabled product:

```kusto
OTelLogs
| where TimeGenerated > ago(1h)
| where tostring(ResourceAttributes["osag.product.name"]) == "firewall"
| summarize Events=count()
```

Use `osag.product.name`, not the obsolete `osag.log_type` attribute. Replace `firewall` with the enabled product you are validating.

After registering the custom unifiers, run these queries in **Logs** to check that the ASIM entry points resolve:

```kusto
_Im_AlertEvent() | take 10
_Im_Authentication() | take 10
_Im_DhcpEvent() | take 10
_Im_Dns() | take 10
_Im_NetworkSession() | take 10
_Im_WebSession() | take 10
```

For the non-filtering family, verify `_ASim_NetworkSession()` and the corresponding `_ASim_<Schema>()` entry points after registering their custom unifiers. For example, `_Im_NetworkSession(eventresult="Failure") | summarize count() by EventResult` must not include Success events.

A successful query with no rows only proves that the entry point resolves; it does not prove that Open Systems is registered or sending data. For an enabled Firewall source, compare the source-specific parser and the standard entry point over the same time range:

```kusto
vimNetworkSessionOpenSystemsFirewall()
| where TimeGenerated > ago(1h)
| summarize Events=count()
```

```kusto
_Im_NetworkSession()
| where TimeGenerated > ago(1h)
| where EventVendor == "Open Systems" and EventProduct == "Firewall"
| summarize Events=count()
```

Investigate unexpected differences, duplicate registrations, or absent rows before relying on the deployment. Check that existing non-Open Systems parser results remain available too.

## Redeploying, rolling back, and cleaning up

- **Review redeployments with `what-if`.** The template uses incremental deployment: it creates or updates declared resources and leaves unrelated resources alone. It reapplies packaged parser definitions, so manual edits to those functions can be overwritten; workspace-managed `Im_<Schema>Custom` registrations are not part of this deployment.
- **Your Sender Principal should not change on a routine redeploy.** If it needs to be rotated or replaced, that is a deliberate migration — talk to your Open Systems contact before changing `senderPrincipalId`.
- **To roll back**, redeploy the previous version of this package with the same parameters.
- **Cleaning up**: if a newer version of this package no longer includes a product you had before, its Data Collection Rule or parser is not deleted automatically — redeploying only creates or updates what the current template declares. Remove anything no longer needed yourself, or ask your Open Systems contact for help.

## Questions or issues

Contact your Open Systems representative for help with deploying or upgrading this package.
