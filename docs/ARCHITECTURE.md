# Architecture & Design Notes

Deeper context on why this connector is built the way it is, and what's 
worth knowing before extending it.

## CCF in 30 seconds

The Codeless Connector Framework (CCF) is Microsoft Sentinel's declarative 
data ingestion system. You describe how to call an API in JSON, and 
Sentinel's managed poller service does the calling, ingestion routing, and 
health monitoring. No Azure Functions, no Logic Apps, no custom code.

The four building blocks of every CCF pull connector:

1. **Custom table** : where the data lands in Log Analytics
2. **Data Collection Rule (DCR)** : KQL transform that maps API fields to 
   table columns
3. **Connector UI Definition** : the card in the Sentinel Data Connectors 
   gallery
4. **RestApiPoller** : the actual polling configuration (auth, endpoint, 
   request body, response parsing)

This connector ships components 3 and 4 in `connector/connector.json`, 
and components 1 and 2 in `infrastructure/dcr-and-table.json`.

## Why credentials are ARM parameters, not UI inputs

Microsoft's CCF V2 documentation describes a "polished" Connect button in 
the Sentinel UI where users type credentials into a form. Unfortunately, 
that pattern only works for connectors **published through Microsoft's 
Content Hub** as part of a solution package. For custom community-built 
ARM templates deployed directly, the UI Connect button doesn't have the 
backend wiring needed to substitute credentials and create the poller 
resource.

Three things were tried during development of this connector before 
arriving at the deploy-time-credentials design:

1. **Embedded `pollingConfig`** inside the UI definition (legacy CCF V1 
   pattern) — the modern Sentinel UI doesn't read this field
2. **`isActive: false` on a UI-parameter-bound poller** — the connectivity 
   check still runs at deploy time and fails because the credentials are 
   placeholder strings
3. **Two-step deployment with separate UI Connect** — only works if the 
   connector is registered in a Content Hub solution

The architecture that actually works for community-built CCF connectors:

- Credentials supplied as ARM parameters at deploy time
- API token marked as `SecureString` so Azure masks it and never logs it
- Connector polls immediately after deployment with no UI Connect step
- Credential rotation is just a redeploy with new parameter values

This pattern is widely used in community CCF connectors on GitHub (e.g. 
`Azure/Azure-Sentinel/Solutions/*/Data Connectors`).

## GraphQL specifics

Cloudflare's API is GraphQL, which means:

- HTTP method is POST (CCF's default GET would not work)
- Request body is JSON with a `query` field containing the GraphQL query
- `queryParametersTemplate` in CCF holds the JSON body as a string
- `isPostPayloadJson: true` tells CCF to send the body as JSON rather than 
  form-encoded

The GraphQL query embeds the Zone ID via ARM `concat()` at deploy time. 
Time window placeholders `{_QueryWindowStartTime}` and `{_QueryWindowEndTime}` 
are CCF's runtime substitution syntax, they're replaced by the poller 
on each call with ISO-8601 timestamps.

## JSONPath quirk

Cloudflare's response structure is:

```json
{
  "data": {
    "viewer": {
      "zones": [
        { "firewallEventsAdaptive": [ ... ] }
      ]
    }
  }
}
```

The expected `eventsJsonPaths` for nested arrays would be 
`$.data.viewer.zones[*].firewallEventsAdaptive[*]`. However, CCF's 
JSONPath implementation has inconsistent support for double wildcards.

This connector uses `$.data.viewer.zones[0].firewallEventsAdaptive[*]` 
because the query filters by a single `zoneTag` — `zones` will always 
contain exactly one element.

If you extend this connector to query multiple zones, you'll need to 
either deploy multiple connector instances (one per zone) or test 
whether the double-wildcard works in your environment.

## Data loss risk and pagination

Time-windowed polling with `limit: 1000` per call has known limits:

- For 1-minute windows, sustained traffic up to 16-17 events/second is 
  safe
- Bursts above the limit drop the **oldest** events (because the query 
  uses `orderBy: [datetime_DESC]`)

The connector does NOT implement true pagination. CCF supports several 
pagination types (`LinkHeader`, `NextPageToken`, `Offset`, 
`PersistentToken`) but Cloudflare's `firewallEventsAdaptive` doesn't 
return cursor metadata in its response.

A future version could implement cursor-based pagination via 
`PersistentToken` where the token is the timestamp of the oldest event 
returned, and subsequent calls use `datetime_lt: <that_timestamp>` to 
fetch older events. This is non-trivial to get right and is left as a 
future improvement.

## DCR transform design

The DCR transform uses `extend` rather than `project` , this is 
deliberate. If a field is missing from the API response (which can 
happen for less-active zones or under certain Cloudflare plan tiers), 
`extend` produces null whereas `project` causes the entire transform 
to fail and silently drops every row.

The transform also adds aliased columns (`Referer`, `Url`, 
`HttpUserAgent`, `HttpHost`) that mirror Sentinel's standard schema 
naming conventions. This makes the data easier to integrate with 
ASIM (Advanced Security Information Model) parsers if needed.

## Stream column type for `datetime`

In the DCR `streamDeclarations`, the `datetime` field is declared as 
`string`, not `datetime`. This is intentional, Cloudflare returns 
datetime as an ISO-8601 string, and CCF's stream type coercion can 
silently drop records when the format doesn't perfectly match Azure's 
expected datetime layout.

Declaring it as `string` defers conversion to the transform, where 
`todatetime()` handles ISO-8601 robustly.

## Future improvements

Areas where contributions would be valuable:

- **OAuth2 auth path** : for organisations using Cloudflare API 
  Tokens via OAuth instead of static Bearer tokens
- **PersistentToken pagination** : for high-volume zones
- **Multi-zone support** : single connector polling N zones, with 
  zone metadata in the output
- **Additional Cloudflare datasets** : `httpRequestsAdaptiveGroups`, 
  `dnsRequests`, `accessRequests`
- **Bicep variant** : for organisations standardising on Bicep
- **GitHub Actions workflow** : for CI validation of template changes
