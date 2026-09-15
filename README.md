![FAI — Feature Agents Infrastructure. One agent per feature; it knows the rules your code does not state. Built from the code, git history, tickets and PRs, and the unwritten rules you supply — into a feature agent such as fai-checkout, which then drives /fai:plan, /fai:review, /fai:train, and /fai:save-plan.](fai-banner.png)

# FAI — Feature Agents Infrastructure

Reusable infrastructure for creating **feature-scoped AI agents** that know your business logic, and
using them to plan, review, and ship changes.

**FAI** stands for *Feature Agents Infrastructure*. It is not an assistant — it is the folders,
skills, and conventions that let feature agents be created, used, persisted, and kept current. The
name is the namespace: a `.fai/` folder, `/fai:*` commands, and `fai-<slug>` agents.

Drop it into any project, in any language.

**Repo:** <https://github.com/kostyngricuk/fai> · distributed through the [khdev](https://github.com/kostyngricuk/khdev) marketplace

---

## The problem

A general-purpose coding assistant starts every session from zero. It reads your code, infers a
plausible design, and writes a plan that compiles. What it cannot do is know:

- **why** a rule exists — the reasoning lives in a Jira ticket from eight months ago
- that a constant was **deliberately deleted** and must not come back
- which three files must change **in lockstep** with this one (the i18n keys, the type union, the
  stack registration)
- that the invariant "at most one non-stackable promo per order" is real, load-bearing, and
  currently enforced in exactly one place

So you get plans that are plausible, confident, well-formatted — and quietly wrong about the
business logic. You catch it in review, or you catch it in production.

## The idea

**One agent per feature.** Each is a self-contained specialist, built once from four sources:

| Source | What it contributes |
| --- | --- |
| The code | The file map, the contracts, where rules are enforced |
| Git history | Why things are the way they are; what was tried and reverted |
| Tickets and PRs | Product intent, acceptance criteria, open follow-ups |
| You | The rules that were never written down anywhere |

Then it gets **better over time**: `/fai:train` feeds it new commits, merged PRs, and updated docs,
reconciling every old claim against the current code rather than just appending.

When work spans several features, the agents collaborate — each one planning inside its own
boundary and handing the next one an explicit contract, like a team.

---

## How it works

```
/fai:create checkout ──► research code + git history + tickets, then interview you
                     ├─► .fai/features/checkout.md        (the knowledge — readable, hand-editable)
                     └─► .claude/agents/fai-checkout.md  (routing stub — so Claude can dispatch it)

/fai:plan <ticket>   ──► fetch ticket ─► route to agents ─► sequential handoff ─► one merged plan
/fai:save-plan       ──► .fai/plans/feat-1234.md   (resumable in a future session)
/fai:review          ──► feature agents + generic reviewer ─► merged, severity-ranked findings
/fai:train checkout  ──► new commits / PRs / docs ─► reconcile, propose, update, bump watermark
/fai:train           ──► no args: same, across every agent you have
```

---

## Requirements

- **Claude Code.** That is the only hard requirement.
- **git** — optional but strongly recommended. History is where the *why* lives; without it,
  `/fai:create` still works but loses its best source.
- **`gh` CLI** — optional. Enables reading GitHub issues and PRs.
- **A tracker MCP** (Linear, Jira, …) — optional. Enables reading tickets directly.

Every integration degrades gracefully. Missing a tool produces a printed note, never a failure.

## Install

FAI ships as a Claude Code plugin. Two commands, from inside any project:

```
/plugin marketplace add kostyngricuk/khdev
/plugin install fai@khdev
```

That is the whole install. The five commands — `/fai:create`, `/fai:plan`, `/fai:save-plan`,
`/fai:review`, `/fai:train` — are available immediately.

To update later:

```
/plugin update fai@khdev
```

No directory setup. `.fai/features/`, `.fai/plans/`, and `.claude/agents/` are created on demand in
*your* project the first time a skill needs to write to them — `/fai:create` makes `.fai/features/`
and `.claude/agents/`, `/fai:save-plan` makes `.fai/plans/`.

