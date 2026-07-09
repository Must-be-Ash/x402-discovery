---
name: x402-discovery
description: >-
  Make an API's existing x402 (v2) endpoints discoverable at its own domain. Use AFTER x402 is already
  set up (e.g. by the x402-start skill) when the user wants agents that land on their domain to
  enumerate paid endpoints, how to call them, and what they cost — without querying the central Bazaar.
  Emits an origin-local /discovery/resources (mirroring the Bazaar shape), a plain llms.txt endpoint
  list, and optional non-normative x-402 annotations on an existing OpenAPI. Projects the existing x402
  route config; never creates or changes the payment setup.
---

# x402-discovery — origin-local discovery for your x402 endpoints

Make the paid endpoints an x402-configured API already exposes **discoverable at the provider's own
domain**, so an agent that reaches `theirsite.com` can list the paid endpoints, learn how to call them,
and see what they cost — without going through the central Bazaar.

You **project an existing x402 setup** into discovery artifacts. You do not create or change payments —
that's the `x402-start` skill's job. Read the `reference/` guidance in this order:
`reference/knowledge.md` (projection rules), `reference/bazaar-shape.md` (exact shape to mirror),
`reference/pitfalls.md` (before finalizing).

## Non-negotiable rules

1. **Don't reinvent discovery.** x402's canonical discovery is the **Bazaar**; this is an *origin-local
   convenience* that mirrors the same shape. Every artifact must say so.
2. **Derive from the route config — one source of truth.** Read the existing `routes` map; never author
   a second, hand-maintained list that can drift.
3. **Mirror the Bazaar shape exactly** (`reference/bazaar-shape.md`) — never invent fields, never claim
   `.well-known/x402` is a ratified standard (it's a convention).
4. **Never fabricate a price** for a dynamic-price route — emit a pricing model; the live 402 is
   authoritative. `upto`/`batch` amounts are caps.
5. **Discovery is free.** Never put a discovery endpoint behind payment.
6. **Never touch payment logic**, re-register with the Bazaar, or expose secrets. `payTo` is public; keys never.
7. **v2 only** (`x402Version: 2`).

## Workflow

### 1. Precondition — find the existing x402 config
Confirm the repo is x402-configured (`@x402/*` deps + a `routes`/`RoutesConfig` map in
`payments.ts`/`proxy.ts`). If not, **stop** and tell the dev to run `x402-start` first — this skill
projects an existing setup; it doesn't create one.

### 2. Build the projection model
Parse the route map once (`reference/knowledge.md` §1): per route capture method/path/description,
each `accepts[]` (scheme/network/payTo/price), and the already-declared bazaar `input`/`output`.
Classify paid / free-SIWX / unpaid (exclude unpaid like `/health`). Detect static vs dynamic price.

### 3. Emit the artifacts (all from the one model)
- **Primary — `/discovery/resources`** (Bazaar-shaped): prefer a **runtime, free** route (alias
  `/.well-known/x402.json`) that reflects over the live `routes` object so it can't drift. For
  static-price routes, build `accepts[]` via the SDK's payment-requirements builder for an exact match;
  dynamic routes get a pricing model. Static-file option only if the team prefers it.
- **Companion — `llms.txt` / `AGENTS.md`**: plain list — method, path, purpose, how to call, cost.
- **Optional — OpenAPI**: if one exists, **augment** with a non-normative `x-402` block (never
  overwrite); if none, *offer* a minimal one.

### 4. Validate the projection
Cross-check every emitted resource against the live route config / a 402 probe: same paths, methods,
schemes, networks, `payTo`, and (for static prices) amounts. Confirm the discovery endpoints return
valid JSON in the Bazaar shape and are reachable **without payment**. Confirm `/health` (and other
non-product routes) are not advertised, dynamic routes show models (no fabricated numbers), and
`upto`/`batch` show caps.

### 5. Hand off
Summarize what's now discoverable and where (the endpoint URLs + files). Restate: this is an
origin-local convenience; the Bazaar is the canonical cross-provider index. Leave the git commit to the dev.

## How to use this folder
- `reference/knowledge.md` — the projection rules (route config → artifacts; static vs dynamic price).
- `reference/bazaar-shape.md` — the exact `/discovery/resources` shape to mirror.
- `reference/pitfalls.md` — check before finalizing (drift, dynamic-price honesty, don't-overwrite-openapi).
