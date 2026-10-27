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
- Renovate bumps the patch version with every image update (helm-values manager, which
  detects every `image:` block, including the sidecars). For a manual change while the
  chart is 0.x: patch for fixes, minor for new values or breaking value changes (with an
  upgrade note in `README.md`); from 1.0 on, breaking value changes are major.
- Monthly Home Assistant releases are never automerged (Renovate counts them as minor);
  patch releases are. Review the skipped releases' backward-incompatible changes in the PR.

## Conventions to preserve

- Everything site-specific stays configurable through `values.yaml`; defaults must work
  in a plain cluster with only Home Assistant enabled.
- Optional features default to off.
- Every container image is an `image:` block with `repository`, `tag` and `pullPolicy`
  (top-level `image` for Home Assistant, `<sidecar>.image` for sidecars), pinned to an
  exact tag. Renovate's helm-values manager only detects keys ending in `image`, so never
  use flat `repository`/`tag` keys; no custom regex manager is needed.
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

## Pitfalls

- **Verify published artifacts without local registry logins.** Helm and Docker fall back to
  a Docker credential store (`credsStore` in `~/.docker/config.json`, e.g. Docker Desktop).
  A stale GHCR login there makes public packages fail with `403 denied`, and
  `--registry-config /dev/null` does not stop Helm from using it. Use an empty
  `DOCKER_CONFIG` directory plus an empty `--registry-config` file, or the anonymous token
  flow: `curl "https://ghcr.io/token?scope=repository:<owner>/<name>:pull&service=ghcr.io"`,
  then `GET https://ghcr.io/v2/<owner>/<name>/tags/list` with the bearer token.
- GHCR package visibility is a separate setting from repository visibility.
- `gh pr edit` can fail with "Projects (classic) is being deprecated". Use
  `gh api -X PATCH repos/<owner>/<repo>/pulls/<n> -f title=... -F body=@file.md`.
- This repository is public: never reference private consumer repositories (names, paths,
  hostnames, cluster details) in docs, AI files, comments, commit messages or PR texts.
- Home Assistant migrates its storage files when it starts a new monthly release, so a
  rollback to the previous image needs a backup taken before the upgrade. Mention this in
  the upgrade PR (see "Releases" for the review of skipped releases).
- The `python-matter-server` sidecar image is archived upstream (8.1.2 is the final
  release; the successor is the matter.js-based server). Home Assistant's Matter client
  stays compatible as long as both use the same schema version (11 as of 2026.10).

## Scope limit for agents

Only change what the task asks. Do not change released versions, the release flow or
workflow permissions as a side effect. When an industry best practice would materially
improve the chart, explain the alternative and its trade-offs before applying it.
