# AGENTS.md

Tool-neutral guidance for AI coding agents (GitHub Copilot, Claude Code) working in
this repository. `CLAUDE.md` and `.github/copilot-instructions.md` reference this file;
keep repository facts here and avoid duplicating them elsewhere.

## What this repository is

A generic, reusable Helm chart for [Home Assistant](https://www.home-assistant.io/),
published as an OCI artifact to `oci://ghcr.io/dominik-ba/charts/home-assistant`. The
repo is public; anyone can open pull requests from forks.

By default only the Home Assistant container is enabled. Optional sidecars (backup
shipper, Matter hub, python-matter-server), ingress and persistence are opt-in. The
chart has no site-specific configuration. Consumers set node placement, ingress host and
storage classes in their own values file and only need a wrapper chart when they add
resources this chart does not render, e.g. secret-manager objects (see "Site-specific
setup" in `README.md`). Site-specific values belong there, not here.

## Layout

- `Chart.yaml`: chart metadata; `version` drives releases. No subchart dependencies.
- `values.yaml`: all configurable defaults. The Home Assistant image is under `image:`;
  the sidecar images are flat `repository`/`tag` keys under `backupShipper`, `matter`
  and `pythonMatterServer`.
- `templates/`: StatefulSet (Home Assistant + sidecars), Service, Ingress, PVCs,
  ServiceAccount, backup-shipper ConfigMap, helpers.
- `.github/workflows/ci.yaml`: PR and main CI (lint, default and all-sidecars renders,
  kubeconform).
- `.github/workflows/release.yaml`: publishes an unreleased `Chart.yaml` version from `main`.
- `renovate.json`: image updates, chart version bump, action digest pinning.

## Commands

| Purpose | Command |
| --- | --- |
| Lint | `helm lint . --strict` |
| Render defaults | `helm template home-assistant . --namespace home-assistant` |
| Render all sidecars | see the "all sidecars enabled" step in `.github/workflows/ci.yaml` |
| Validate manifests | `kubeconform -strict -summary -ignore-missing-schemas <rendered.yaml>` |

CI uses Helm v4 (`azure/setup-helm`, version in the workflows).

## Validation expectations

Before proposing a change as complete, run and report:

1. `helm lint . --strict`.
2. Both `helm template` scenarios from `ci.yaml` and kubeconform on each output if available.
3. For changes to templates or defaults: a diff of the rendered output before and after
   the change, and a note whether it changes what the `home-assistant-app` wrapper deploys
   (it renders the StatefulSet `home-assistant` with local-path PVCs on one node;
   resource names, selector labels and volumeClaim names must not change).

## Releases

- A release is a new `version:` in `Chart.yaml` merged to `main`. `release.yaml` then
  pushes `ghcr.io/dominik-ba/charts/home-assistant:<version>`, tags `v<version>` and
  creates a GitHub Release.
- Released versions are immutable. The workflow never re-pushes a version that already
  has a tag or an OCI artifact. Any change to what the chart renders needs a version bump.
- Renovate bumps the patch version with every image update detected by its helm-values
  manager. For a manual change: patch for fixes, minor for new values, major for
  breaking value changes.

## Conventions to preserve

- Everything site-specific stays configurable through `values.yaml`; defaults must work
  in a plain cluster with only Home Assistant enabled.
- Optional features default to off.
- `hostNetwork: true` is the default on purpose (mDNS/SSDP discovery).
- The chart never creates the Matter hub token Secret; it only references
  `matter.existingSecret`.
- 2-space indentation for YAML and templates; honor `.editorconfig`.
- English for code, comments and docs.

## Security notes

- No secrets in the repository.
- Fork PRs run CI with a read-only `GITHUB_TOKEN` and no secrets. Do not use
  `pull_request_target`, do not add `workflow_run` jobs that consume PR artifacts, and do
  not give PR jobs write permissions or registry logins. Write permissions belong only to
  the release job on `main`.
- Third-party actions are pinned to commit SHAs (Renovate keeps them current). Keep new
  actions pinned the same way.

## Known gaps

- Renovate's helm-values manager only detects the `image:` block. The flat sidecar image
  keys (`backupShipper`, `matter`, `pythonMatterServer`) are not tracked, so those tags
  are only updated by hand. Fixing this needs either a values restructure (breaking for
  wrappers) or a custom regex manager; raise it as a proposal.

## Scope limit for agents

Only change what the task asks. Do not change released versions, the release flow or
workflow permissions as a side effect. When an industry best practice would materially
improve the chart, explain the alternative and its trade-offs before applying it.
