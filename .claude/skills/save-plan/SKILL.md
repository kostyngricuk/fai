---
name: save-plan
description: "Save a plan produced by a FAI feature agent to .fai/plans/ with the standard location, naming, and structure so it can be reviewed, resumed, and handed off across sessions. Use whenever a fai-* agent finishes a file-level implementation plan for a ticket, or when asked to persist/record a plan for later work."
argument-hint: <ticket id + the plan to save>
---

Persist a plan as a versioned markdown file so it survives the session — reviewable, resumable, and
handoff-ready.

## When to use

- A feature agent (or `/fai:plan`) has just produced a concrete, file-level plan.
- You are asked to save, record, or persist a plan for future work.

Do **not** use this to invent a plan — only to store one that already exists. If no plan exists yet,
produce it first (`/fai:plan <ticket>`), then run this skill.

## Where plans live

`.fai/plans/`. Create it yourself before writing — never ask the user to make it:

```sh
mkdir -p .fai/plans
```

## Naming

`feat-<ticket-number>.md`, using only the numeric part of the ticket.

- Linear `TID-3212` → `.fai/plans/feat-3212.md`
- No ticket → a short kebab-case slug: `feat-<slug>.md`
- One plan per file. If a plan for the ticket already exists, **update it in place** rather than
  creating a duplicate; note what changed at the top of the affected section.

## Required structure

Mirror `example-plan.md` in this skill's folder. Every plan MUST have this frontmatter:

```yaml
---
ticket: TID-3212
feature-agents: [fai-checkout, fai-payment]
status: draft          # draft | approved | in-progress | done
created: 2026-09-14
updated: 2026-09-14
---
```

…followed by these sections, in this order:

1. **Title** — `# <TICKET> — <one-line scope>`, stating the scope boundary explicitly (e.g.
   "product-listing part only").
2. **`## Context`** — what and why; the backend/API contract it depends on; the current behavior
   being changed; and an explicit **Scope boundary** (what is and is not included, what is deferred,
   what is owned elsewhere).
3. **`## Current-state facts (verified)`** — bullets citing the exact existing code the plan builds
   on, with real paths and line ranges (e.g. `src/cart/promo.ts` — `applyPromo` (L44-91) …). State
   which files, symbols, and tests exist and which **do NOT**.
4. **`## Implementation steps`** — numbered `### N. <step>` sections. Each names the exact file(s)
   to touch, shows the concrete code or diff, tags the owning feature agent, and gives a short
   rationale for non-obvious choices. Reference real symbols and follow patterns already in the
   codebase, not generic advice.
5. **`## Verification`** — the project's own commands, as recorded in the feature agent's
   Verification section (§9 of `.fai/features/<slug>.md`) — never a guessed `yarn test`. Give expected
   results and any known pre-existing failures to ignore. Note if manual/E2E is not exercisable and
   why.
6. **`## Out of scope / dependencies`** — deferred items, work owned by others (with ticket ids),
   external confirmations still required.
7. **`## Execution log`** — an append-only checklist of the implementation steps, so a later session
   can resume exactly where this one stopped:

   ```markdown
   - [x] 1. Add `isStackable` to the promo payload — done 2026-09-14, `a1b2c3d`
   - [ ] 2. Restore the stacking short-circuit
   - [ ] 3. Extend promo tests
   ```

## Grounding rules

- Plans are file-level and evidence-based. Every claim about existing code cites the real path (and
  line range when useful).
- Only mark facts "verified" if they were actually checked against the codebase.
- Keep the plan self-contained: a fresh reader with no conversation context must be able to execute
  it.

## Procedure

1. Determine the ticket number → resolve the target filename.
2. Check `.fai/plans/` for an existing file: create new, or update in place.
3. Write the plan following the frontmatter, required structure, and grounding rules.
4. Confirm the saved path back to the user (e.g. `.fai/plans/feat-3212.md`).

## Resuming a saved plan

In a later session, to pick a plan back up:

1. Read `.fai/plans/feat-<n>.md` and find the first unchecked box in the **Execution log**.
2. **Re-verify the "Current-state facts" first** — code moves. Anything that no longer holds
   invalidates the steps built on it; correct the plan before executing it, and consider
   `/fai:train <slug>` if the drift is in the feature agent itself.
3. Execute from that step, ticking boxes and recording commit SHAs as you go.
4. Bump `status` and `updated` in the frontmatter.

`/fai:plan` is the usual producer of plans saved here.
