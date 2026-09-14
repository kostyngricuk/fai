# Routing stub template

The skeleton for `.claude/agents/fai-<slug>.md`.

Claude Code only auto-discovers subagents from `.claude/agents/`. This stub is what makes a feature
agent dispatchable by name; the knowledge lives in `.fai/features/<slug>.md`. Keeping them split
means the knowledge file stays a readable, hand-editable document while routing stays native.

## Rules

- The `description` is copied **verbatim** from the knowledge file's frontmatter. It is the routing
  surface: pack it with triggers (paths, symbols, route names, ticket prefixes, domain nouns).
  Never let the two drift — `/fai-train` regenerates this file whenever the description changes.
- The stub carries **no knowledge of its own**. Never duplicate facts here.
- Pick `color` from 🟣 purple, 🟢 green, 🔵 blue, 🟠 orange, 🟡 yellow, 🔴 red, ⚪ white,
  🟤 brown — stable per agent name, and not already taken by another `fai-*` agent in this repo.
- Add a `tools:` line only to *restrict* the agent. Omit it to grant everything, which is the
  default and is usually what you want — feature agents should be free to use whatever tools,
  skills, and MCP servers the project has.

---8<--- TEMPLATE BEGINS ---8<---

---
name: fai-<slug>
description: <copied verbatim from .fai/features/<slug>.md frontmatter>
model: opus
color: <color>
---

You are the `fai-<slug>` feature agent.

Your full instructions live in `.fai/features/<slug>.md`. **Read that file first** and follow it
exactly — it is the source of truth for this feature, and this stub deliberately carries no
knowledge of its own.

If that file is missing, say so and stop; do not improvise a plan from general knowledge.
