# agentskills-subcommand-dispatch

[![Blackwell Systems™](https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg)](https://github.com/blackwell-systems)
[![Agent Skills](assets/badge-agentskills.svg)](https://agentskills.io)

The [Agent Skills](https://agentskills.io) spec defines a Resources tier for on-demand reference files but leaves loading to convention — the model decides when to load them. In practice, skills with multiple subcommand families face an impossible tradeoff: load all references upfront (wastes context budget on every invocation) or rely on the model to load selectively (misses files, improvises on logic it doesn't have).

This repo fixes that for dispatch-time subcommand routing. Skills declare which reference files to load for which subcommands. A pre-invocation hook loads them before the model starts — no model judgment required.

**Scope: dispatch-time subcommand triggers only.** If you know at invocation time which subcommand is being called, this handles it deterministically. Mid-execution references that depend on runtime state (e.g. failure routing after agents report back) are out of scope — the hook fires before execution begins.

> Uses existing Agent Skills conventions (`scripts/`, `references/`, `metadata:`). No spec changes required. Reference implementation for Claude Code (`UserPromptSubmit`) and Gemini CLI (`beforeAgent`).

## How it works

Skills add `triggers:` to their YAML frontmatter. Each trigger maps a regex to a reference file. When the user's prompt matches, the reference loads automatically.

Two layers, same trigger definitions:

**Layer 1 -- Script (any agent).** A `scripts/inject-context` script ships with the skill. The model runs it before executing:

```bash
bash scripts/inject-context "/saw program execute add caching"
# outputs contents of references/program-flow.md
```

One line in `SKILL.md`: "run `scripts/inject-context` with the user's prompt before proceeding." The model still decides what string to pass — this is model-initiated, not enforced. But it is simpler than a routing table and harder to accidentally skip.

**Layer 2 -- Hook (Claude Code).** A `UserPromptSubmit` hook runs the same script before the model sees the prompt. No model decision. The reference is in context when the model starts. Deterministic.

## Trigger Definitions

Skills declare triggers in YAML frontmatter using the `triggers:` field (via the spec's `metadata:` extension point):

```yaml
---
name: my-skill
description: Does things
triggers:
  - match: "^/my-skill subcommand"
    inject: references/subcommand-flow.md
  - match: "^/my-skill other-subcommand"
    inject: references/other-flow.md
---
```

- `match`: regex pattern tested against the full prompt text
- `inject`: path relative to the skill directory
- Multiple matches -> all matching references injected (concatenated)
- No match -> no injection, zero overhead

**Subcommand-anchored patterns only.** Pre-invocation hooks (`UserPromptSubmit` in Claude Code, `beforeAgent` in Gemini CLI) fire after skill body expansion — the full `SKILL.md` content is in the prompt, not just what the user typed. Keyword triggers like `failure|blocked` match against the skill's own instructions and fire on every invocation. Use patterns anchored to the invocation prefix (e.g. `^/saw program`, `^/saw amend`) that cannot appear in the skill body. Mid-execution references that depend on runtime state (failure routing, post-merge integration) should stay convention-based — hooks fire too early for them.

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

| Layer | Mechanism | Platform | Enforcement |
|-------|-----------|----------|-------------|
| Hook | `UserPromptSubmit` | Claude Code | Deterministic (pre-model) |
| Hook | `beforeAgent` | Gemini CLI | Deterministic (pre-model) |
| Script | `scripts/inject-context` | Any agent with Bash | Model-initiated |
| Fallback | Routing table in SKILL.md | Any agent | Convention-based |

Users get the best available layer. No regression at any level.

## Spec Alignment

This project uses only conventions the Agent Skills spec already defines:
- `scripts/` directory for executable code ([spec](https://agentskills.io/skill-creation/using-scripts))
- `references/` directory for on-demand content ([spec](https://agentskills.io/specification#references))
- Frontmatter extensibility for custom fields ([spec](https://agentskills.io/specification#metadata-field))

The `triggers:` field is declared at the top level of the skill frontmatter rather than nested under `metadata:`. Top-level placement signals that it is intended for platform consumption — the hook reads it before the model runs — not just passive skill metadata. Agents that don't understand `triggers:` ignore it with no behavior change.

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
