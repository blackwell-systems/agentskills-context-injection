# Proposal Draft — agentskills/agentskills Issue

**Title:** `Proposal: triggers: frontmatter field for deterministic Resources tier loading`

---

## Problem

The spec defines a Resources tier for on-demand reference files and recommends
keeping SKILL.md under 500 lines by moving detailed content to references/.
But loading is convention-based — the model decides when to read them.

For simple skills this works. For skills with multiple subcommand families
and large reference files, it produces an impossible tradeoff:

- Load all references upfront: wastes context budget on every invocation.
  A skill with 3 reference files (~430 lines each) loads ~40% of its
  effective context budget on content irrelevant to the current subcommand.
- Rely on the model to load selectively: misses files, improvises on logic
  it doesn't have.

There is no mechanism for skill authors to declare deterministically which
reference files are needed for which invocations.

## Convergent evidence

Every major Agent Skills-compatible platform has independently built a
pre-invocation hook that fires before the model processes the user's prompt:

| Platform     | Hook                 |
|--------------|----------------------|
| Claude Code  | `UserPromptSubmit`   |
| Gemini CLI   | `BeforeAgent`        |
| OpenAI Codex | `UserPromptSubmit`   |
| Cursor       | `beforeSubmitPrompt` |
| OpenCode     | `chat.message`       |

The ecosystem has converged on this pattern. What is missing is a standard
declaration on the skill side — so authors write intent once and all
conforming platforms honor it.

## Proposal

Add `triggers:` as a recognized top-level frontmatter field:

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

**Field semantics:**
- `match`: regex tested against the user's prompt at invocation time
- `inject`: path to a Resources tier file, relative to the skill root
- Multiple triggers may match; all matching files are injected
- No match → no injection, zero overhead

**Platform behavior:** When a user invokes a skill, the platform fires its
pre-invocation hook, reads the skill's `triggers:` field, tests each pattern
against the prompt, and prepends matching reference file contents to the
model's context before the turn begins. Platforms without a pre-invocation
hook degrade gracefully to model-directed loading (existing behavior).

## Why top-level, not under metadata:

`metadata:` is defined as a map of string key-value pairs — passive
author-defined data passed through to the model as context. `triggers:`
is structurally different: it is a list of objects, and it is consumed
by the platform hook before the model runs, not by the model itself.
Nesting it under `metadata:` would misrepresent its semantics and abuse
the string-values constraint. A new top-level field correctly signals
that `triggers:` is platform-consumed operational data.

## Scope

Dispatch-time subcommand triggers only. The hook fires before the model
runs, so `triggers:` can only act on information available at invocation
time — the user's prompt. Mid-execution references that depend on runtime
state (agent outputs, failure conditions) are explicitly out of scope and
should continue to use convention-based loading.

## Portable format constraint

To ensure any platform can implement a conforming parser without a YAML
library dependency, the `triggers:` field is restricted to a portable
subset of YAML:

- Flat list of `{match, inject}` entries only
- Single-line scalar values (no multi-line strings, no block scalars)
- No YAML anchors, aliases, or tags
- `match` and `inject` on consecutive lines within each list item

Full YAML parsers handle this subset correctly. Lightweight parsers
(awk, line-oriented) can also conform. The constraint ensures both
approaches produce identical results.

## Backward compatibility

Platforms that do not implement `triggers:` ignore the field — standard
YAML behavior for unknown keys. No existing skills are affected.

## Reference implementation

https://github.com/blackwell-systems/agentskills-subcommand-dispatch

Includes:
- `scripts/inject-context` — portable bash script, reads `triggers:` and
  injects matching references. Works on any platform via model instruction.
- `hooks/inject_skill_context` — Claude Code `UserPromptSubmit` hook.
  Iterates all installed skills, delegates to each skill's inject-context.
  Zero changes required when adding new skills.
- `scripts/validate-triggers` — checks trigger patterns against the skill
  body to detect false-positive risk before deployment.
- `examples/saw/` — complete example from the Scout-and-Wave skill.

The implementation has been in production use in the Scout-and-Wave
protocol (https://github.com/blackwell-systems/scout-and-wave) across
16+ repositories.
