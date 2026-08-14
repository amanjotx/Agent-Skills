---
name: ingest
description: >-
  Deeply explore a codebase — intent, problem, current standing, and wiring —
  then persist one distilled Hydra standing document with a git sync waypoint.
  Use when the user says /ingest, ingest, map this repo into hydra, or asks for
  a full project memory bootstrap.
disable-model-invocation: true
---

# /ingest

Build a high-fidelity mental model of the **current** repository **and the
product it is trying to become**, then persist **one** standing document so a
new agent can orient with a single inspect — without skimming folder names.

This is the **full bootstrap** path (depth over speed). Prefer `/hydrate` for
routine sync after `{project}-standing` already exists.

Every successful ingest MUST record a **Sync waypoint** (commit hash + metadata)
at the top of standing so later `/hydrate` runs can diff from a trusted benchmark.

## Hard rules

- **Code is implementation truth.** Product markdown is for _intent_. If they
  conflict, standing must say so (code wins for “what exists”; dated product
  locks win for “what we are building next” only when still explicitly locked).
- **Do not dump** `CONTEXT.md`, `PRODUCT.md`, `HANDOFF.md`, or long spec files
  into Hydra. You **may and should read them** (and similar `docs/*` locks) to
  distill problem, audience, and destination into short standing sections.
- Never store secrets, `.env` values, tokens, private keys, or credentials.
- Not a file dump, chat transcript, session checkpoint, or raw tree listing.
- Keep the skill project-agnostic. Infer names, stacks, and commands from the repo.
- **Exactly one project `source_id`:** `{project}-standing`. Never create
  `{project}-codebase-map`, `{project}-product-locked`, `{project}-decisions-*`,
  or `{project}-scars` as sibling memories.
- Upsert that id. Delete leftover sibling ids after the write.
- Ingest with **`infer: false`**. `infer: true` is only for messy chat/signal.
- Stay **≤990 words** (Hydra `memory_tokens` per-request cap is 1000; cost ≈
  whitespace-separated words). Fill that budget. Compress; do not split ids.
  Too short to explain _why the product exists_ and _what is not built yet_ is
  a failed ingest. Count words before upsert.
- Personal prefs/skills belong in collection `personal`, not in project standing.
- Never invent a git sha.

## Identity

Derive `{project}` as a lowercase hyphenated slug. Announce once
(e.g. `Using standing id: shepherd-standing`).

This repo’s MCP must pin `HYDRADB_COLLECTION` to that slug.

## Quality bar (fail ingest if unmet)

A new agent who only inspects standing must be able to answer:

1. **Problem** — what operator/user failure this repo exists to fix
2. **Product** — one-liner, who it’s for, what it is _not_
3. **Now vs next** — what code actually does today vs locked destination slice
4. **Layout + wiring** — packages, entrypoints, request/data/UI flows
5. **Contracts** — invariants that break the product if ignored + verify commands
6. **Decisions + scars** — why, wire path, what not to reintroduce
7. **Waypoint** — git sha for `/hydrate`

If standing only lists folders and “auth exists,” it is a skim — go back to
exploration before writing.

## Workflow

### 1. Identify the project

Display name, slug, monorepo vs single package, languages from manifests.
Prefer **disk folder names** over README aliases.

### 2. Collect sync waypoint (git)

- `git rev-parse HEAD` / `--short=12` / `--abbrev-ref HEAD`
- `git log -1 --format=%s` and `%cI`
- `git status --porcelain` (no secrets in memory)

### 3. Check existing Hydra state

`hydradb_inspect` `{project}-standing` only. `hydradb_list` to find siblings to
delete later. Do not query-scatter for orientation.

### 4. Deep dive (required — not a catalog pass)

Use Glob/Grep/Read and explore subagents. **Read real entrypoints and call
chains**, not just package.json descriptions.

**Intent (distill, don’t paste)**

- Product/vision docs if present (`PRODUCT.md`, `HANDOFF.md`, locked `docs/*`)
- README one-liner vs code reality
- Audience, in/out of scope, differentiation — 8–15 lines max in standing

**Implementation**

1. Root & tooling — manifests, CI, verify scripts, pre-commit
2. Every app/service/package/crate — role, stack, **maturity (real/stub)**
3. Domain surfaces — routes, Tauri commands, UI gates, workers, queues, schema
4. Wiring — who calls whom; auth cookies vs Bearer; local ports
5. Persistence & externals — DB, KV, queues, third-party APIs (**names only**)
6. Auth / tenancy / security if present
7. Tests that exist vs missing
8. Gaps — stubs, TODOs that change architecture

Stop when you can explain a request’s path across at least one **real** vertical
(e.g. desktop login → API → DB) and name every **fake** surface an agent might
mistake for shipped.

### 5. Synthesize

Internal model before writing:

- Problem + thesis
- Locked destination (clearly labeled **not shipped** if code disagrees)
- Current standing
- Layout + wiring + auth/API surface table if auth-heavy
- Contracts, decisions, scars, gaps
- Pointers to long specs (“read X when implementing Y”) — not the spec body

### 6. Persist

`hydradb_ingest`: `source_id` `{project}-standing`, `infer: false`,
`is_markdown: true`, title `{Project} — standing`.

```markdown
# {Project} — standing

## Sync waypoint

…

## Problem

… operator/user failure; why this repo exists

## Product

… one-liner, audience, not-for, principles still true

## Destination (locked, not necessarily shipped)

… next slice / architecture locks. Label clearly vs code.

## Layout + wiring

… packages, entrypoints, flows. Disk paths not README aliases.

## Current standing

… shipped vs stub

## Contracts

… invariants + verify commands

## Decisions

… dated Decision / Wire / Effect

## Scars

…

## Gaps

…

## Spec pointers

… path → when to open (implementers only)
```

Then delete old sibling ids.

### 7. Report back

Slug, standing id, waypoint, 8–12 line orientation (problem + now + next),
gaps, docs distilled vs skipped. Do not dump the full markdown unless asked.
