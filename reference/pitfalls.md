# Pitfalls — origin-local discovery

## 1. Drift — a stale static file that disagrees with the code
The whole value is "discovery reflects the real endpoints + prices." A hand-written or one-off static
file goes stale the moment a route/price changes. **Fix:** prefer a **runtime** `/discovery/resources`
endpoint that reflects over the live `routes` object (can't drift). If you must ship a static
`.well-known/x402.json`, generate it from the route config and tell the dev to regenerate on change.

## 2. Fabricating a number for a dynamic price
Several routes price via callbacks (no fixed number without a concrete request). **Never invent one.**
Emit a pricing *model* (`kind: "dynamic"`, `model`, `example`) and point to the live 402 as
authoritative. For `upto`/`batch-settlement`, mark the amount as a **cap/max**, not a fixed charge.

## 3. `amount` vs human `price`
The Bazaar shape uses `amount` (integer, asset minor units — USDC has 6 decimals, so `"200"` = $0.0002).
The route config carries a human `price` string (`"$0.001"`). **Don't hand-convert and risk an error** —
reuse the x402 SDK's payment-requirements builder (the same one the middleware uses) so `amount`/`asset`/
`extra` match a real 402 exactly. Only static prices resolve to `amount`; dynamic ones stay models (#2).

## 4. Overwriting an existing OpenAPI
Most real APIs already maintain `openapi.json` by hand or via tooling. **Augment** it with a non-normative
`x-402` block per paid operation; **never** overwrite, reorder, or regenerate it from scratch. Only offer
a fresh minimal spec when none exists — and only if asked.

## 5. Charging for discovery
Discovery endpoints (`/discovery/resources`, `/.well-known/x402.json`, `llms.txt`) must be **free and
unpaid**. Never put them behind x402, and never in the paid route map. Agents must read them without paying.

## 6. Inventing fields / overclaiming a standard
Mirror the exact Bazaar shape (`bazaar-shape.md`); if unsure of a field, read the bundled bazaar docs or
the installed `@x402/extensions/bazaar` types rather than guessing. And label `.well-known/x402` as a
**convention**, not a ratified standard — the Bazaar remains canonical. Overclaiming undermines trust.

## 7. Re-registering with the Bazaar / re-doing x402
This sub-skill only **projects locally**. It does not register with the Bazaar (x402-start already did),
and it never adds/edits payment schemes or prices. If there's no x402 config, it stops and defers to
x402-start — it does not create one.

## 8. Leaking secrets
`payTo` addresses are public and fine to expose. Keys are not — never read or emit them. Read `payTo`
from the route config (which may itself read env); don't fabricate addresses.
