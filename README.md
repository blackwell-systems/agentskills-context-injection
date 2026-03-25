# agentskills-context-injection

[![Blackwell Systems™](https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg)](https://github.com/blackwell-systems)
[![Agent Skills](assets/badge-agentskills.svg)](https://agentskills.io)

Context injection for [Agent Skills](https://agentskills.io) progressive disclosure. Automatically loads reference files into model context based on trigger patterns declared in skill frontmatter.

> Built on the [Agent Skills](https://agentskills.io) open standard. Uses only existing spec conventions (`scripts/`, `references/`, `metadata:` extension point) — no spec modifications required. Compatible with Claude Code, Cursor, GitHub Copilot, and other Agent Skills-compatible tools.

## The Problem

The [Agent Skills spec](https://agentskills.io/specification#progressive-disclosure) defines three-tier progressive disclosure:

1. **Metadata** (~100 tokens) -- loaded at startup
2. **Instructions** (<5000 tokens) -- loaded on skill activation
3. **Resources** (as needed) -- loaded when instructions reference them

Tier 3 (Resources) loading is convention-based -- the model decides when to read reference files. It can ignore routing tables, pre-load everything, or load references at the wrong time. There is no enforcement.

## The Solution

Two layers of context injection using the spec's existing conventions (`scripts/`, `references/`, `metadata:` extension point). No spec modifications required.

### Layer 1: Injection Script (vendor-neutral)

A `scripts/inject-context` script bundled with each skill. Works on **any** Agent Skills-compliant client that can execute Bash.

```bash
# Model runs this before executing the skill
bash scripts/inject-context "/saw program execute add caching"
# Outputs: contents of references/program-flow.md
```

The skill's `SKILL.md` instructions tell the model: "Before executing, run `scripts/inject-context` with the user's prompt." Simple convention, hard to skip.

### Layer 2: UserPromptSubmit Hook (Claude Code)

For Claude Code users, a lifecycle hook injects references **before** the model runs -- no model decision required.

1. User types `/saw program execute "add caching"`
2. Hook fires, iterates installed skills
3. Each skill's `scripts/inject-context` runs against the prompt
4. Matching reference content is returned as `additionalContext`
5. Model receives references in context before it starts

Deterministic. The model cannot skip or misroute.

## Trigger Definitions

Skills declare triggers in YAML frontmatter using the `triggers:` field (via the spec's `metadata:` extension point):

```yaml
---
name: my-skill
description: Does things
triggers:
  - match: "^/my-skill subcommand"
    inject: references/subcommand-flow.md
  - match: "error|failed|blocked"
    inject: references/troubleshooting.md
---
```

- `match`: regex pattern tested against the full prompt text
- `inject`: path relative to the skill directory
- Multiple matches -> all matching references injected (concatenated)
- No match -> no injection, zero overhead

## Installation

### The injection script (any skill)

Copy `scripts/inject-context` into your skill's `scripts/` directory:

```bash
cp scripts/inject-context ~/.agents/skills/my-skill/scripts/
chmod +x ~/.agents/skills/my-skill/scripts/inject-context
```

Add `triggers:` to your skill's frontmatter, and add this to your `SKILL.md` instructions:

```markdown
Before executing any subcommand, run:
  bash scripts/inject-context "<user prompt>"
and incorporate the output as context.
```

### The Claude Code hook (optional, Claude Code only)

```bash
# Install the hook script
cp hooks/inject_skill_context ~/.local/bin/
chmod +x ~/.local/bin/inject_skill_context
```

Add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "inject_skill_context"
          }
        ]
      }
    ]
  }
}
```

The hook iterates all skill directories (`~/.claude/skills/`, `~/.agents/skills/`) and delegates to each skill's `scripts/inject-context`. Adding a new skill requires zero hook changes.

## Redundancy Model

All three layers are active simultaneously:

| Layer | Mechanism | Vendor-neutral | Enforcement |
|-------|-----------|----------------|-------------|
| Hook | `UserPromptSubmit` | Claude Code only | Deterministic (pre-model) |
| Script | `scripts/inject-context` | Any agent with Bash | Model-initiated |
| Fallback | Routing table in SKILL.md | Any agent | Convention-based |

Users get the best available layer. No regression at any level.

## Spec Alignment

This project uses only conventions the Agent Skills spec already defines:
- `scripts/` directory for executable code ([spec](https://agentskills.io/skill-creation/using-scripts))
- `references/` directory for on-demand content ([spec](https://agentskills.io/specification#references))
- Frontmatter extensibility for custom fields ([spec](https://agentskills.io/specification#metadata-field))

The `triggers:` field is skill-specific metadata using the spec's existing extensibility. Agents that don't understand `triggers:` simply ignore it.

## The Four-Tier Model

The Agent Skills spec defines three tiers. This project documents a fourth — **Tier 0 (Discovery)** — that sits outside the skill in project config files (`CLAUDE.md`, `.cursorrules`, etc.):

| Tier | What | When loaded |
|------|------|-------------|
| 0 — Discovery | Skill index in project config | Session start (always in context) |
| 1 — Metadata | Name + description from frontmatter | Session start (catalog) |
| 2 — Instructions | Full SKILL.md body | Skill activation |
| 3 — Resources | Reference files via triggers | **Context injection** (this project) |

See [`docs/tier-0-discovery.md`](docs/tier-0-discovery.md) for the full Tier 0 pattern.

## Example

See [`examples/saw/`](examples/saw/) for the Scout-and-Wave skill's trigger configuration.

## License

MIT
