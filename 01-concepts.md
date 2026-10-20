# 01 · Concepts

## What Helm is
Helm is a package manager for Kubernetes. Instead of maintaining many raw YAML manifests per environment, you write **templates** once and feed them **values**.

## Key terms

**Chart** — a package: a directory (or `.tgz`) of templates + default values + metadata.

**Release** — one installed instance of a chart in a cluster. The same chart can be installed many times under different release names (e.g. `redis-cache`, `redis-sessions`).

**Revision** — every `install`, `upgrade` or `rollback` creates a new numbered revision of a release. This is what makes rollbacks possible.

**Repository** — a place charts are published (an HTTP server with an `index.yaml`, or an OCI registry).

**Values** — configuration passed into templates (`values.yaml`, `-f file.yaml`, `--set key=val`).

## How it works

```
Chart templates + Values  ──render──▶  Kubernetes manifests  ──apply──▶  Cluster
                                                                      │
                                       Release record (Secret) ◀──────┘
```

- Helm 3+ has **no server-side component** (Tiller was removed in Helm 3). The CLI talks to the Kubernetes API directly using your kubeconfig.
- Release info is stored as **Secrets** in the release's namespace (type `helm.sh/release.v1`, named `sh.helm.release.v1.<release>.v<revision>`).
- Releases are **namespaced** — same release name can exist in different namespaces.

## Helm versions
- **Helm 2** — used Tiller (in-cluster server). Obsolete.
- **Helm 3** — no Tiller, 3-way strategic merge on upgrade, OCI support, JSON schema validation for values.
- **Helm 4** (released late 2025) — day-to-day commands and chart format stay largely the same.
