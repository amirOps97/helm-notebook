# 08 · Repositories & OCI

## Packaging
```bash
helm package ./mychart                     # mychart-0.1.0.tgz
helm package ./mychart --version 0.2.0 --app-version 1.4.0
```

## Classic HTTP chart repository
A repo is just static files: `.tgz` packages + an `index.yaml`.
```bash
helm package ./mychart -d ./repo
helm repo index ./repo --url https://example.github.io/charts
```
Host on GitHub Pages, S3, any web server. The `chart-releaser` GitHub Action automates this for GitHub Pages.

## OCI registries (recommended today)
Charts can be stored in OCI registries like container images (GHCR, Docker Hub, ECR, ACR, Harbor…).
```bash
helm registry login ghcr.io -u <user>
helm push mychart-0.1.0.tgz oci://ghcr.io/<user>/charts

helm pull     oci://ghcr.io/<user>/charts/mychart --version 0.1.0
helm install  app oci://ghcr.io/<user>/charts/mychart --version 0.1.0
helm show values oci://ghcr.io/<user>/charts/mychart
```
- No `helm repo add` needed for OCI — reference the chart directly.
- Chart name and version come from `Chart.yaml`; the tag is the chart version.

## Finding charts
- [Artifact Hub](https://artifacthub.io) — central search for public charts.
- `helm search hub <keyword>`
