---
name: saw
description: "Parallel agent coordination: Scout analyzes the codebase and produces a plan; Wave agents implement in parallel."
triggers:
  - match: "^/saw program"
    inject: references/program-flow.md
  - match: "^/saw amend"
    inject: references/amend-flow.md
  # failure-routing.md is intentionally NOT triggered here. It's a mid-execution
  # reference loaded after agents report back, not at dispatch time. The skill
  # body contains "failure", "blocked", etc. which would false-positive on every
  # invocation since UserPromptSubmit receives the expanded prompt.
  # Verify with: bash scripts/validate-triggers
---

# Scout-and-Wave: Parallel Agent Coordination

This is an example frontmatter showing how SAW declares triggers for subcommand dispatch routing.

The `triggers:` block tells `scripts/inject-context` which reference files to load based on the user's prompt. For example:

- `/saw program execute "add caching"` matches `^/saw program` and injects `references/program-flow.md`
- `/saw amend --add-wave` matches `^/saw amend` and injects `references/amend-flow.md`

Prompts that don't match any trigger (e.g., `/saw wave`, `/saw scout`) inject nothing -- zero overhead.

Note: `failure-routing.md` is intentionally not triggered here. It's a mid-execution reference
loaded after agents report back, not at dispatch time. Broad keyword triggers (like "failure" or
"blocked") false-positive when the hook receives the expanded skill body, which contains those
words in its own instructions. Only dispatch-time references (subcommand routing) should use triggers.
