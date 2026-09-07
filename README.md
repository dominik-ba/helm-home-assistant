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

## Wrapping this chart

This chart is intentionally free of any site-specific configuration (fixed
hostnames, node pinning, secret manager integration, storage classes tied to
a particular NAS, etc.). To layer that on top, create a small wrapper chart
that depends on this one and:

- sets `nodeSelector`, `ingress.host`, `persistence.storageClassName`, and
  other environment-specific values in its own `values.yaml` under the
  `home-assistant:` key,
- creates whatever Secret/CRD your secret manager needs and points
  `home-assistant.matter.existingSecret` at it.

For a concrete example of this pattern (1Password integration, fixed node
placement, ingress host), see the `home-assistant-app` wrapper chart in
[dominik-ba/my-raspi-cluster](https://github.com/dominik-ba/my-raspi-cluster/tree/main/gitops/helm/home-assistant-app),
which consumes this chart as an OCI dependency.

## Releases

Tagging a commit `vX.Y.Z` (matching the version in `Chart.yaml`) packages the
chart and publishes it to `oci://ghcr.io/dominik-ba/charts/home-assistant` via
the `release` GitHub Actions workflow.
