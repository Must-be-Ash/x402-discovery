# The Bazaar `/discovery/resources` shape (mirror this exactly — do not invent fields)

x402's canonical discovery is the **Bazaar**: a facilitator that supports it serves
`GET {facilitator_url}/discovery/resources`, returning **a list of resource objects**. This sub-skill
produces the *same shape*, self-hosted at the provider's own origin, scoped to their own endpoints.

## One resource object (the authoritative shape)

```json
{
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "amount": "200",
      "payTo": "0xa2477E16dCB42E2AD80f03FE97D7F1a1646cd1c0",
      "maxTimeoutSeconds": 60,
      "extra": { "name": "USD Coin", "version": "2" }
    }
  ],
  "extensions": {
    "bazaar": {
      "info": {
        "input":  { "type": "http", "method": "GET", "queryParams": { "city": "San Francisco" } },
        "output": { "type": "json", "example": { "city": "San Francisco", "weather": "foggy", "temperature": 60 } }
      }
    }
  },
  "lastUpdated": "2025-08-09T01:07:04.005Z",
  "resource": "https://api.example.com/x402/weather",
  "type": "http",
  "x402Version": 2
}
```

The discovery response is a JSON **array** of these objects (one per paid resource).

## Field notes

- **`resource`** — the absolute URL of the endpoint on the provider's **own origin** (not the facilitator).
- **`type`** — `"http"` for HTTP endpoints. (`"mcp"` exists for MCP tools; out of scope here.)
- **`x402Version`** — always `2`.
- **`lastUpdated`** — ISO-8601 timestamp; for a runtime endpoint, "now" at request time.
- **`accepts[]`** — one entry per payment option. Fields mirror the 402 challenge's `accepts`:
  - `scheme` (`exact` | `upto` | `batch-settlement`), `network` (CAIP-2), `payTo`, `maxTimeoutSeconds`.
  - **`amount`** — the on-chain integer in the asset's **minor units** (USDC has 6 decimals, so `"200"` =
    $0.0002). Note: the route config carries a *human* `price` string (`"$0.001"`); the x402 SDK resolves
    it to `amount` + `asset` + `extra` when it builds payment requirements. **Reuse that resolution**
    (see `knowledge.md`) rather than hand-converting — that's how you stay shape-exact and drift-proof.
  - `asset` (token contract), `extra` (`{ name, version }` for the asset).
- **`extensions.bazaar.info`** — reuse what the route already declared via `declareDiscoveryExtension`:
  - `input`: `{ type: "http", method, queryParams }` for GET; for body routes it carries the body shape
    (`bodyType`/params) instead of `queryParams`.
  - `output`: `{ type: "json", example, schema? }` — `example` required for indexability, `schema` optional.

## Where the data comes from (single source of truth)

Most of the above already exists in the provider's x402 **route config** (`payments.ts` / `proxy.ts`):
`scheme/network/payTo/price` from `accepts[]`, and the `output` from the route's
`declareDiscoveryExtension(...)`. Two caveats: (1) `amount`/`asset`/`extra`/`maxTimeoutSeconds` are
resolved by the SDK's payment-requirements builder (often defaults) — get them from there, don't
hand-author them; (2) **`input` is frequently absent** (routes often declare only `output` + `bodyType`
+ `pathParamsSchema`) — when so, derive the call shape and don't fabricate a required input schema (see
`knowledge.md` §1). The projection is a faithful re-rendering of the config — never a new source of truth.
