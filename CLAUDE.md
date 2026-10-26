# CLAUDE.md

Read **`AGENTS.md`** in the repository root first. It holds the repository purpose,
layout, commands, release flow, conventions and security notes. This file only adds
Claude Code operating rules and does not repeat that content.

## Operating rules for this repository

- Before reporting a change as done, run the checks in "Validation expectations" in
  `AGENTS.md` and report the actual output.
- Any change to what the chart renders needs a `version:` bump in `Chart.yaml`, because
  released versions are immutable. Say which bump you chose and why.
- Never change resource names, selector labels or PVC names without calling out the data
  impact on existing installations (e.g. node-bound local-path volumes).
- Do not weaken workflow permissions or switch CI to `pull_request_target`; fork PRs
  must stay without secrets and write access.
- No secrets belong in this repository.