Commit what lands there; that is the point. The knowledge is a team asset, not a personal cache. The
plugin itself is not vendored into your repo — only the knowledge it produces.

> Copying `.claude/skills/*` by hand is no longer supported. The skills are named `create`, `plan`,
> `review`, `train`, and `save-plan`, and only earn their `fai:` prefix by being loaded as a plugin —
> copied in loose, they would collide with your own skills of the same name.

## Quickstart

```
/fai:create checkout
```

Researches the feature, mines git history for ticket ids and reverted decisions, reads the tickets
it finds, then asks you the handful of things the code cannot answer — each question anchored to a
real line it found. Writes `.fai/features/checkout.md` and `.claude/agents/fai-checkout.md`.

```
/fai:plan https://linear.app/acme/issue/TID-1234
```

Fetches the ticket, routes it to `fai-checkout` (and `fai-pricing`, if it touches both), runs them
in sequence so the second builds on the first, and prints one dependency-ordered plan.

```
/fai:save-plan TID-1234
```

Writes `.fai/plans/feat-1234.md` with an execution log, so next week's session can resume mid-plan.

```
/fai:review
```

Reviews your working diff through both lenses and prints severity-ranked findings.

```
/fai:train checkout
```

Feeds everything merged since the agent's watermark back into it. Drop the slug — just
`/fai:train` — to do that for every agent at once.

---

## Skill reference

### `/fai:create <feature>`

Creates a feature agent.

1. **Preflight** — slugify; refuse to overwrite an existing agent (points you at `/fai:train`).
2. **Codebase research** — three parallel `Explore` agents covering the surface (routes, screens,
   handlers), the data layer (state, API, types, services), and supporting material (tests, config,
   i18n, docs).
3. **Git archaeology** — commit conventions, ticket ids harvested from subjects *and* trailers, when
   each file was introduced, who owns it, and — most valuable — `git log -S` sweeps for constants and
   files that were **deleted on purpose**.
4. **Ticket archaeology** — resolves the harvested ids through the fetch cascade to recover intent.
5. **Interview** — two gated rounds, both skippable. First, if it discovered references you did not
   name (docs, open PRs, API contracts), it asks which of them to use and reads only those. Then at
   most four questions, each anchored to a specific `path:line`, sha, or quoted passage — never an
   open-ended "any gotchas?", and never anything the repo already answers.
6. **Writes** the knowledge file and the routing stub, and records a watermark SHA.

### `/fai:plan <ticket URL, id, or description>`

Produces one implementation plan.

Fetch cascade: tracker MCP → `gh issue view` → `WebFetch` → ask you to paste. Then it scans
`.fai/features/*.md` frontmatter to pick the owning agents, confirming with you when the match is
ambiguous or spans several features.

Agents run **sequentially, never in parallel** — each sees the full output of every prior agent, so
decisions compound instead of colliding. The merged plan is dependency-ordered across features and
carries a **cross-feature contracts** section plus an explicit **conflicts & risks** section that
surfaces disagreement rather than hiding it.

### `/fai:save-plan <ticket>`

Writes the plan to `.fai/plans/feat-<n>.md` with frontmatter (`ticket`, `feature-agents`, `status`,
`created`, `updated`), the six required content sections, and an **execution log** — a checklist
that lets a future session pick up at the first unchecked step.

### `/fai:review [scope]`

Reviews the working diff by default; also accepts `staged`, a path, a branch, or a PR number.

Two lenses run in parallel:

- **Feature agents** own the changed paths and check what only they can — weakened invariants,
  reintroduced deletions, convention drift, missed lockstep updates, boundary contracts.
- **A generic reviewer** checks correctness, security, performance, and test impact.

Findings are deduped (corroborated ones rank higher), severity-ranked, colorized per agent, and
**printed only** — nothing is written to disk.

### `/fai:train [slug] [source]`

Updates an agent from new commits (the default, since its watermark), a PR, a doc, a URL, or a
direct correction from you.

