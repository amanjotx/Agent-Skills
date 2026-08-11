---
name: ship
description: >-
  Prepare a clean branch from latest main, commit with conventional prefixes,
  push, and open a GitHub PR with a detailed body including diagrams. Use when
  the user says /ship, ship, commit and PR, or asks to push and raise a pull
  request for the current work.
disable-model-invocation: true
---

# /ship

Commit current work, push, and open a pull request. Invoking this skill is
explicit permission to create/switch branches, commit, push, and run `gh pr create`.

## Hard rules

- Follow the user's git safety protocol: never change git config; never skip
  hooks; never force-push to `main`/`master`; amend only when their amend
  rules are fully satisfied.
- Never commit secrets (`.env`, credentials, private keys, tokens). Warn and
  exclude them if present.
- Stage only files that belong to this change. Do not use blanket
  `git add .` / `git add -A` unless the tree is clearly intentional and clean
  of junk.
- Use HEREDOC for commit messages and PR bodies.
- Keep this workflow project-agnostic. Infer verify commands and layout from
  the repo; do not hardcode app or package names.
- Return the PR URL when finished.

## Conventional commit prefixes

Use production-grade Conventional Commits. Pick **one** primary type:

| Prefix     | When                                                  |
| ---------- | ----------------------------------------------------- |
| `feat`     | New user-facing capability                            |
| `fix`      | Bug fix                                               |
| `refactor` | Internal restructure with no intended behavior change |
| `perf`     | Performance improvement                               |
| `chore`    | Maintenance, tooling, deps, housekeeping              |
| `docs`     | Documentation only                                    |
| `test`     | Tests only                                            |
| `ci`       | CI configuration / pipelines                          |
| `build`    | Build system or bundler changes                       |
| `style`    | Formatting / lint-driven churn with no logic change   |
| `revert`   | Revert a previous commit                              |

Format:

```text
type(optional-scope): imperative summary

Optional body: why this change exists (1–2 sentences).
```

Examples: `feat(auth): add refresh token rotation`, `fix: prevent double-submit on checkout`, `chore: bump lint config`.

## Workflow

### 1. Inspect (run in parallel)

- `git status`
- `git diff` and `git diff --staged`
- `git branch -vv` (current branch + tracking)
- `git log -8 --oneline` (message style)
- Default base branch (`main` preferred, else `master` or repo default)
- Remote sync state (`git status -sb`, unpushed commits)

If there is nothing to commit and no unpushed commits relevant to this work,
stop. Do not open an empty PR.

### 2. Ensure the correct branch

Goal: land work on a **dedicated feature branch** based on **latest base**.

1. Determine whether the **current branch is meant for these changes**:
   - Yes if it is already a descriptive topic branch for this work
     (`feat/…`, `fix/…`, `chore/…`, or clearly matching the change intent)
     and it is **not** the default branch.
   - No if on `main`/`master`/default, a mismatched topic branch, a stale
     unrelated branch, or a detached HEAD.

2. If the current branch is **not** meant for these changes:
   - If the working tree is dirty, **stash** with a clear message
     (default; do not ask unless stash is impossible).
   - `git checkout <base>`
   - `git pull --ff-only` (or equivalent fast-forward update from tracking remote)
   - Create and check out a new descriptive branch, e.g.
     `feat/short-topic`, `fix/short-topic`, `chore/short-topic`
   - `git stash pop` onto the new branch; resolve conflicts if any.

3. If the current branch **is** meant for these changes:
   - Stay on it.
   - Before committing, update from base when needed
     (rebase or merge from latest base per repo norms; prefer non-destructive
     options; do not force-push unless the user explicitly asks).

Never build a PR directly from the default branch.

### 3. Commit

- Prefer one coherent commit unless the user asked for a split.
- Message uses a proper conventional prefix and an imperative summary.
- Body (optional): focus on **why**, not a file list.
- After commit, run `git status` to confirm the expected state.
- If a hook fails: fix the issue and create a **new** commit. Do not amend
  unless amend rules are fully met.

### 4. Push

```bash
git push -u origin HEAD
```

Request required permissions when sandbox/network/`gh` blocks the operation.

### 5. Open the PR

Analyze **all** commits that will merge into base (not only the latest).

**Title:** same spirit as the commit summary — clear, specific, conventional
when it helps. No vague titles.

**Body template (required):**

```markdown
## Summary

- 2–5 bullets on what ships and why it matters

## Why

- Problem, motivation, or trigger for this change

## What changed

- Bullets grouped by concern (behavior, API/contracts, data, UI, infra, tests, docs)
- Explicitly call out breaking changes, migrations, or auth/session behavior shifts

## Architecture / flow

Include at least one:

1. Mermaid diagram (sequence, flowchart, or state) for the primary path
2. ASCII diagram for structure, layering, or before → after

Use Mermaid for control flow; ASCII for trees or layer maps.
Diagrams must match the actual diff — do not invent architecture.

## Risk / rollback

- Likely failure modes
- Safe revert / mitigation notes

## Test plan

- [ ] Concrete steps a reviewer can run
- [ ] Repo-appropriate verification commands discovered from the project
```

Create with `gh pr create` and a HEREDOC body. Set `--base` explicitly when needed.

### 6. Report back

Keep it short:

- branch name
- commit summary
- PR title
- PR URL
- any intentionally uncommitted files
