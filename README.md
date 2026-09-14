# FAI — Feature Agents Infrastructure

Reusable infrastructure for creating **feature-scoped AI agents** that know your business logic, and
using them to plan, review, and ship changes.

**FAI** stands for *Feature Agents Infrastructure*. It is not an assistant — it is the folders,
skills, and conventions that let feature agents be created, used, persisted, and kept current. The
name is the namespace: a `.fai/` folder, `fai-*` skills, and `fai-<slug>` agents.

Drop it into any project, in any language.

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

Then it gets **better over time**: `/fai-train` feeds it new commits, merged PRs, and updated docs,
reconciling every old claim against the current code rather than just appending.

When work spans several features, the agents collaborate — each one planning inside its own
boundary and handing the next one an explicit contract, like a team.

---

## How it works

```
/fai-create checkout ──► research code + git history + tickets, then interview you
                     ├─► .fai/features/checkout.md        (the knowledge — readable, hand-editable)
                     └─► .claude/agents/fai-checkout.md  (routing stub — so Claude can dispatch it)

/fai-plan <ticket>   ──► fetch ticket ─► route to agents ─► sequential handoff ─► one merged plan
/fai-save-plan       ──► .fai/plans/feat-1234.md   (resumable in a future session)
/fai-review          ──► feature agents + generic reviewer ─► merged, severity-ranked findings
/fai-train checkout  ──► new commits / PRs / docs ─► reconcile, propose, update, bump watermark
/fai-train           ──► no args: same, across every agent you have
```

---

## Requirements

- **Claude Code.** That is the only hard requirement.
- **git** — optional but strongly recommended. History is where the *why* lives; without it,
  `/fai-create` still works but loses its best source.
- **`gh` CLI** — optional. Enables reading GitHub issues and PRs.
- **A tracker MCP** (Linear, Jira, …) — optional. Enables reading tickets directly.

Every integration degrades gracefully. Missing a tool produces a printed note, never a failure.

## Install

Copy the skills into your project. That is the whole install:

```sh
cp -r /path/to/ai-feature-agents/.claude/skills/fai-* your-project/.claude/skills/
```

No directory setup. `.fai/features/`, `.fai/plans/`, and `.claude/agents/` are created on demand the
first time a skill needs to write to them — `/fai-create` makes `.fai/features/` and
`.claude/agents/`, `/fai-save-plan` makes `.fai/plans/`.

Commit what lands there; that is the point. The knowledge is a team asset, not a personal cache.

## Quickstart

```
/fai-create checkout
```

Researches the feature, mines git history for ticket ids and reverted decisions, reads the tickets
it finds, then asks you the handful of things it could not discover. Writes
`.fai/features/checkout.md` and `.claude/agents/fai-checkout.md`.

```
/fai-plan https://linear.app/acme/issue/TID-1234
```

Fetches the ticket, routes it to `fai-checkout` (and `fai-pricing`, if it touches both), runs them
in sequence so the second builds on the first, and prints one dependency-ordered plan.

```
/fai-save-plan TID-1234
```

Writes `.fai/plans/feat-1234.md` with an execution log, so next week's session can resume mid-plan.

```
/fai-review
```

Reviews your working diff through both lenses and prints severity-ranked findings.

```
/fai-train checkout
```

Feeds everything merged since the agent's watermark back into it. Drop the slug — just
`/fai-train` — to do that for every agent at once.

---

## Skill reference

### `/fai-create <feature>`

Creates a feature agent.

1. **Preflight** — slugify; refuse to overwrite an existing agent (points you at `/fai-train`).
2. **Codebase research** — three parallel `Explore` agents covering the surface (routes, screens,
   handlers), the data layer (state, API, types, services), and supporting material (tests, config,
   i18n, docs).
3. **Git archaeology** — commit conventions, ticket ids harvested from subjects *and* trailers, when
   each file was introduced, who owns it, and — most valuable — `git log -S` sweeps for constants and
   files that were **deleted on purpose**.
4. **Ticket archaeology** — resolves the harvested ids through the fetch cascade to recover intent.
5. **Interview** — asks only what research could not answer: scope boundary, undocumented invariants,
   known traps, verification commands, neighbouring features.
