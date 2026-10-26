# GitHub Copilot instructions

Primary guidance for AI agents is in **`AGENTS.md`** at the repository root; read it for
the layout, commands, release flow, conventions and security notes. This file is a
short standalone summary for Copilot Chat, agent mode and Copilot CLI.

## Repository

Generic Helm chart for Home Assistant, published to
`oci://ghcr.io/dominik-ba/charts/home-assistant`. Only the Home Assistant container is on
by default; backup shipper, Matter hub, python-matter-server, ingress and persistence are
opt-in. Site-specific configuration belongs in a consumer's values file (or a wrapper
chart for extra resources), not here.

## Commands

- `helm lint . --strict`
- `helm template home-assistant . --namespace home-assistant` plus the all-sidecars
  scenario in `.github/workflows/ci.yaml`

## Rules

- A new `version:` in `Chart.yaml` on `main` is a release (`release.yaml`). Released
  versions are immutable, so every change to rendered output needs a version bump.
- Do not change resource names, selector labels or PVC names (the wrapper deployment
  keeps its data on local-path volumes).
- No secrets in the repo. Fork PRs run without secrets and with a read-only token: no
  `pull_request_target`, no write permissions or registry logins in PR jobs, actions
  pinned to commit SHAs.
- 2-space YAML indentation, `.editorconfig`, English for code and docs.
