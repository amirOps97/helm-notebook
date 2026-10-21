# 07 · Dependencies & Hooks

## Dependencies (subcharts)
Declared in `Chart.yaml`:
```yaml
dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: https://charts.bitnami.com/bitnami   # or oci://... or file://../common
    condition: postgresql.enabled       # toggle via values
    alias: db                           # optional: rename
    tags: [database]
```
```bash
helm dependency update ./mychart   # fetch into charts/, write Chart.lock
helm dependency build  ./mychart   # reproduce from Chart.lock
helm dependency list   ./mychart
```

### Configuring a subchart
Put its values under a key matching the chart name (or alias) in the parent's `values.yaml`:
```yaml
postgresql:
  enabled: true
  auth:
    database: app
```

### Global values
Available to parent and all subcharts as `.Values.global.*`:
```yaml
global:
  imageRegistry: registry.example.com
```

### Notes
- A subchart **cannot** see its parent's values (except `global`).
- Parent **can** override subchart values.
- Commit `Chart.lock`; whether to commit `charts/*.tgz` is a team choice (usually not).

## Hooks
Run resources at specific points in a release's lifecycle, via annotations:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp:{{ .Chart.AppVersion }}
          command: ["./migrate"]
```

| Hook | When |
|------|------|
| `pre-install` / `post-install` | before/after resources are created on install |
| `pre-upgrade` / `post-upgrade` | before/after an upgrade |
| `pre-delete` / `post-delete` | before/after uninstall |
| `pre-rollback` / `post-rollback` | before/after rollback |
| `test` | run by `helm test` |

- **hook-weight**: lower runs first (string number, can be negative).
- **hook-delete-policy**: `before-hook-creation`, `hook-succeeded`, `hook-failed`.
- Hook resources are **not** managed as part of the release — they aren't removed on uninstall unless a delete policy says so.
- Helm waits for hook Jobs/Pods to complete; a failed hook fails the release.
