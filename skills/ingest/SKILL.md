---
name: ingest
description: >-
  Deeply explore a codebase, map architecture and connections, and store a
  durable project snapshot in Hydra DB with a git sync waypoint. Use when the
  user says /ingest, ingest, map this repo into hydra, or asks for a full
  project memory bootstrap.
disable-model-invocation: true
---

# /ingest

Build a high-fidelity mental model of the **current** repository, then persist
a distilled project snapshot into Hydra DB so future agents can orient quickly.

This is the **full bootstrap** path (depth over speed). Prefer `/hydrate` for
routine sync after a project map already exists.

Every successful ingest MUST record a **Sync waypoint** (commit hash + metadata)
on the codebase map so later `/hydrate` runs can diff from a trusted benchmark.

## Hard rules

- Code is the source of truth. Do not treat markdown docs as primary.
- Do **not** open or dump large orientation docs by default
  (`CONTEXT.md`, `PRODUCT.md`, `HANDOFF.md`, etc.). Use code + narrow Hydra reads.
- Never store secrets, `.env` values, tokens, private keys, or credentials.
- Write distilled maps — not file dumps, chat transcripts, or raw tree listings.
- Keep the skill project-agnostic. Infer names, stacks, and commands from the repo.
- Prefer a few stable `source_id`s over many noisy one-off writes.
- When Hydra already has project sources, inspect them, then **replace/supersede**
  with a better map rather than appending contradictory duplicates.
- Never invent a git sha. If git metadata is unavailable, omit the waypoint and
  report that future hydrate efficiency will be limited.

## Source id conventions

Derive `{project}` as a lowercase hyphenated slug from the repo/product name.

| `source_id`                  | Role                                           |
| ---------------------------- | ---------------------------------------------- |
| `{project}-codebase-map`     | Primary distilled architecture + sync waypoint |
| `{project}-product-locked`   | Locked product decisions still true in code    |
| `{project}-decisions-<area>` | Area ADRs when warranted                       |
| `{project}-scars`            | Durable gotchas worth preserving               |

Write with `hydradb_ingest`: markdown, `infer: true`, `is_markdown: true`,
stable `source_id`, clear `title`.

## Goals

After `/ingest`, Hydra should answer:

1. What the product/project is
2. Repo layout and major modules
3. How major pieces connect (request/data/UI/agent flows)
4. What is shipped vs stubbed
5. Key contracts, invariants, and verify commands
6. Important locked decisions still reflected in code
7. The git waypoint to use for efficient later hydrate

## Workflow

### 1. Identify the project

Derive:

- Display name (human title)
- `{project}` slug for `source_id`s
- Repo shape: monorepo vs single package
- Primary languages/runtimes from manifests

Announce the slug once before writing (e.g. `Using source prefix: shepherd-`).

### 2. Collect sync waypoint (git)

Capture from the current checkout:

- full sha: `git rev-parse HEAD`
- short sha: `git rev-parse --short=12 HEAD`
- branch: `git rev-parse --abbrev-ref HEAD`
- subject: `git log -1 --format=%s`
- committed_at: `git log -1 --format=%cI`

Also note dirty worktree via `git status --porcelain` (do not put secrets in memory;
mention only that uncommitted changes existed, if relevant).

### 3. Check existing Hydra state (narrow)

- `hydradb_query` for `{project} codebase map architecture current standing`
  (`max_results` ~3–5, mode `thinking` if reconciling)
- `hydradb_list` / `hydradb_inspect` for:
  `{project}-codebase-map`, `{project}-product-locked`

Use this only to avoid blind duplication and to find stale entries to supersede.

### 4. Deep codebase exploration

Explore systematically. Use Glob/Grep/Read (and explore subagents when useful).
Cover each layer that exists:

1. **Root & tooling** — workspace manifests, package manager, CI, verify scripts
2. **Apps / services / packages / crates** — role, stack, entrypoints, maturity
3. **Domain surfaces** — routes, commands, UI shells, workers, queues, schemas
4. **Wiring** — how requests/events move across boundaries; DI; shared contracts
5. **Persistence & external systems** — DBs, KV, queues, third-party APIs (names only)
6. **Auth / tenancy / security boundaries** if present
7. **Gaps** — stubs, TODOs that affect architecture, missing clients/tests/docs

Read enough real entrypoints and call chains to explain connections confidently.
Do not pretend complete coverage from folder names alone.

### 5. Synthesize the map (before writing)

Produce an internal model with:

- Shipping surface vs stubs
- Connection narrative (who calls whom)
- Invariants / contracts that agents must not break
- Verify commands discovered from the repo
- Open gaps worth remembering
- Sync waypoint fields from step 2

Optional: one Mermaid or ASCII connection diagram in the stored map.

### 6. Persist to Hydra

Use `hydradb_ingest` with `infer: true`, `is_markdown: true`.

#### Required: `{project}-codebase-map`

Title example: `{Project} — codebase map (source-derived)`

Put **Sync waypoint first**, then the map body:

```markdown
# {Project} — codebase map (source-derived)

## Sync waypoint

- commit: <full sha>
- short: <12-char sha>
- branch: <branch name>
- subject: <one-line subject>
- committed_at: <ISO from git>
- captured_at: <ISO timestamp of this skill run>

## What it is

…

## Tooling / verify

…

## Layout

…

## How it connects

… (flows / boundaries / key call paths)

## Shipping surface vs stubs

…

## Key entrypoints

…

## Notable contracts / invariants

…

## Known gaps

…
```

#### Usually include: `{project}-product-locked`

Only decisions that are both product-important and still true in code.
If uncertain, omit or mark as provisional — do not invent locks.

#### Optional area sources

Create `{project}-decisions-<area>` only when an area has durable decisions
worth separate recall. Use:

```markdown
## Decision

…

## Why

…

## Wire

…

## Effect

…

## Alternatives considered

…
```

#### Scars

Add `{project}-scars` only for painful, reusable gotchas discovered during ingest.

If ingest replaces an older wrong map, delete superseded memories/sources when
clearly overturned (`hydradb_delete`).

### 7. Report back

Keep the chat summary short:

- `{project}` slug used
- Sources written/updated
- Sync waypoint (short sha + branch + subject)
- 5–10 line orientation blurb (what it is + shipping surface)
- Highest-priority gaps
- Anything uncertain / not captured

Do not dump the full stored markdown unless asked.
