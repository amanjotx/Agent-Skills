---
name: simply
description: >-
  Re-explain the current discussion, plan, or design in plain language with
  short analogies and simple diagrams. Use when the user says /simply, simply,
  eli5, plain English, simplify, or asks to restate the plan so it is easy to
  follow.
disable-model-invocation: true
---

# /simply

Restate the **ongoing discussion or proposed plan** so it is easy to understand
quickly. Do not implement, expand scope, or re-litigate decisions unless asked.

## Hard rules

- No new recommendations unless required to clear a real confusion.
- Avoid code dumps and long path lists; one concrete example is enough when useful.
- Prefer short sentences and everyday words.
- If a technical term is unavoidable, define it in one line on first use.
- Lead with the big picture in ≤3 sentences.

## Output format

### 1. In one breath

2–3 sentences: what we are doing and why.

### 2. The plan like a story

Numbered steps a smart non-expert can follow:

1. …
2. …
3. …

### 3. Picture it

One of:

- a small ASCII diagram, or
- a short Mermaid flowchart (about 5–12 lines)

### 4. Words that sound scary

| Term | Plain meaning |
| ---- | ------------- |
| …    | …             |

Only include terms that appeared in the discussion.

### 5. What “done” looks like

3–5 bullets of observable outcomes (not implementation chores).

### 6. Open questions (only if real)

Bullets for unresolved choices that still need a human decision. Omit if none.

## Tone

Direct, clear, lightly conversational. No fluff, no praise filler, no prompt restating.
