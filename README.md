# Home Assistant

A generic, configurable Helm chart for deploying [Home Assistant](https://www.home-assistant.io/)
on Kubernetes. It has no dependency on any particular secret manager, cloud
provider, or cluster topology, so it can be reused as-is or wrapped by a
site-specific chart.

By default, only the Home Assistant container itself is enabled. Every
optional sidecar (backup shipper, Matter hub, Matter controller) and every
ingress/persistence knob defaults to off or empty, so `helm install` gives you
a minimal, working Home Assistant deployment that you then opt into extra
features from.

## Installation

```sh
helm install home-assistant oci://ghcr.io/dominik-ba/charts/home-assistant --version <version> -n home-assistant --create-namespace
```

Or from source:

```sh
git clone https://github.com/dominik-ba/helm-home-assistant.git
helm install home-assistant ./helm-home-assistant -n home-assistant --create-namespace
```

## Optional integrations

Install HACS from inside the container:

```sh
kubectl -n home-assistant exec -it statefulset/home-assistant -c home-assistant -- bash -c "wget -O - https://get.hacs.xyz | bash -"
```

## Key values

- `persistence.enabled`, `persistence.size`, and `persistence.storageClassName` control the PVC backing the Home Assistant config directory. Leave `storageClassName` empty to use your cluster's default storage class.
- `hostNetwork` is enabled by default because Home Assistant commonly relies on host-level networking for mDNS, SSDP, and multicast-based discovery on local networks. Set it to `false` if you don't need LAN device discovery.
- `ingress.enabled` is `false` by default. Set it to `true` and configure `ingress.className`, `ingress.host`, `ingress.annotations`, and `ingress.tls` to expose Home Assistant outside the cluster.
- `backupShipper.enabled` (default `false`) turns on a sidecar that mirrors Home Assistant's own `.tar` backup archives out of `/config/backups` into a second volume, controlled by `backupLocation.*`. Requires `backupLocation.enabled: true` as well. `backupPolicy.retention.keep` controls how many archives are kept at the destination.
- `matter.enabled` (default `false`) toggles the [home-assistant-matter-hub](https://github.com/t0bst4r/home-assistant-matter-hub) sidecar, which exposes Home Assistant entities as Matter devices to other ecosystems. It needs a long-term Home Assistant access token: set `matter.existingSecret` to the name of a Secret you create yourself (via 1Password Operator, External Secrets Operator, a plain `kubectl create secret`, etc.) with the token under the key named by `matter.existingSecretKey` (default `HAMH_HOME_ASSISTANT_ACCESS_TOKEN`). This chart does not create that Secret for you.
- `pythonMatterServer.enabled` (default `false`) toggles an additional [python-matter-server](https://github.com/matter-js/python-matter-server) sidecar — a Matter *controller* consumed by Home Assistant's built-in Matter integration, distinct from the matter-hub sidecar above. It relies on the pod's `hostNetwork: true` for mDNS/Thread and gets its own PVC (`pythonMatterServer.persistence.*`) for `/data`.

## Site-specific setup

This chart is intentionally free of any site-specific configuration (fixed
hostnames, node pinning, secret manager integration, storage classes tied to
a particular NAS, etc.). How you add yours depends on whether you only
configure the chart or also need extra Kubernetes resources.

### Configuration only: a values file

`nodeSelector`, `ingress.host`, `persistence.storageClassName`, sidecar
toggles, the name of an existing Secret for `matter.existingSecret`:
everything the chart exposes in `values.yaml` goes into your own values file.
No wrapper chart is needed.

```sh
helm install home-assistant oci://ghcr.io/dominik-ba/charts/home-assistant \
  --version <version> -n home-assistant --create-namespace -f my-values.yaml
```

With ArgoCD, keep the values file in your own Git repository and reference it
from a multi-source `Application`:

```yaml
spec:
  sources:
    - chart: home-assistant
      repoURL: ghcr.io/dominik-ba/charts
      targetRevision: <version>
      helm:
        valueFiles:
          - $values/home-assistant/values.yaml
    - repoURL: https://github.com/<you>/<your-config-repo>.git
      targetRevision: main
      ref: values
```

### Additional resources: a wrapper chart

If you need resources this chart does not render, such as a secret-manager
object (1Password `OnePasswordItem`, External Secrets `ExternalSecret`) that
creates the Secret for `matter.existingSecret`, static PVs, an extra ingress
or middleware, create a small wrapper chart that depends on this one and adds
them in its own `templates/`. Values for this chart then go under the
`home-assistant:` key of the wrapper's `values.yaml`.

```yaml
# Chart.yaml
dependencies:
  - name: home-assistant
    version: "<version>"
    repository: "oci://ghcr.io/dominik-ba/charts"
```

Commit `Chart.lock` and let `helm dependency build` (or ArgoCD) fetch the
chart; don't commit the downloaded `charts/*.tgz`.

## Releases

Releases are driven by the `version:` in `Chart.yaml`. When `main` holds a
version that has not been released yet, the `release` workflow publishes it to
`oci://ghcr.io/dominik-ba/charts/home-assistant`, tags the commit `vX.Y.Z` and
creates a GitHub Release. To release, bump the version in a PR and merge it.

Renovate bumps the patch version together with every image update, so image
updates are released without manual steps.

Released versions are immutable: the workflow never re-pushes a version that
already has a tag or an OCI artifact.

## Pull requests from forks

CI runs on PRs from forks with a read-only token and without secrets.
Workflows from first-time and outside contributors need approval before they
run.
