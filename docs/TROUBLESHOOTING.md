# Troubleshooting

Diagnostic guide for common issues with the Cloudflare CCF connector. 
Work through the layers in order,stop at the first one that fails.

## Layer 1 : Did the deployment succeed?

If the ARM template deployment failed with a connectivity check error:

### `BadRequest: Call failed with status code 400`

Cloudflare rejected the request. Common causes:
- Wrong API token (typo, expired, or revoked)
- API token missing required permissions (`Account Analytics:Read` and 
  `Zone Analytics:Read`)
- Wrong Zone ID format (must be a 32-character hex string from the zone's 
  Overview page, not the domain name)

**Test directly with curl** (see [DEPLOY.md](DEPLOY.md) Step 1).

### `BadRequest: Call failed with status code 401` or `403`

API token is invalid or doesn't have permission for the zone you specified. 
Recreate the token, ensuring the Zone Resources scope includes your zone.

### `InvalidTemplate: ... language expression evaluation failed`

The ARM template has a syntax issue. Confirm you copied the entire 
`connector/connector.json` file without truncation.

## Layer 2 : Did the SentinelHealth poller log the issue?

If the connector deployed successfully but no data is flowing, the 
`SentinelHealth` table is your best diagnostic source.

### Enable health monitoring (one-time setup)

Sentinel → Settings → **Auditing and health monitoring** → enable 
**Data collection — health monitoring**. Wait 30 minutes after enabling.

### Query for connector errors

```kusto
SentinelHealth
| where TimeGenerated > ago(2h)
| where SentinelResourceName contains "Cloudflare"
| project TimeGenerated, Status, Description, Reason
| sort by TimeGenerated desc
```

Look for:
- `Status = Failure` with descriptions like:
  - `transformKql failed` → DCR transform issue (Layer 4)
  - `column not found` → schema mismatch
  - `401`, `403` → auth issue (Layer 1)
  - `400` → API request issue (Layer 1)
  - `empty response` → API returns no data (Layer 3)

## Layer 3 : Does your Cloudflare zone have firewall events?

The API can return 200 OK with an empty array. CCF treats this as success 
and ingests nothing , which from your perspective looks identical to a 
broken connector.

### Verify with curl

Run a curl test against a wide time window (last 24 hours):

```bash
curl -X POST https://api.cloudflare.com/client/v4/graphql \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ viewer { zones(filter: { zoneTag: \"YOUR_ZONE_ID\" }) { firewallEventsAdaptive(filter: { datetime_geq: \"2026-05-01T00:00:00Z\", datetime_leq: \"2026-05-01T23:59:59Z\" }, limit: 10) { action clientIP datetime ruleId } } } }"}' \
  | python3 -m json.tool
```

If the response shows `"firewallEventsAdaptive": []`, your zone has no 
firewall events in that window. Either:
- The zone has no WAF rules enabled , enable Cloudflare's managed rules 
  to start generating events
- The zone has no traffic , wait for legitimate traffic, or test from 
  a known-bad IP to trigger a rule
- The Zone ID is wrong (check the zone's Overview page)

## Layer 4 : Is the DCR transform working?

If the API returns events but they don't appear in `cloudflarefirewall_CL`, 
the DCR transform is failing.

### Check the DCR's stream name matches

Your connector parameter `streamName` must match the DCR's 
`streamDeclarations` key exactly (case-sensitive). The default is 
`Custom-cloudflarefirewall_CL`.

In Azure Portal → Monitor → Data Collection Rules → your DCR → JSON view:

```json
"streamDeclarations": {
  "Custom-cloudflarefirewall_CL": { ... }   ← this key
},
"dataFlows": [
  {
    "streams": ["Custom-cloudflarefirewall_CL"],   ← and this value
    ...
    "outputStream": "Custom-cloudflarefirewall_CL"  ← and this value
  }
]
```

All three must be identical, and must match the connector's `streamName` 
parameter.

### Check the streamDeclarations include `datetime` as `string`

This is a common bug. Cloudflare returns `datetime` as an ISO-8601 string. 
If the stream declaration types it as `datetime` (instead of `string`), 
the transform may silently drop records.

In your DCR JSON, ensure:

```json
{ "name": "datetime", "type": "string" }
```

The transform's `todatetime(['datetime'])` converts it correctly.

### Check the transformKql

The transform KQL must reference fields that exist in the API response. 
If a field is missing or misspelled, the entire transform fails.

Safe pattern: use `extend` (which produces null for missing fields) 
instead of `project` (which fails on missing columns).

```kusto
source
| extend TimeGenerated = todatetime(['datetime'])
| extend Referer = clientRequestHTTPHost
| extend Url = clientRequestPath
| extend HttpUserAgent = userAgent
| extend HttpHost = clientRequestHTTPHost
| project-away ['datetime']
```

### Enable DCR diagnostics

Azure Portal → your DCR → **Diagnostic settings** → Add → check 
"DCR log errors" → Send to Log Analytics workspace → Save.

After 30 minutes:

```kusto
AzureDiagnostics
| where ResourceType == "DATACOLLECTIONRULES"
| where TimeGenerated > ago(2h)
| project TimeGenerated, Category, ResultDescription
| sort by TimeGenerated desc
```

This shows transform errors with the exact reason.

## Layer 5 : Region mismatch?

A DCR and DCE must be in the **same Azure region** as the Log Analytics 
workspace. Cross-region setups silently fail ingestion.

Check all three:

```
Azure Portal → workspace → Properties → Location
Azure Portal → DCR → Overview → Location
Azure Portal → DCE → Overview → Location
```

All three must show the same region (e.g. all "Australia East" or all 
"East US"). If different, recreate the misaligned resource in the correct 
region.

## Layer 6 : Connector resource state

If everything looks right but no polling is happening:

```
Azure Portal → resource group → filter type "dataConnectors" 
→ click CloudflareFirewallEventsAdaptivePoller → JSON view
```

Verify:
- `properties.isActive` is `true`
- `properties.connectorDefinitionName` matches the connector definition name
- `properties.dcrConfig.streamName` matches the DCR stream name
- `properties.auth.ApiKey` is populated (will appear masked as `***`)

## When all else fails

1. Delete the connector resources via Azure CLI:

```bash
az rest --method delete --uri "https://management.azure.com/subscriptions/{SUB_ID}/resourceGroups/{RG}/providers/Microsoft.OperationalInsights/workspaces/{WS}/providers/Microsoft.SecurityInsights/dataConnectors/CloudflareFirewallEventsAdaptivePoller?api-version=2024-09-01"

az rest --method delete --uri "https://management.azure.com/subscriptions/{SUB_ID}/resourceGroups/{RG}/providers/Microsoft.OperationalInsights/workspaces/{WS}/providers/Microsoft.SecurityInsights/dataConnectorDefinitions/CloudflareFirewallConnector?api-version=2024-09-01"
```

2. Verify the DCR and table are healthy by sending a test record via the 
   Logs Ingestion API directly (see Microsoft's 
   [Logs Ingestion API tutorial](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal)).

3. If a direct API send works, redeploy the connector with verified-working 
   credentials.

4. If a direct API send fails, the issue is in the DCR/DCE/table, not the 
   connector fix that first.
