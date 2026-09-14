---
name: fai-create
description: "Create a new FAI feature agent — research a feature's code, git history, and tickets, interview the user about business rules, then write a self-contained specialist agent to .fai/features/<slug>.md plus a routing stub in .claude/agents/. Use when asked to create/add/set up a feature agent, onboard a feature, or make an agent that knows a specific feature."
argument-hint: <feature name or description, e.g. "checkout" or "the discount engine in src/pricing">
---

Build a **feature agent**: a self-contained specialist that knows one feature's code, business
logic, and history well enough to turn a ticket into a file-level plan that preserves the business
logic.

FAI = Feature Agents Infrastructure. Everything this skill writes follows the `fai-` namespace.

## Output (two files)

| File | Role |
| --- | --- |
| `.fai/features/<slug>.md` | The knowledge. Source of truth. Everything lives here. |
| `.claude/agents/fai-<slug>.md` | A thin routing stub so Claude Code can dispatch to the agent by name. Carries no knowledge of its own. |

Templates: `feature-template.md` and `stub-template.md`, both in this skill's folder. Read them
before writing.

## Non-negotiable rules

1. **Evidence-based.** Every claim about code cites a real `path:line`. If something does *not*
   exist, say so explicitly — "there is no `constants.ts` here" is as valuable as a path.
2. **Read-only on external systems.** Read Linear/Jira/GitHub tickets and PRs freely. NEVER write:
   no comments, status changes, reviews, merges, or edits.
3. **Graceful degradation.** No git, no `gh`, no tracker MCP, no test runner — continue anyway and
   print a one-line note saying what you could not check. Never fail the whole skill over a missing
   tool.
4. **Never invent.** If research and the user both come up empty on a section, drop the section
   rather than filling it with plausible-sounding filler.

---

## Procedure

### 0. Preflight

- Slugify the argument into a short kebab-case `<slug>` (e.g. "the discount engine in src/pricing"
  → `discount`). Confirm the slug with the user if the argument is vague.
- If `.fai/features/<slug>.md` already exists: **stop.** Tell the user it exists and recommend
  `/fai-train <slug>` instead. Only overwrite if they explicitly ask.
- Create the destinations before writing anything — never ask the user to set up directories:

  ```sh
  mkdir -p .fai/features .claude/agents
  ```

### 1. Codebase research

Launch up to **3 `Explore` agents in parallel** (one message, multiple tool calls), each with a
distinct focus. Tell each to report real `path:line` citations and to name files that do *not*
exist where relevant.

- **Agent A — surface.** Entry points: routes, screens, pages, CLI commands, HTTP handlers, jobs.
  What the user of this feature actually touches. Navigation/registration files.
- **Agent B — data.** State stores/slices/selectors/actions, API clients and endpoints, models and
  types, DB schema, services, sagas/effects/queues. Where the rules are enforced.
- **Agent C — supporting.** Tests (and their mocking conventions), config and feature flags, i18n
  keys, assets, docs, and anything that must be updated in lockstep with the feature.

Collect the union of paths into a candidate `paths:` list for the frontmatter.

### 2. Git archaeology

First: `git rev-parse HEAD` — not `--git-dir`, which also succeeds in an initialized repo that
has no commits yet. If it fails, print `note: no commit history available, skipping history
research` and jump to step 4.

Otherwise mine history. History is where the *why* lives; code only shows the *what*.

**Pass `--all` to every one of these.** Without it `git log` searches only the current branch, and
a feature whose work landed on another branch returns *zero* results — which reads as "no history"
rather than "wrong branch". This is the single most common way history research silently fails.

```sh
# Commits that name the feature, anywhere in the repo
git log --all --oneline -i --grep="<keyword1>\|<keyword2>"

# Message convention and cadence
git log --all --oneline -30 -- '<feature paths>'

# Ticket ids in subjects AND trailers (e.g. a "META: ABC-123" line)
git log --all --format='%H %s%n%b' -i --grep="<keywords>" \
  | grep -Eo '[A-Z]{2,}-[0-9]+' | sort -u

# Who owns it (shortlog has no --all, so count authors directly)
git log --all --format='%an' -- '<feature paths>' | sort | uniq -c | sort -rn

# When each file was introduced
git log --all --diff-filter=A --format='%h %ad %s' --date=short -- '<feature paths>'

# When a rule/constant appeared OR WAS REMOVED
git log --all -S'<key symbol>' --oneline
```

Quote path arguments and prefer a trailing `*` glob (`'src/features/Checkout*'`). If a path-scoped
command returns nothing while the keyword search returns plenty, the feature's files are **not on
the current checkout** — find the branch (`git log --all --oneline -i --grep=...` then
`git branch -a --contains <sha>`) and read the files from there with `git show <sha>:<path>` rather
than concluding they do not exist.

If `gh` is available and the repo has a GitHub remote, read merged PRs — their descriptions usually
carry the business rationale that commit messages omit:

```sh
gh pr list --search "<keywords>" --state merged --limit 20 --json number,title,body
gh pr view <n>
```

**Hunt specifically for removed and reversed decisions.** Run `git log -S` against constants,
flags, and files that *used* to exist. "We deleted `X`, don't reintroduce it" is the single most
valuable kind of line in a feature agent, and it is invisible to anyone reading only current code.

### 3. Ticket archaeology

Take the ticket ids harvested in step 2 and resolve them through the fetch cascade, stopping at the
first that works:

1. A connected tracker MCP (Linear, Jira, …) — detect what is actually available, do not assume.
2. GitHub — `gh issue view <n> --json title,body,labels,comments`
3. `WebFetch` on the ticket URL
4. Ask the user to paste the ticket text

Extract: product intent, acceptance criteria, explicit non-goals, and open follow-ups. Read-only.

### 4. Interview the user

Use `AskUserQuestion` — at most 4 questions per call, at most 2 rounds. **Only ask what research
could not answer.** Lead with what you found so the user is confirming, not authoring.

Cover:

- **Scope boundary** — present the candidate file set; ask what is adjacent but owned elsewhere.
- **Business rules and invariants not visible in code** — why a rule exists, what must never break,
  what a violation would cost.
- **Known open items, tech debt, traps** — things the code does not admit to.
- **Verification** — exact lint / typecheck / test commands, and any known-failing baseline to
  ignore.
- **Related features** that must not regress — becomes the "Related features & boundaries" section.

### 5. Write the two files

Read `feature-template.md`, then write `.fai/features/<slug>.md` with every section filled from
researched, cited content. Drop sections that genuinely do not apply — an honest 12-section file
beats an 18-section file padded with filler.

Read `stub-template.md`, then write `.claude/agents/fai-<slug>.md`, copying the `description`
verbatim from the knowledge file's frontmatter.

Record the watermark: `git rev-parse --short HEAD` plus today's date. Without git, write
`watermark: none (not a git repo, <date>)`.

### 6. Report

Print:

- both file paths
- a 5-line summary of what the agent covers
- the trigger list that will route work to it
- how to use it: `/fai-plan <ticket>`, `/fai-review`, or by name (`fai-<slug>`)
- the watermark recorded, and anything you could not verify

Then suggest the natural next step: `/fai-plan <ticket>`.
