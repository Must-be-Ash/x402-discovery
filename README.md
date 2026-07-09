# x402-discovery

A small Claude Code skill that makes your **already-x402-enabled** API discoverable at your own domain.

After you've set up x402 payments (e.g. with the [`x402-start`](https://github.com/Must-be-Ash/x402-start)
skill), run this to let agents that land on your domain see **what endpoints you offer, how to call
them, and what they cost** — without having to query the central Bazaar.

## What it does

- Reads your existing x402 route config (one source of truth) and projects it into:
  1. **`/discovery/resources`** — a free endpoint that mirrors the x402 **Bazaar** shape, listing your
     paid endpoints + prices + schemas (also served at `/.well-known/x402.json`).
  2. **`llms.txt` / `AGENTS.md`** — a plain list agents and humans can read: each endpoint, what it's
     for, how to call it, and its cost.
  3. **OpenAPI (optional)** — if you already have an `openapi.json`, it adds non-normative `x-402`
     pricing annotations (it never overwrites your spec).
- Because it's derived from your route config, **when your code changes, discovery changes** — the
  primary endpoint reflects the live config, so it can't drift.

It **does not** set up or change payments (that's `x402-start`), and it's honest that the **Bazaar
remains the canonical, cross-provider index** — this is an origin-local convenience.

## Install

```bash
git clone https://github.com/Must-be-Ash/x402-discovery.git ~/.claude/skills/x402-discovery
```

## How to use

In your x402-enabled project, ask Claude Code:

> **Make my x402 endpoints discoverable at my domain — add a /discovery/resources endpoint and an llms.txt.**

It reads your route config and generates the discovery artifacts. If the project isn't x402-enabled
yet, it'll tell you to run `x402-start` first.

## Requirements
- A project already configured for x402 (v2).
- Nothing else — no keys, no secrets. Discovery is free and public by design.
