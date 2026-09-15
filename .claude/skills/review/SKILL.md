---
name: review
description: "Review changes through two lenses at once — the FAI feature agents that own the touched code (business rules, invariants, conventions, regressions) and a generic reviewer (correctness, security, performance, tests) — then merge into one severity-ranked, colorized report. Use when asked to review a diff, changes, a branch, or a PR."
argument-hint: <optional scope — a path, branch, PR number, or 'staged'; defaults to the working diff>
---

Review `$ARGUMENTS` (default: the current working diff) with **two lenses**, merged into one report.

A generic reviewer catches what is wrong with the code. A feature agent catches what is wrong with
the *change* — an invariant weakened, a deleted thing reintroduced, an i18n key or registration
missed. Neither lens substitutes for the other, so run both.

Findings are **printed, never written to disk.**

## Non-negotiable rules

1. **Evidence-based.** Every finding cites `file:line`. No finding without a location.
2. **Read-only.** If reviewing a PR, read it; never post comments, reviews, or approvals.
3. **Graceful degradation.** No feature agents, no `Reviewer` agent, no git — run what you can and
   say which lens was unavailable.
4. **Say when it is clean.** A review that manufactures findings to look thorough is worse than
   useless. "No blocking issues" is a valid verdict.

---

## Procedure

### Step 1 — Resolve scope

| Argument | Command |
| --- | --- |
| *(none)* | `git diff` — working tree |
| `staged` | `git diff --staged` |
| a path | `git diff -- <path>` |
| a branch | `git diff <base>...HEAD` |
| a PR number | `gh pr diff <n>` (and `gh pr view <n>` for intent) |

No commit history (`git rev-parse HEAD` fails), or the diff is empty → ask the user which
paths to review.

### Step 2 — Route

```sh
git diff --name-only <scope>
```

Match the changed paths against each `.fai/features/*.md` frontmatter `paths:` list and code map.
Build the set of matched feature agents. Report which files matched no agent — those get the generic
lens only, and that gap is worth saying out loud.

### Step 3 — Run both lenses in parallel

Dispatch in a single message so they run concurrently.

**Feature lens** — one `fai-<slug>` subagent per matched feature. Give each the diff for its own
files plus the ticket context if known, and tell it to follow **section 13 ("How to review a change
in this feature")** of its own knowledge file: invariants, reintroductions, convention drift,
lockstep updates, regression surfaces, boundary contracts.

**Generic lens** — correctness, security, performance, test impact. Use the `Reviewer` subagent
**if it exists in this environment**; otherwise fall back to `general-purpose` with an equivalent
brief. This kit runs on other people's machines — check, do not assume.

### Step 4 — Merge

- **Dedupe** findings both lenses raised. Keep the feature agent's phrasing (it explains the
  business consequence) and note the corroboration. **Corroborated findings rank higher.**
- **Drop** generic findings a feature agent explicitly refutes as intentional for this feature —
  but mention the refutation, do not silently delete.
- **Sort by severity**, then by file.

### Step 5 — Print

```markdown
## 🔴 Critical

> 🟣 **fai-checkout** · `src/cart/total.ts:88`
> **Discount stacking guard removed.** `applyPromo` no longer checks `isStackable` before
> combining promos.
> **Why it matters:** two stackable-false promos now compound, so a 30% + 40% pair applies as 58%
> off instead of 40%. This is the invariant at `.fai/features/checkout.md` §5.
> **Fix:** restore the `isStackable` short-circuit, and extend `__tests__/cart/promo.ts` with the
> two-promo case.

## 🟠 Major

> 🟢 **fai-payment** · `src/payment/methods.ts:12` — *corroborated by the generic reviewer*
> …

## 🟡 Minor
## 🔵 Notes

### Verdict

🟠 **Request changes** — 1 critical, 2 major, 1 minor.
Fix the stacking guard first; the rest can follow in the same PR.

*Lenses run: fai-checkout 🟣, fai-payment 🟢, generic Reviewer. Not covered: `scripts/build.mjs`
(no feature agent owns it).*
```

Severity vocabulary: 🔴 CRITICAL (ship-blocking) · 🟠 MAJOR (fix before merge) · 🟡 MINOR (worth
fixing) · 🔵 NOTE (informational).

Agent color dots are stable per agent name: 🟣 🟢 🔵 🟠 🟡 🔴 ⚪ 🟤.

Every finding needs: **agent + `file:line`**, what is wrong, **why it matters for this feature
specifically**, and a concrete fix. End with a one-line verdict and the suggested next action.