**Called with no arguments it trains every agent you have** — the routine maintenance mode. Run it
after a merge, or once a week, to pull the whole roster back in sync. Agents with no commits since
their watermark are detected and skipped for free, the rest reconcile in parallel, and you approve
the entire run in **one** table instead of being interrupted per agent. A failure on one agent never
aborts the others.

The important part is that it is a **reconciliation, not an append**. Every existing claim is
re-checked against the code and classified `confirmed` / `stale` / `removed` / `new`. Deletions in
the delta become "do not reintroduce" warnings. You approve the change set before anything is
written, edits are applied surgically so your hand-written nuance survives, the watermark is bumped,
and the routing stub is regenerated if the triggers changed.

---

## Anatomy of a feature agent

An agent is two files, deliberately split:

| File | Why it is separate |
| --- | --- |
| `.fai/features/<slug>.md` | All the knowledge. A readable document you can review in a PR and edit by hand. |
| `.claude/agents/fai-<slug>.md` | A 5-line stub. Claude Code only auto-discovers agents from `.claude/agents/`, so this is what makes the agent dispatchable by name. It carries no knowledge. |

The stub's `description` is copied verbatim from the knowledge file's frontmatter and regenerated by
`/fai:train`, so routing never drifts from reality.

The knowledge file's sections, and who reads them:

| § | Section | Read by |
| --- | --- | --- |
| — | Role, read-only mandate, **staleness watermark** | always |
| 1 | What this feature is | always |
| 2 | Architecture & core contracts | plan, review |
| 3 | Code map | plan, review, routing |
| 4 | State & data flow | plan |
| 5 | **Business rules & invariants** | plan, review |
| 6 | Conventions to reuse, not reinvent | plan, review |
| 7 | **Built vs. explicitly still open** (+ "do not reintroduce") | plan, review, train |
| 8 | Tests | plan, review |
| 9 | Verification commands | plan, save-plan |
| 10 | Commit / PR / ticket convention | plan |
| 11 | Related features & boundaries | plan routing |
| 12 | External contracts & references | plan, review |
| 13 | How to produce a plan for a new task | `/fai:plan` |
| 14 | How to review a change in this feature | `/fai:review` |
| 15 | Learning log | `/fai:train` |

Sections 5 and 7 are the ones that justify the whole exercise. Everything else a capable assistant
could re-derive from the code; those two it cannot.

## The plan format

Plans in `.fai/plans/` carry frontmatter (`ticket`, `feature-agents`, `status`, `created`, `updated`)
and seven sections: Title with an explicit scope boundary · Context · **Current-state facts
(verified)** with `path:line` citations and explicit "does NOT exist" statements · Implementation
steps tagged by owning agent · Verification using the project's real commands · Out of scope and
dependencies · **Execution log**.

The execution log is what makes a plan survive a session boundary. To resume: read the plan, find
the first unchecked box, **re-verify the current-state facts** (code moves), then continue from
there. A worked example ships at `.claude/skills/save-plan/example-plan.md`.

## Multi-feature work

When a goal touches several features, `/fai:plan` runs their agents **one at a time**, handing each
the complete output of the ones before it, with a standing instruction:

> Plan only within your own feature. Where you depend on another feature, state the **contract** you
> need — the exact type, endpoint, event, or flag — and leave the implementation to that feature's
> agent. Reconcile with decisions already made rather than restating them. If you disagree, say so
> explicitly and explain the consequence; do not silently plan around it.

That produces two things a parallel fan-out cannot: a **cross-feature contracts** section that both
sides have actually agreed on, and a **conflicts & risks** section where disagreement is visible
instead of quietly averaged away.

## Extending the kit

Feature agents are ordinary Claude Code subagents. They can use **any** tools, skills, plugins, and
MCP servers the project has — Context7 for library docs, a tracker MCP for tickets, Chrome DevTools
or Maestro for E2E, your own project skills.

- **Tools** — the stub grants everything by default. Add a `tools:` line to the stub frontmatter
  only to *restrict* it.
