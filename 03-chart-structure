# 03 · Chart Structure

```
mychart/
├── Chart.yaml          # metadata (required)
├── Chart.lock          # locked dependency versions (generated)
├── values.yaml         # default values
├── values.schema.json  # optional JSON schema for values
├── charts/             # dependency charts (subcharts)
├── crds/               # CRDs — installed before templates, never templated or upgraded
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # files starting with _ are not rendered as manifests
│   ├── NOTES.txt       # printed after install/upgrade
│   └── tests/          # test pods (helm test)
├── .helmignore         # like .gitignore, for helm package
└── README.md
```

## Chart.yaml
```yaml
apiVersion: v2              # v2 = Helm 3+ charts
name: mychart
description: A demo app
type: application           # or "library" (only helpers, no manifests)
version: 0.1.0              # CHART version — bump on every chart change (SemVer)
appVersion: "1.16.0"        # version of the APP being deployed (informational)
kubeVersion: ">=1.25.0-0"   # optional constraint
keywords: [web, demo]
maintainers:
  - name: Your Name
    email: you@example.com
dependencies:
  - name: redis
    version: "~19.0.0"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

### `version` vs `appVersion`
- `version` → the chart's own version. Change anything in the chart → bump it.
- `appVersion` → the app/image version. Commonly used as the default image tag:
  `{{ .Values.image.tag | default .Chart.AppVersion }}`

## Chart types
- **application** — normal, deployable chart.
- **library** — reusable named templates only; can't be installed on its own. Used as a dependency.
