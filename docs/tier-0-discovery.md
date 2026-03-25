# Tier 0: The Discovery Layer

The [Agent Skills spec](https://agentskills.io/specification#progressive-disclosure) defines three tiers of progressive disclosure:

1. **Metadata** (~100 tokens) — skill name + description, loaded at startup
2. **Instructions** (<5000 tokens) — full SKILL.md body, loaded on activation
3. **Resources** (as needed) — reference files, loaded when instructions reference them

This document describes **Tier 0** — a discovery layer that sits outside the skill itself and answers the question: *how does the user or model know which skills exist before any skill is activated?*

## The Problem Tier 0 Solves

The spec's Tier 1 (Metadata) loads skill names and descriptions at startup. But this only works when the agent's harness builds a skill catalog automatically ([client implementation guide](https://agentskills.io/client-implementation/adding-skills-support#step-3-disclose-available-skills-to-the-model)). In practice:

- Not all clients build catalogs the same way
- Users working across multiple projects forget which skills are available where
- New team members don't know what skills exist or when to use them
- The model may not connect a user's request ("add caching to the API") to the right skill (`/saw scout`)

Tier 0 provides a human-readable, always-loaded index that bridges this gap.

## Implementation: Project Config Files

Most AI coding tools load a project configuration file at session start — before any skill is activated, before any user message is processed:

| Tool | Config file |
|------|-------------|
| Claude Code | `CLAUDE.md` (project root or `~/.claude/CLAUDE.md`) |
| Cursor | `.cursorrules` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Windsurf | `.windsurfrules` |

These files are the natural home for Tier 0. They're always in context, they're project-specific, and they cost zero tokens at invocation time (they're already loaded).

## What Goes in Tier 0

A Tier 0 entry is a minimal index — just enough for the model to route the user to the right skill:

```markdown
## Available Skills

- `/saw` — Parallel agent coordination for feature work.
  Use for any task that can be decomposed across files.
  Subcommands: scout, wave, status, bootstrap, interview, program, amend.

- `/deploy` — Deploy to staging or production.
  Use when the user wants to ship code.

- `/review` — AI code review on open PRs.
  Use when the user mentions reviewing or approving changes.
```

### What belongs

- Skill name and one-sentence purpose
- The trigger condition ("use when X")
- Top-level subcommand list (breadth-first, no depth)

### What does not belong

- Subcommand flags or options (Tier 2)
- Flow logic or implementation details (Tier 2)
- Reference material (Tier 3)
- Anything duplicating content from the skill's frontmatter or SKILL.md

The heuristic: if removing it from the index would prevent the model from routing to the skill, it belongs. Everything else is Tier 2 or Tier 3.

## The Four-Tier Model

With Tier 0, the full progressive disclosure stack is:

| Tier | What | When | Token cost | Where |
|------|------|------|------------|-------|
| 0 — Discovery | Skill index | Session start | ~20-50 tokens per skill | Project config (CLAUDE.md, .cursorrules, etc.) |
| 1 — Metadata | Name + description | Session start | ~50-100 tokens per skill | SKILL.md frontmatter |
| 2 — Instructions | Full SKILL.md body | Skill activation | <5000 tokens | SKILL.md body |
| 3 — Resources | Reference files | Trigger match | Varies | `references/`, `scripts/`, `assets/` |

Tier 0 and Tier 1 are both loaded at startup, but they serve different audiences:
- **Tier 0** is for the user and the model's routing logic — "what skills exist and when to use them"
- **Tier 1** is for the agent harness — structured metadata for catalog building and tool registration

## Tier 0 as a Multi-Skill Index

In a project with multiple skills, Tier 0 acts as a table of contents. The model reads the index, identifies the relevant skill, and activates it. Only then do Tiers 2 and 3 load.

For a project with 10 installed skills, the cost is ~200-500 tokens for the full index — compared to 50,000+ tokens if all 10 skill bodies loaded at once. The index pays for itself on the first invocation.

## Relationship to Context Injection

Tier 0 complements context injection — they operate at different points in the lifecycle:

1. **Tier 0** helps the model identify which skill to activate (pre-activation)
2. **Context injection** loads the right reference files once a skill is activated (post-activation, pre-execution)

Without Tier 0, the model may not activate the skill at all. Without context injection, the model may activate the skill but miss critical reference material. Both are needed for reliable end-to-end progressive disclosure.
