---
name: fai-train
description: "Update FAI feature agents from new information — recent commits, a merged PR, updated documentation, a URL, or a direct correction. Reconciles every claim against the current code, proposes the change set for approval, applies it surgically, bumps the watermark, and regenerates the routing stub. Called with no arguments it trains every existing agent at once. Use when a feature agent is out of date, or after shipping work that changes how a feature behaves."
argument-hint: "[feature-slug] [source: commits | <sha>..<sha> | PR url/number | doc path | url | free text] — no args trains every agent"
---

Keep `.fai/features/<slug>.md` true. A feature agent that drifts from the code is worse than no
agent — it produces confident, cited, wrong plans.

This is not an append. It is a **diff of the document against reality**: every existing claim gets
re-checked, not just new facts added.

## Non-negotiable rules

1. **Propose before writing.** Always show the change set and get approval. Never silently rewrite
   a file the user may have hand-edited.
2. **Edit surgically.** Use targeted edits, not a full rewrite, so hand-written nuance survives.
3. **Read-only on external systems.** Read PRs, tickets, and docs; never write to them.
4. **Verify, do not assume.** A claim is only `confirmed` if you actually re-read the cited path.

---

## Procedure

### Step 1 — Load

**With a slug:** read `.fai/features/<slug>.md` and take `watermark` from its frontmatter. If the
file does not exist, say so and point at `/fai-create <slug>`.

**With no arguments: train every agent.** This is the routine maintenance mode — run it after a
merge, or on a Monday, to pull the whole roster back in sync at once.

```sh
ls .fai/features/*.md 2>/dev/null
```

- No agents at all → say so and point at `/fai-create`.
- Otherwise take every agent, read each watermark, and follow the **Bulk mode** rules below.

Do not ask which one to train. Asking is only correct when the user named a slug that does not
resolve — offer the close matches then.

### Step 2 — Gather the delta

By source type:

**Default, or `commits`** — everything since the watermark:

```sh
git log <watermark>..HEAD --oneline -- <paths from frontmatter>
git diff <watermark>..HEAD -- <paths from frontmatter>
git log <watermark>..HEAD --format='%s%n%b' -- <paths>   # new ticket ids, incl. trailers
```

Guard with `git rev-parse HEAD` first — `--git-dir` also succeeds in a repo with no commits.
No watermark, or no commit history → ask the user for the range, or fall back to another
source.

**A PR** — `gh pr view <n> --json title,body,files` and `gh pr diff <n>`. The PR body usually
carries the rationale the commits omit.

**A doc path** — read it. **A URL** — `WebFetch`.

**Free text** — treat the user's statement as authoritative; it outranks what the code appears to
say about intent, though not about mechanics.

Resolve any new ticket ids through the fetch cascade (tracker MCP → `gh` → `WebFetch`) to recover
the *why* behind the change.

### Step 3 — Reconcile every claim

Walk the knowledge file section by section. For each factual claim, re-read the cited path and
classify:

| Verdict | Meaning | Action |
| --- | --- | --- |
| `confirmed` | Still true | Leave it alone |
| `stale` | Path moved, symbol renamed, behavior changed | Rewrite with the new fact |
| `removed` | The thing no longer exists | Delete the claim **and add a "do not reintroduce" note** if it was deliberate |
| `new` | The delta introduces something the file does not cover | Add it to the right section |

Pay particular attention to:

- **Section 5 (Business rules & invariants)** — did the delta add, relax, or move a rule?
- **Section 7 (Built vs. open)** — which open items shipped? Move them to "Built". Which new gaps,
  TODOs, or known bugs appeared?
- **Deletions in the diff** — every deliberate removal is a candidate "do not reintroduce" line.
  These are the highest-value lines in the file.
- **Frontmatter `paths:` and `description`** — new files, routes, or ticket prefixes mean new
  routing triggers.

### Step 4 — Propose

Show a compact change set and get approval before touching anything:

