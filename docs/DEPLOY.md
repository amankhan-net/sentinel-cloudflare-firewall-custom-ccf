# Deployment Guide

Detailed step-by-step deployment instructions for the Cloudflare Firewall 
CCF connector.

## Prerequisites

Before starting, ensure you have:

1. **Microsoft Sentinel** enabled on a Log Analytics workspace
2. **Contributor** or higher role on the resource group
3. **Cloudflare account** with at least one zone (domain) configured
4. The zone has firewall rules enabled and is receiving traffic (otherwise 
   there's nothing to ingest — verify with curl first)

## Step 1 : Create a Cloudflare API token

1. Log in to [https://dash.cloudflare.com](https://dash.cloudflare.com)
2. Click your profile icon → **My Profile** → **API Tokens**
3. Click **Create Token** → **Create Custom Token**
4. Configure:
   - Token name: `Sentinel Firewall Events`
   - Permissions:
     - Account → Account Analytics → Read
     - Zone → Zone Analytics → Read
   - Account Resources: Include your account
   - Zone Resources: Include the specific zone you want to monitor
   - TTL: optionally set an expiry
5. Click **Continue to summary** → **Create Token**
6. **Copy the token immediately** : it's only shown once
7. Test the token works:

```bash
curl -X POST https://api.cloudflare.com/client/v4/graphql \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ viewer { zones(filter: { zoneTag: \"YOUR_ZONE_ID\" }) { firewallEventsAdaptive(filter: { datetime_geq: \"2026-05-01T00:00:00Z\", datetime_leq: \"2026-05-01T23:59:59Z\" }, limit: 5) { action clientIP datetime ruleId } } } }"}'
```

If you get a JSON response with events (or an empty array), the token works.

## Step 2 : Find your Cloudflare Zone ID

1. In Cloudflare dashboard, click on your domain
2. On the **Overview** page, scroll down to the **API** section in the right sidebar
3. Copy the **Zone ID** (a 32-character hex string)

## Step 3 : Deploy the infrastructure

The infrastructure template creates:
- A Data Collection Endpoint (DCE)
- A custom Log Analytics table `cloudflarefirewall_CL`
- A Data Collection Rule (DCR) with the transform KQL

```
Azure Portal → search "Deploy a custom template" → Build your own template 
in the editor → paste infrastructure/dcr-and-table.json → Save
```

Fill in the parameters:

| Parameter                | Value                                                  |
|--------------------------|--------------------------------------------------------|
| `workspaceName`          | Your Log Analytics workspace name                      |
| `workspaceResourceGroup` | Resource group of the workspace (defaults to current)  |
| `location`               | Region (defaults to resource group region)             |
| `dceName`                | `DCE-Cloudflare` (or your choice)                      |
| `dcrName`                | `DCR-cloudflare` (or your choice)                      |
| `tableName`              | `cloudflarefirewall_CL` (default — must end with `_CL`)|

Click **Review + create** → **Create**. Wait for deployment to complete (~30 seconds).

### Capture the deployment outputs

After deployment completes, click on the deployment in the portal and 
expand **Outputs**. You'll see:

| Output                     | What it is                                                  |
|----------------------------|-------------------------------------------------------------|
| `dceLogsIngestionEndpoint` | The DCE URL — used in the connector template                |
| `dcrImmutableId`           | The DCR ID — used in the connector template                 |
| `streamName`               | Stream name — defaults to `Custom-cloudflarefirewall_CL`    |
| `tableName`                | Table name — defaults to `cloudflarefirewall_CL`            |

Copy `dceLogsIngestionEndpoint` and `dcrImmutableId` , you'll paste them 
into the connector template in the next step.

If you missed the outputs page, you can also get these values manually:

- **DCR Immutable ID**: Resource group → `DCR-cloudflare` → Overview → 
  Immutable Id (format: `dcr-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`)
- **DCE Logs Ingestion endpoint**: Resource group → `DCE-Cloudflare` → 
  Overview → Logs Ingestion endpoint URL

## Step 4 : Deploy the connector

```
Azure Portal → Deploy a custom template → Build your own template in the editor 
→ paste connector/connector.json → Save
```

Fill in the parameters:

| Parameter                       | Value                                          |
|---------------------------------|------------------------------------------------|
| `workspaceName`                 | Name of your Log Analytics workspace           |
| `workspaceLocation`             | Auto-fills from resource group region          |
| `cloudflareApiToken`            | Paste from Step 1 (masked input)               |
| `zoneTag`                       | Zone ID from Step 2                            |
| `dataCollectionEndpoint`        | DCE URL from Step 3                            |
| `dataCollectionRuleImmutableId` | DCR Immutable ID from Step 3                   |
| `streamName`                    | `Custom-cloudflarefirewall_CL` (default)       |
| `queryWindowInMin`              | `1` (default)                                  |

Click **Review + create** → **Create**.

Deployment takes ~1-2 minutes. The connector performs a connectivity check 
against Cloudflare during deployment — if your token or Zone ID is wrong, 
deployment fails with a 400 error from Cloudflare.

## Step 5 : Validate

After deployment, the connector starts polling immediately. The first batch 
of events should appear within 1-30 minutes.

Run in Log Analytics:

```kusto
cloudflarefirewall_CL
| where TimeGenerated > ago(30m)
| sort by TimeGenerated desc
| take 10
```

If you see rows, the connector is working.

If after 30 minutes you see nothing, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

## Updating credentials or zone

To rotate the API token or change the zone:

1. Run the connector deployment again with the new parameter values
2. ARM updates the existing connector resources in place
3. Polling continues with the new credentials within ~1 minute

## Uninstall

To completely remove the connector:

1. Delete the `dataConnectors` resource via Azure CLI:

```bash
az rest --method delete \
  --uri "https://management.azure.com/subscriptions/{SUB_ID}/resourceGroups/{RG_NAME}/providers/Microsoft.OperationalInsights/workspaces/{WORKSPACE}/providers/Microsoft.SecurityInsights/dataConnectors/CloudflareFirewallEventsAdaptivePoller?api-version=2024-09-01"
```

2. Delete the `dataConnectorDefinitions` resource similarly
3. Optionally delete the DCR, DCE, and custom table if no longer needed
4. **Revoke the Cloudflare API Token** in the Cloudflare dashboard

Or delete the entire resource group if it was a sandbox.
