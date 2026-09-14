---
name: fai-plan
description: "Turn a ticket or an issue description into a file-level implementation plan by routing it to the right FAI feature agents. Fetches the ticket (Linear/Jira/GitHub/Trello), picks the matching fai-* agents, runs them in sequential handoff so they build on each other like a team, and merges the result into one ordered plan. Use when asked to plan a feature, plan a fix, or work out how to implement a ticket."
argument-hint: <ticket URL/id, or a description of the issue or feature>
---

Produce **one** implementation plan for `$ARGUMENTS`, grounded in the features it actually touches.

The value here is routing: a generic planner re-derives the feature from scratch and misses the
rules that live in history. A feature agent already knows them. When a goal spans several features,
the agents work it like a team — each building on what the previous one decided.

## Non-negotiable rules

1. **Read-only on external systems.** Read tickets and PRs freely; never comment, re-status,
   review, merge, or edit.
2. **Evidence-based.** Facts about existing code cite `path:line`. Mark what does not exist yet.
3. **Graceful degradation.** Missing MCP, `gh`, or git is a note in the output, not a failure.
4. **One plan out.** Never hand the user N disconnected agent reports to reconcile themselves.

---

## Procedure

### Step 1 — Resolve the goal

Work the fetch cascade, stopping at the first that succeeds:

1. **Tracker MCP**, if one is actually connected (Linear, Jira, …) — check what is available, do
   not assume. Fetch the issue by id or URL.
2. **GitHub** — `gh issue view <n> --json title,body,labels,comments` (also works for a URL).
3. **`WebFetch`** the URL.
4. **Ask the user** to paste the ticket text.

If the argument is a free-text description rather than a link, use it directly.

Extract and restate: **goal**, acceptance criteria, constraints, explicit non-goals. If the goal is
genuinely ambiguous, ask now — before spending agent turns.

### Step 2 — Route to feature agents

```sh
ls fai/features/*.md 2>/dev/null
```

Read each file's frontmatter (`description`, `paths`). Score against the ticket text and against any
symbols or paths it names — grep the repo for named symbols to find whose code map owns them.

Produce an ordered list: **primary agent first** (owns the core of the change), then secondaries.

- **No feature agents exist** → tell the user, suggest `/fai-create <guess>`, and offer to proceed
  with the built-in `Plan` agent instead, clearly flagged as *not* feature-grounded.
- **No agent matches** → same, naming the agents you considered and why each missed.
- **Ambiguous, or 2+ agents match** → confirm the selection and the order with `AskUserQuestion`
  before running anything. Agent turns are expensive; a 10-second check is not.

### Step 3 — Sequential handoff

Run the agents **one at a time, never in parallel.** The handoff is the point: agent 2 must see what
agent 1 decided, or you get two plans that contradict each other at the seam.

Dispatch each as its `fai-<slug>` subagent. Give every agent:

- the resolved ticket text and the goal
- **the complete output of every prior agent**
- this boundary instruction:

  > Plan only within your own feature. Where you depend on another feature, state the **contract**
  > you need from it — the exact type, endpoint, event, or flag — and leave the implementation to
  > that feature's agent. Reconcile with the decisions already made by prior agents rather than
  > restating or relitigating them. If you disagree with a prior decision, say so explicitly and
  > explain the consequence; do not silently plan around it.

Ask each for a file-level plan following section 12 of its own knowledge file.

### Step 4 — Assemble one plan

Merge into a single document:

1. **Goal & scope boundary** — what is in, what is out, what is deferred.
2. **Current-state facts (verified)** — the union of all agents' cited facts, deduped. Keep the
   `path:line` citations. Keep the explicit "does NOT exist yet" statements.
3. **Implementation steps** — numbered and in **dependency order**, not agent order. A step that
   another step needs comes first, even if a different agent wrote it. Tag each step with the
   owning agent.
4. **Cross-feature contracts** — shared types, API shapes, event and flag names each side must
   honor. This is what keeps the seam from breaking.
5. **Conflicts & risks** — anything two agents disagreed about, surfaced rather than quietly
   resolved. Say which side you would take and why, and let the user decide.
6. **Verification** — the exact commands, merged from each agent's section 9.
7. **Out of scope / dependencies** — deferred items, work owned elsewhere with ticket ids, external
   confirmations still needed.

### Step 5 — Present, colorized

Tag each agent's contribution with its stable color dot and name so the user can see who said what —
the teamwork should be legible:

```markdown
### 3. Add `isStackable` to the promo payload  🟣 fai-checkout

`src/cart/promo.ts:44` — …
```

Use the shared severity vocabulary for risks: 🔴 CRITICAL · 🟠 MAJOR · 🟡 MINOR · 🔵 NOTE.

### Step 6 — Offer to save

Close by offering `/fai-save-plan` so the plan survives into a future session, and name the path it
would be written to (`fai/plans/feat-<ticket>.md`).