```markdown
## Proposed updates to .fai/features/checkout.md

**Watermark:** `a1b2c3d` (2026-08-02) → `f9e8d7c` (2026-09-14) — 14 commits, 3 tickets

| # | Section | Change | Evidence |
|---|---------|--------|----------|
| 1 | §5 Business rules | 🟠 **stale** — stacking guard moved from `promo.ts:44` to `rules/stack.ts:18` | `f9e8d7c` |
| 2 | §7 Open items | ✅ ABC-312 shipped → move to Built | PR #481 |
| 3 | §7 Do not reintroduce | ➕ new — `LEGACY_PROMO_MAP` deleted, superseded by the dictionary | `c4d5e6f` |
| 4 | frontmatter | ➕ `src/cart/rules/**` added to `paths:`; description gains "stacking rules" | — |

Apply all, or tell me which to skip.
```

### Step 5 — Apply

On approval, edit surgically. Then, in order:

1. Update frontmatter `description` and `paths:` if triggers changed.
2. **Regenerate `.claude/agents/fai-<slug>.md`** from the skill template at
   `.claude/skills/fai-create/stub-template.md` if the description changed — the stub's description
   must stay verbatim-identical to the knowledge file's, or routing silently drifts.
3. Move shipped items from "still open" to "built".
4. Add "do not reintroduce" entries for deliberate deletions.
5. Bump `watermark` to the new short SHA and date.
6. Append a Learning-log row:

   ```markdown
   | 2026-09-14 | commits a1b2c3d..f9e8d7c, PR #481 | Stacking rules moved to rules/stack.ts; ABC-312 shipped; LEGACY_PROMO_MAP retired. |
   ```

### Step 6 — Report

Print: sections updated, stale claims corrected, claims removed, new claims added, the new
watermark, whether the stub was regenerated, and anything you could not verify.

---

## Bulk mode (no arguments)

Steps 2–6 run per agent, with four differences that keep a whole-roster run cheap and reviewable.

### Skip what has not moved

Before doing any real work, confirm the watermark is reachable, then check each agent for a delta:

```sh
git rev-parse --verify --quiet <watermark>^{commit} >/dev/null || echo "unreachable"
git log --oneline <watermark>..HEAD -- '<paths from frontmatter>' | wc -l
```

**Do not add `--all` here.** `--all` adds every ref to the walk and thereby overrides the
`<watermark>..HEAD` restriction — the count comes back as the entire history and nothing is ever
skipped. `--all` belongs to the unranged discovery searches in `/fai-create`, never to a range
query.

Zero commits → the agent is **up to date**. Skip it entirely, spend no turns on it, and list it as
up-to-date in the summary. On a typical roster most agents skip, and the run costs a fraction of
training each one by hand.

A watermark that is missing or unreachable (rebased away, shallow clone) cannot be diffed — flag the
agent as **needs attention** and move on rather than guessing a range.

### Reconcile in parallel, one agent per subagent

Reconciliation (step 3) is read-only research and independent per feature, so dispatch the
surviving agents concurrently in a single message. Each returns a proposed change set; nothing is
written yet.

### Propose once, not N times

Merge every agent's change set into **one** approval table grouped by agent. The user approves the
whole run in a single decision instead of being interrupted per agent:

```markdown
## Proposed updates — 5 agents, 3 with changes

| Agent | Watermark | Changes |
| --- | --- | --- |
| 🟣 `checkout` | `a1b2c3d` → `f9e8d7c` | 2 stale, 1 shipped, 1 do-not-reintroduce |
| 🟢 `pricing` | `a1b2c3d` → `f9e8d7c` | 1 new rule, `paths:` gains `src/pricing/rules/**` |
| 🔵 `auth` | `9f8e7d6` → `f9e8d7c` | 1 stale citation |
| ⚪ `search` | — | up to date, skipped |
| 🟠 `notifications` | `c3d4e5f` | ⚠️ watermark not in history — needs attention |

<then the per-agent detail tables, as in step 4>

Apply all, or name the agents or rows to skip.
```

### Apply sequentially, never abort the run

Apply approved changes one agent at a time. If one agent fails — an edit does not match, a stub
cannot be written — record the failure, **continue with the rest**, and report it at the end. One
bad agent must not cost the user the other four.

### Summary

Close with a table: agents updated · claims corrected · claims removed · claims added · stubs
regenerated · agents skipped as up-to-date · agents needing attention, each with the reason.

Then name any agent that looks structurally stale rather than merely behind — a code map whose
paths mostly no longer exist is a `/fai-create` candidate, not a `/fai-train` one. Say so.
