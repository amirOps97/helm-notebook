# 09 · Best Practices

## Charts
- Bump `version` in `Chart.yaml` on every chart change (SemVer).
- Use the standard labels:
  `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/version`, `app.kubernetes.io/managed-by`, `helm.sh/chart`.
- Keep selector labels **stable** — changing a Deployment's `selector` breaks upgrades.
- Truncate generated names to 63 chars (`trunc 63 | trimSuffix "-"`).
- Prefix named templates with the chart name (`mychart.fullname`) to avoid clashes with subcharts.
- Put reusable bits in `_helpers.tpl`; prefer `include` over `template`.
- Use `required` for values that have no sane default.
- Provide `values.schema.json` for anything non-trivial.
- Document every value in `values.yaml`.

## Values & secrets
- Never commit real secrets in values files. Options: external secret managers (External Secrets Operator), Sealed Secrets, or the `helm-secrets` plugin with SOPS.
- Keep environment files small — only differences from defaults.

## Deploying
- Use `helm upgrade --install --atomic --wait` in CI/CD.
- Pin chart versions (`--version`) — never deploy "latest" by accident.
- Run `helm diff` before upgrading in production.
- Set resource requests/limits and probes through values.
- Consider GitOps tools (Argo CD, Flux) for declarative Helm deployments.

## Gotchas
- CRDs in `crds/` are installed once and **never upgraded or deleted** by Helm.
- `lookup` returns empty during `helm template` and `--dry-run` (client-side).
- Hook resources aren't part of the release; set a delete policy.
- `--reuse-values` ignores new defaults from upgraded charts.
- YAML type coercion: `yes`, `no`, `on`, `off`, `1.10` — quote when in doubt.
