# Projection rules — route config → origin-local discovery

The job: read the provider's **existing** x402 route config and re-render it as origin-local discovery
artifacts. **One projection, multiple renderings.** Parse the route map once into an internal model,
then render each artifact from that model so they can never disagree.

> This sub-skill never *creates* an x402 setup or picks schemes/prices — that's `x402-start`. It only
> projects what's already there. If there's no x402 config, stop and say "run x402-start first."

## 1. Parse the route config into a model

From `payments.ts` / `proxy.ts` (the `routes` / `RoutesConfig` object), for each route key
`"METHOD /path"` capture:
- `method`, `path`, `description`, `mimeType`
- each `accepts[]` entry: `scheme`, `network`, `payTo`, and `price` (plus `asset`/`extra`/
  `maxTimeoutSeconds` if set — but usually these are SDK defaults; get them from the SDK
  payment-requirements builder in §2, don't hand-author them)
- the bazaar **`output`** already declared via `declareDiscoveryExtension(...)` — **reuse it verbatim**
- **`input` is often NOT declared.** Routes commonly pass only `output` (plus `bodyType` and
  `pathParamsSchema`) and no `input` example — that's the case in typical setups. **Never fabricate a
  required input schema.** When `input` is absent, derive the *call shape* for the renderings from: the
  HTTP method, `pathParamsSchema` (path params), and the request-body fields the handler/pricing helper
  actually reads (e.g. `width`/`height` for image, `text`/`targetLang` for translate) — and mark these
  as **inferred** (not an authoritative declared schema). If `input` *is* declared, reuse it verbatim.
- classify the route:
  - **paid** — has `accepts[]` with a price → include, with pricing
  - **free/identity-gated** — empty `accepts[]` + a SIWX declaration (e.g. the poll endpoint) → include,
    marked `payment: "none — sign-in-with-x required"`
  - **unpaid** — not in the x402 map at all (e.g. `/health`) → **exclude** from discovery

## 2. Static vs dynamic price (the honesty rule)

- **Static price** (`price: "$0.001"`): resolvable to a concrete `amount`/`asset`. **Prefer reusing the
  x402 SDK's own payment-requirements builder** (the same call the middleware uses to build the 402) so
  the emitted `accepts[]` — including `amount` in minor units, `asset`, and `extra` — is byte-identical
  to what a real 402 returns. This is both shape-exact and drift-proof.
- **Dynamic price** (`price` is a **function**): there is no single number without a concrete request.
  **Never fabricate one.** Represent it as a model:
  ```json
  "price": { "kind": "dynamic", "quotedIn": "the 402 response for a concrete request",
             "model": "<from the route description / pricing.ts comment>", "example": "1MP ≈ $0.002" }
  ```
  Pull the human-readable model from the route `description` and the pricing helper's comments.
- **`upto` / `batch-settlement`**: the price is a **cap/max** (settles ≤ that). Label it as such
  (`"amountIsMax": true`) so consumers know it's a ceiling, not a fixed charge.

## 3. Render the artifacts (all from the one model)

### 3a. Primary — Bazaar-shaped JSON (see `bazaar-shape.md`)
- Prefer a **runtime endpoint**: add a free, unpaid `GET /discovery/resources` (alias
  `GET /.well-known/x402.json`) that reflects over the **same `routes` object** the payment middleware
  uses and emits the array of resource objects. Because it reads the live config per request, it
  **cannot drift**. For static-price routes it can even build `accepts[]` via the SDK requirement
  builder for exactness; dynamic routes get the pricing model from §2.
- If the team wants a **static file** instead, generate `.well-known/x402.json` from the config and note
  it must be regenerated on change (the runtime endpoint avoids this).
- `resource` URLs use the provider's own origin. `type: "http"`, `x402Version: 2`, `lastUpdated` = now.

### 3b. Companion — plain endpoint list (`llms.txt` / `AGENTS.md`)
A human- and agent-readable list. Per paid endpoint: `METHOD /path` — purpose (from `description`) —
how to call (query params for GET / JSON body fields for POST — from the bazaar `input` if declared,
otherwise the inferred call shape from §1) — cost (fixed price, or `dynamic — <model>`, or `free (SIWX)`). Lead with:
> "This API accepts x402 (v2) payments. Machine-readable discovery: `/discovery/resources`. Canonical
> cross-provider index: the x402 Bazaar."
Serve at `/.well-known/llms.txt` and/or as `AGENTS.md`. Same model as 3a — two renderings, never divergent.

### 3c. Optional — OpenAPI augmentation (companion, non-normative)
- If an `openapi.json`/`.yaml` exists: **augment, never overwrite.** Add per paid operation:
  ```json
  "x-402": { "scheme": "exact", "network": "eip155:84532", "asset": "USDC",
             "price": "$0.001",  // or the §2 dynamic model
             "note": "Non-normative. Authoritative price = live 402. Canonical discovery = x402 Bazaar." }
  ```
  Read `network`/`asset`/`scheme`/`price` from the route config (same source as §2) — don't hardcode
  them, or the annotation can drift from the code. Preserve existing content/order/formatting.
- If none exists: **offer** a minimal OpenAPI (paths, methods, the bazaar `input`/`output` as
  request/response schemas, + `x-402`). Don't force it. OpenAPI describes shape, not payment.

## 4. Always state the truth

Every artifact carries: *origin-local convenience; the x402 Bazaar (`/discovery/resources` on the
facilitator) is the canonical, cross-provider index.* And: `.well-known/x402` is a **convention** this
provider adopts (mirroring the Bazaar shape), **not** a ratified x402 standard.
