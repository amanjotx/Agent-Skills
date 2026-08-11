---
name: hydrate
description: >-
  Sync Hydra DB with the current codebase standing using a git sync waypoint:
  explore code (delta-first when possible), compare to stored project memory,
  then write missing or corrected details. Use when the user says /hydrate,
  hydrate, sync hydra, refresh project memory, or asks to keep Hydra updated
  with the repo.
disable-model-invocation: true
---

# /hydrate

Keep Hydra DB aligned with a fast-moving codebase. This is an **incremental
sync**, not a first-principles rewrite.

Default flow:

1. Explore codebase (delta-first via Sync waypoint when valid)
2. Fetch standing from Hydra
3. Store missing details and correct stale information
4. Advance the Sync waypoint to current HEAD

If no meaningful `{project}-codebase-map` exists, say so and recommend `/ingest`
(or offer to run ingest if the user confirms).

## Hard rules

- Code wins when Hydra and code disagree. Fix Hydra in the same run.
- Do not dump large markdown docs into context or Hydra as raw paste.
- Never store secrets, env values, tokens, or credentials.
- Prefer updating stable sources over creating duplicate source ids.
- Keep writes distilled and high-signal. Skip transient WIP noise.
- Stay project-agnostic; infer `{project}` and verify commands from the repo.
- Do not rewrite unrelated personal prefs sources.
- Never invent a git sha. Always refresh the waypoint after a successful sync
  when git is available.

## Source id conventions

| `source_id`                  | Role                                                 |
| ---------------------------- | ---------------------------------------------------- |
| `{project}-codebase-map`     | Primary map + Sync waypoint (required update target) |
| `{project}-product-locked`   | Update only when locks changed and still true        |
| `{project}-decisions-<area>` | Add/update when durable decisions appear             |
| `{project}-scars`            | Add reusable gotchas when discovered                 |

## What “in sync” means

Hydra should reflect:

- Current layout and entrypoints
- Current shipping surface vs stubs
- Current verify commands
- Still-true locked decisions / overturned decisions cleaned up
- Important new wiring agents would otherwise rediscover
- An up-to-date Sync waypoint at current HEAD

## Workflow

### 1. Identify project + current git tip

- Derive display name + `{project}` slug
- Collect current tip metadata (same fields as ingest waypoint):
  - `git rev-parse HEAD`
  - `git rev-parse --short=12 HEAD`
  - `git rev-parse --abbrev-ref HEAD`
  - `git log -1 --format=%s`
  - `git log -1 --format=%cI`
  - `git status --porcelain` (dirty worktree)

### 2. Fetch Hydra standing + waypoint

- `hydradb_query` narrowly for this project’s map/standing
  (`max_results` ~3–5; `thinking` when reconciling)
- `hydradb_inspect` on `{project}-codebase-map`
- Parse `## Sync waypoint` → previous `commit` sha (and branch/subject if present)

If the map is missing/empty: stop incremental sync and recommend `/ingest`.

### 3. Decide exploration mode

Validate the previous waypoint sha:

1. Exists in this repo
2. Is an ancestor of current `HEAD`

Then choose:

| Condition                                                             | Mode                                                                     |
| --------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Valid waypoint ancestor of HEAD                                       | **Delta-first**                                                          |
| Missing / invalid / not ancestor / history rewrite                    | **Broader standing pass**                                                |
| Valid but very large delta (broad structural churn across many areas) | **Broader standing pass**, or recommend `/ingest` if map quality is poor |
| HEAD matches waypoint and worktree is clean                           | Still do a light confirm pass; report “already at waypoint” if no drift  |
| HEAD matches waypoint but worktree dirty                              | Delta includes dirty files                                               |

### 4. Explore

#### Delta-first (preferred)

- `git log --oneline <waypoint-sha>..HEAD`
- `git diff --stat <waypoint-sha>..HEAD`
- `git diff <waypoint-sha>..HEAD` for material paths
- Include dirty worktree paths from `git status`
- Deep-read changed paths **and** their immediate dependents/callers
- Re-check verify scripts / manifests if those files changed

Delta is a prioritization aid, not a blinders rule: if changed files imply
architecture/contract shifts, follow the connection far enough to keep the map honest.

#### Broader standing pass (fallback)

JIT-read current standing:

- Root manifests / workspace config
- Central entrypoints
- Active product surface modules
- CI / package scripts for verify commands
- Obvious added/removed apps, packages, or crates

### 5. Diff: code vs Hydra

Classify each claim / gap:

| Status              | Action                                                              |
| ------------------- | ------------------------------------------------------------------- |
| Accurate            | Keep                                                                |
| Missing             | Add to the appropriate source                                       |
| Stale / wrong       | Correct via ingest update; delete superseded memories if overturned |
| Provisional / noisy | Do not store                                                        |
| Secret / tactical   | Do not store                                                        |

Focus on material drift: new modules, stubs that became real, contract changes,
verify command changes, wiring changes, removed surfaces.

### 6. Write updates + advance waypoint

Use `hydradb_ingest` (`infer: true`, `is_markdown: true`) with stable `source_id`s.

- Prefer refreshing `{project}-codebase-map` as a coherent updated snapshot when
  drift is broad; keep sections consistent (no contradictory fragments).
- Always rewrite `## Sync waypoint` to the **current HEAD** metadata from step 1
  after a successful sync (even if content changes were small).
- Update `{project}-product-locked` only when locks changed and remain true.
- Add `{project}-decisions-<area>` when a durable decision is newly evident.
- Delete clearly superseded memories/sources.

Waypoint block to write:

```markdown
## Sync waypoint

- commit: <full sha>
- short: <12-char sha>
- branch: <branch name>
- subject: <one-line subject>
- committed_at: <ISO from git>
- captured_at: <ISO timestamp of this skill run>
```

### 7. Report back

Short summary only:

- Mode used: delta-first vs broader (and why)
- Previous waypoint → new waypoint (`short` shas + subject)
- Commit range covered when delta-first (`old..HEAD`, count if useful)
- Sources updated (and any deletes)
- Key corrections (Hydra was wrong → code truth)
- New facts added
- Intentionally skipped items
- Whether a deeper `/ingest` is recommended
