---
name: hydrate
description: >-
  Sync Hydra DB standing with the current codebase using a git sync waypoint:
  explore code (delta-first when possible), compare to the one project memory,
  rewrite it. Use when the user says /hydrate, hydrate, sync hydra, refresh
  project memory, or asks to keep Hydra updated with the repo.
disable-model-invocation: true
---

# /hydrate

Keep Hydra DB aligned with a fast-moving codebase. This is an **incremental
rewrite of one document**, not a first-principles ingest and not a scatter of
new source ids.

Default flow:

1. Explore codebase (delta-first via Sync waypoint when valid)
2. `hydradb_inspect` `{project}-standing`
3. Patch sections in that document; upsert the same id
4. Advance the Sync waypoint to current HEAD

If `{project}-standing` is missing/empty, stop and recommend `/ingest`
(or offer to run ingest if the user confirms).

## Hard rules

- Code wins when Hydra and code disagree. Fix standing in the same run.
- Do not dump large markdown docs into context or Hydra as raw paste.
- Never store secrets, env values, tokens, or credentials.
- **One project id only:** `{project}-standing`. Never create sibling ids.
- Upsert with **`infer: false`**, `is_markdown: true`.
- Keep writes distilled. Skip transient WIP and session checkpoints.
- Do not rewrite personal-collection prefs while hydrating a project.
- Never invent a git sha. Always refresh the waypoint after a successful sync
  when git is available.
- Stay ≤990 words (Hydra `memory_tokens` cap is 1000/request). Preserve Problem / Product / Destination; refresh Current standing from code. Compress instead of splitting.

## What “in sync” means

Standing reflects current layout, wiring, shipped vs stub, verify commands,
still-true locks, dated decisions, scars, and a waypoint at current HEAD.

## Workflow

### 1. Identify project + current git tip

- `{project}` slug; standing id `{project}-standing`
- `git rev-parse HEAD` / `--short=12` / `--abbrev-ref HEAD`
- `git log -1 --format=%s` and `%cI`
- `git status --porcelain`

### 2. Fetch standing + waypoint

- `hydradb_inspect` `{project}-standing` only (not a 3–5 hit query)
- Parse `## Sync waypoint` → previous `commit` sha

If missing/empty: recommend `/ingest`.

### 3. Decide exploration mode

Validate the previous waypoint sha exists and is an ancestor of `HEAD`.

| Condition                                          | Mode                                                     |
| -------------------------------------------------- | -------------------------------------------------------- |
| Valid waypoint ancestor of HEAD                    | **Delta-first**                                          |
| Missing / invalid / not ancestor / history rewrite | **Broader standing pass**                                |
| Valid but very large structural delta              | Broader pass, or recommend `/ingest` if standing is poor |
| HEAD matches waypoint, worktree clean              | Light confirm; report “already at waypoint” if no drift  |
| HEAD matches waypoint, worktree dirty              | Delta includes dirty files                               |

### 4. Explore

**Delta-first:** `git log --oneline <waypoint>..HEAD`, `git diff --stat` and
material diffs, dirty paths, dependents/callers, verify scripts if those files
changed.

**Broader:** root manifests, central entrypoints, active product surfaces, CI
scripts, added/removed apps/packages/crates.

### 5. Diff: code vs standing

| Status                                  | Action                                            |
| --------------------------------------- | ------------------------------------------------- |
| Accurate                                | Keep in the rewritten doc                         |
| Missing                                 | Add to the right **section** of standing          |
| Stale / wrong                           | Correct in standing; do not leave a second memory |
| Provisional / noisy / secret / tactical | Do not store                                      |

Focus on material drift: new modules, stubs that became real, contract changes,
verify commands, wiring, removed surfaces, overturned locks.

### 6. Write + advance waypoint

Rebuild the full markdown (same section order as `/ingest`) and
`hydradb_ingest` `{project}-standing` with `infer: false`.

Always rewrite `## Sync waypoint` to current HEAD metadata from step 1.

If leftover sibling ids exist (`*-codebase-map`, `*-decisions-*`, `*-scars`,
`*-product-locked`), delete them.

Waypoint block:

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

Short: mode, previous → new waypoint, commit range if delta-first, sections
changed, corrections (Hydra was wrong → code), skipped items, whether `/ingest`
is recommended.