- **Skills** — reference other skills directly from a knowledge file (e.g. "use the project's
  `/migrate` skill to generate the migration"); the agent will invoke them.
- **MCP** — whatever is connected to the session is available. The skills detect what exists rather
  than assuming, so a missing server degrades to the next option in the cascade.
- **Hand-editing** — `.fai/features/<slug>.md` is a plain document. Edit it. `/fai:train` applies
  surgical edits precisely so your additions survive.

## Guardrails

Each of these exists for a reason, and each skill restates the ones it needs.

- **Read-only on external systems.** Agents read tickets and PRs freely and never write to them —
  no comments, status changes, reviews, or merges. An agent that can post to your tracker is an
  agent that will, at the worst possible moment.
- **Evidence-based.** Every claim about code cites `path:line`. Explicit "this does NOT exist"
  statements are required, because they are what stop the next agent from inventing it.
- **Staleness watermark.** Every agent records the commit it was built from and is instructed to
  trust the code over itself when they disagree, and to report the drift. A confident, cited, wrong
  agent is worse than no agent.
- **Graceful degradation.** No git, no `gh`, no MCP, no test runner — every skill still runs and
  prints what it could not check. This kit runs on other people's machines.
- **Propose before rewriting.** `/fai:train` never silently overwrites a file you may have edited.

## Repo layout

This repo — the plugin itself:

```
.claude-plugin/
  plugin.json                  # the plugin manifest
.claude/
  skills/                      # ← plugin.json points `skills` here
    create/
      SKILL.md
      feature-template.md      # the 15-section knowledge skeleton
      stub-template.md         # the routing-stub skeleton
    plan/SKILL.md
    review/SKILL.md
    train/
      SKILL.md
      stub-template.md         # same skeleton — every skill reads only its own folder
    save-plan/
      SKILL.md
      example-plan.md          # worked example of the plan format
```

Your project — what the commands write, all of it created on demand:

```
.claude/
  agents/                      # fai-<slug>.md routing stubs
.fai/
  features/                    # <slug>.md — the knowledge
  plans/                       # feat-<n>.md — saved, resumable plans
```

Nothing from this repo is copied into your project. The plugin lives in Claude Code's plugin cache;
only the knowledge your agents accumulate lands in your tree, which is exactly what you want to
commit.

The marketplace catalogue that makes `/plugin marketplace add` work lives in a separate repo,
[kostyngricuk/khdev](https://github.com/kostyngricuk/khdev) — this repo is the plugin and nothing
else.

## FAQ

**An agent is not being picked up for work it should own.**
Routing runs off the `description` and `paths:` in `.fai/features/<slug>.md`. Add the missing
triggers — file paths, symbol names, route names, ticket prefixes, domain nouns — then run
`/fai:train <slug>` so the stub is regenerated to match.

**`/fai:plan` says no agent matched.**
Either the feature has no agent yet (`/fai:create <name>`), or the ticket describes it in vocabulary
the description does not contain. You can always proceed with the built-in planner; the output will
be flagged as not feature-grounded.

**The agent is confidently wrong about the code.**
It has drifted past its watermark. Run `/fai:train <slug>` — it reconciles every claim against the
current tree rather than just adding new ones. If the drift came from one PR, pass it:
`/fai:train checkout 481`.

**My project is not a git repo.**
Everything still works. `/fai:create` skips history research with a printed note, and the interview
step carries more weight — you will be asked more questions.

**The stub and the knowledge file disagree.**
`/fai:train <slug>` regenerates the stub from the knowledge file, which is always the source of
truth.

**I already have a `/review` skill.**
Nothing is shadowed. Claude Code namespaces every plugin skill under its plugin name, so this kit's
review is `/fai:review` and yours stays `/review`. The same holds for `/fai:plan`, `/fai:create`,
`/fai:train`, and `/fai:save-plan` — the `fai:` prefix is what keeps them out of your way.

**I used to run `/fai-create`, and it is gone.**
The commands moved to the plugin namespace when FAI became installable: `/fai-create` is now
`/fai:create`, `/fai-plan` is `/fai:plan`, and so on. Your existing `.fai/features/` knowledge files
and `fai-<slug>` agents are untouched and keep working — only the command names changed.