6. **Writes** the knowledge file and the routing stub, and records a watermark SHA.

### `/fai-plan <ticket URL, id, or description>`

Produces one implementation plan.

Fetch cascade: tracker MCP → `gh issue view` → `WebFetch` → ask you to paste. Then it scans
`.fai/features/*.md` frontmatter to pick the owning agents, confirming with you when the match is
ambiguous or spans several features.

Agents run **sequentially, never in parallel** — each sees the full output of every prior agent, so
decisions compound instead of colliding. The merged plan is dependency-ordered across features and
carries a **cross-feature contracts** section plus an explicit **conflicts & risks** section that
surfaces disagreement rather than hiding it.

### `/fai-save-plan <ticket>`

Writes the plan to `.fai/plans/feat-<n>.md` with frontmatter (`ticket`, `feature-agents`, `status`,
`created`, `updated`), the six required content sections, and an **execution log** — a checklist
that lets a future session pick up at the first unchecked step.

### `/fai-review [scope]`

Reviews the working diff by default; also accepts `staged`, a path, a branch, or a PR number.

Two lenses run in parallel:

- **Feature agents** own the changed paths and check what only they can — weakened invariants,
  reintroduced deletions, convention drift, missed lockstep updates, boundary contracts.
- **A generic reviewer** checks correctness, security, performance, and test impact.

Findings are deduped (corroborated ones rank higher), severity-ranked, colorized per agent, and
**printed only** — nothing is written to disk.

### `/fai-train [slug] [source]`

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
`/fai-train`, so routing never drifts from reality.

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
| 12 | How to produce a plan for a new task | `/fai-plan` |
| 13 | How to review a change in this feature | `/fai-review` |
| 14 | Learning log | `/fai-train` |

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
there. A worked example ships at `.claude/skills/fai-save-plan/example-plan.md`.

## Multi-feature work

When a goal touches several features, `/fai-plan` runs their agents **one at a time**, handing each
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
- **Hand-editing** — `.fai/features/<slug>.md` is a plain document. Edit it. `/fai-train` applies
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
- **Propose before rewriting.** `/fai-train` never silently overwrites a file you may have edited.

## Repo layout

```
.claude/
  skills/
    fai-create/
      SKILL.md
      feature-template.md      # the 14-section knowledge skeleton
      stub-template.md         # the routing-stub skeleton
    fai-plan/SKILL.md
    fai-review/SKILL.md
    fai-train/SKILL.md
    fai-save-plan/
      SKILL.md
      example-plan.md          # worked example of the plan format
  agents/                      # fai-<slug>.md routing stubs        (auto-created)
.fai/
  features/                    # <slug>.md — the knowledge           (auto-created)
  plans/                       # feat-<n>.md — saved, resumable plans (auto-created)
```

Only `.claude/skills/fai-*` is copied at install time. The three directories above appear by
themselves when the first agent or plan is written.

## FAQ

**An agent is not being picked up for work it should own.**
Routing runs off the `description` and `paths:` in `.fai/features/<slug>.md`. Add the missing
triggers — file paths, symbol names, route names, ticket prefixes, domain nouns — then run
`/fai-train <slug>` so the stub is regenerated to match.

**`/fai-plan` says no agent matched.**
Either the feature has no agent yet (`/fai-create <name>`), or the ticket describes it in vocabulary
the description does not contain. You can always proceed with the built-in planner; the output will
be flagged as not feature-grounded.

**The agent is confidently wrong about the code.**
It has drifted past its watermark. Run `/fai-train <slug>` — it reconciles every claim against the
current tree rather than just adding new ones. If the drift came from one PR, pass it:
`/fai-train checkout 481`.

**My project is not a git repo.**
Everything still works. `/fai-create` skips history research with a printed note, and the interview
step carries more weight — you will be asked more questions.

**The stub and the knowledge file disagree.**
`/fai-train <slug>` regenerates the stub from the knowledge file, which is always the source of
truth.

**I already have a `/review` skill.**
Nothing is shadowed — this kit's review is `/fai-review`. All five skills are namespaced under
`fai-` precisely so they never collide with your existing skills or agents.
