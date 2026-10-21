# Helm Patterns

A practical reference for designing reusable, configurable, and reliable Helm charts.


## 1. The Most Useful Helm Patterns

### Pattern 1 — Restart Pods When a ConfigMap Changes

**Problem:** Updating a ConfigMap does not necessarily restart pods. An application that reads configuration only at startup may continue using its old configuration.

**Solution:** Add a checksum annotation to the Deployment's **Pod template**.

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

When the rendered ConfigMap changes during a Helm upgrade, the checksum changes. Kubernetes detects the changed Pod template and rolls out new pods.

**Important:** Put the annotation under `spec.template.metadata.annotations`, not the Deployment's top-level `metadata.annotations`.

This is useful for applications that read configuration at startup.

### Pattern 2 — Let Users Define Their Own Environment Variables

**Problem:** Chart users should not need to edit `deployment.yaml` every time they need another environment variable.

In `values.yaml`:

```yaml
env: {}
```

In `templates/deployment.yaml`:

```yaml
{{- with .Values.env }}
env:
  {{- range $key, $val := . }}
  - name: {{ $key }}
    value: {{ $val | quote }}
  {{- end }}
{{- end }}
```

Users can configure:

```yaml
env:
  LOG_LEVEL: debug
  APP_MODE: production
```

Or pass values on the command line:

```bash
helm upgrade --install myapp ./mychart \
  --set env.LOG_LEVEL=debug \
  --set env.APP_MODE=production
```

**Why it is useful:** The same chart can be reused across development, staging, and production without changing its templates.

**Security note:** Do not pass passwords or other sensitive values through `--set`. Use Kubernetes Secrets or an external secret manager.

### Pattern 3 — Use Reusable Named Templates

**Problem:** Names, labels, or other YAML fragments are repeated across multiple templates.

Create a helper in `templates/_helpers.tpl`:

```gotemplate
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

Reuse it in a resource:

```yaml
metadata:
  labels:
    {{- include "mychart.selectorLabels" . | nindent 4 }}
```

**Why it is useful:** Define the logic once and reuse it across Deployments, Services, and other resources.

Use chart-prefixed helper names such as `mychart.fullname`, rather than generic names such as `fullname`. Named templates are globally accessible across charts and subcharts, so namespacing helps avoid collisions.

### Pattern 4 — Make Optional Resources Configurable

**Problem:** Some users need an Ingress, while others expose the application through a different mechanism.

In `values.yaml`:

```yaml
ingress:
  enabled: false
```

In `templates/ingress.yaml`:

```gotemplate
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  # Add the required Ingress configuration here.
{{- end }}
```

When `enabled` is `false`, Helm does not render the Ingress resource.

The same pattern can be used for optional autoscaling, ServiceAccounts, NetworkPolicies, and monitoring resources.

### Pattern 5 — Set Safe Defaults and Validate Required Values

**Problem:** A user forgets a required value, and the chart generates a broken manifest or fails later.

For a required image repository:

```gotemplate
image: "{{ required "image.repository is required" .Values.image.repository }}:{{ .Values.image.tag }}"
```

If the repository is missing or empty, Helm fails with a meaningful error.

For an optional setting:

```gotemplate
replicas: {{ .Values.replicaCount | default 2 }}
```

**Subtlety:** Helm's `default` treats values such as `0`, `false`, and `""` as empty. Do not blindly use it when `0` or `false` is an intentional value.

For stronger validation, use `values.schema.json` to define required fields, types, and allowed values.

### Pattern 6 — Use `toYaml` and `nindent` for Complex Configuration

**Problem:** Users need to configure nested Kubernetes settings without manually templating every field.

In `values.yaml`:

```yaml
nodeSelector:
  kubernetes.io/os: linux

tolerations: []
```

In the Deployment template:

```gotemplate
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

What these functions do:

- `toYaml` converts a map or list into YAML.
- `nindent 8` adds a newline and indents the rendered content by eight spaces.
- `with` changes the current context to the supplied value and skips the block when the value is empty.

This is especially useful for `nodeSelector`, `affinity`, `tolerations`, and user-supplied pod annotations.

**Important:** The indentation passed to `nindent` depends on where the output belongs. Also, `toYaml` alone cannot turn a map such as `env: { LOG_LEVEL: info }` into Kubernetes' required list-of-objects environment-variable format; use an explicit `range` for that.

---

## 2. Patterns for Production-Ready Workloads

### Pattern 7 — Use Explicit Image Tags or Digests

Avoid floating tags such as:

```yaml
image: nginx:latest
```

Prefer a fixed version:

```yaml
image:
  repository: nginx
  tag: "1.28.0"
```

Or pin the image by digest:

```yaml
image: nginx@sha256:YOUR_ACTUAL_IMAGE_DIGEST
```

A fixed tag is easier to reason about than a floating tag, although registries can allow tags to move. A digest identifies the exact image content.

**Why it is useful:** Makes deployments more reproducible and rollbacks more predictable.

### Pattern 8 — Make Health Probes Configurable

Kubernetes supports three important probes:

- **Startup probe:** Has the application finished starting?
- **Readiness probe:** Can the application receive traffic?
- **Liveness probe:** Is the application stuck and in need of a restart?

Example:

```yaml
startupProbe:
  httpGet:
    path: /health
    port: http
  failureThreshold: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: http

livenessProbe:
  httpGet:
    path: /health
    port: http
```

In a reusable chart, expose probe settings through values so users can adapt paths, ports, and timing to their applications.

**Why it is useful:** Kubernetes can distinguish an application that is still starting from one that is ready or unhealthy. A poorly configured liveness probe can cause unnecessary restarts.

### Pattern 9 — Configure CPU and Memory Requests and Limits

In `values.yaml`:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

In the Deployment template:

```gotemplate
resources:
  {{- toYaml .Values.resources | nindent 12 }}
```

**Why it is useful:**

- Requests help Kubernetes schedule pods.
- Limits constrain resource consumption.

Treat these values as application-specific examples, not universal recommendations. Memory limits that are too low can cause OOM kills; CPU limits that are too restrictive can throttle performance.

### Pattern 10 — Keep Selector Labels Stable

A Deployment commonly uses this pattern:

```yaml
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
      app.kubernetes.io/instance: production
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myapp
        app.kubernetes.io/instance: production
        app.kubernetes.io/version: "2.1.0"
```

The selector labels must match the Pod template labels.

Notice that the application version is not part of the selector. Version labels may change between releases, while a Deployment's selector is generally immutable after creation.

**Why it is useful:** Helps prevent Deployment updates from breaking when the application version changes.

### Pattern 11 — Use Configurable Security Settings with Secure Defaults

For example:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

These settings can be rendered into the container's `securityContext`.

**Why it is useful:** Reduces unnecessary privileges.

However, test these settings against the container. Applications can have requirements involving filesystem permissions, ports, or Linux capabilities.

Also consider:

- Pod-level security settings.
- Read-only root filesystems where supported.
- Least-privilege RBAC.

---

## 3. Configuration, Secrets, and Operations

### Pattern 12 — Separate Configuration from Secrets

Keep ordinary application configuration in a ConfigMap:

```yaml
data:
  LOG_LEVEL: "info"
```

Keep sensitive values in a Kubernetes Secret or use an external secret manager.

A chart can let users specify an existing Secret:

```yaml
env:
  LOG_LEVEL: info

existingSecret: myapp-secrets
```

Then reference the Secret from the Deployment:

```yaml
envFrom:
  - secretRef:
      name: {{ .Values.existingSecret | quote }}
```

**Why it is useful:** Users can manage secret values separately from the chart.

**Security notes:**

- Kubernetes Secrets are not automatically encrypted in every environment. Configure suitable encryption and access controls.
- If a Secret changes and the application only reads values at startup, a rollout mechanism may also be needed.
- Do not include secret contents in checksum annotations; this exposes a hash of sensitive data in Pod metadata.

### Pattern 13 — Support Extra Volumes and Mounts

A common requirement is mounting a certificate, custom configuration, or persistent data without rewriting the chart.

In `values.yaml`:

```yaml
extraVolumes: []
extraVolumeMounts: []
```

In the Deployment's Pod specification:

```gotemplate
{{- with .Values.extraVolumes }}
volumes:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

Inside the container definition:

```gotemplate
{{- with .Values.extraVolumeMounts }}
volumeMounts:
  {{- toYaml . | nindent 12 }}
{{- end }}
```

**Why it is useful:** Makes the chart extensible without requiring a dedicated value for every possible mount.

The volume and mount definitions must correspond to each other. Adjust indentation to match the surrounding Deployment structure.

### Pattern 14 — Make the ServiceAccount Configurable

Instead of hardcoding a ServiceAccount, let users configure it:

```yaml
serviceAccount:
  create: true
  name: ""
```

A helper can determine the effective name:

```gotemplate
{{- define "mychart.serviceAccountName" -}}
{{- if .Values.serviceAccount.name -}}
{{- .Values.serviceAccount.name -}}
{{- else -}}
{{- include "mychart.fullname" . -}}
{{- end -}}
{{- end -}}
```

Then use it in the Pod specification:

```yaml
serviceAccountName: {{ include "mychart.serviceAccountName" . }}
```

**Why it is useful:** The chart can create a dedicated ServiceAccount or use one managed outside the chart.

Combine this with least-privilege RBAC. If the application does not need Kubernetes API access, consider disabling automatic ServiceAccount token mounting.

### Pattern 15 — Make Autoscaling Optional

Let users choose between a fixed replica count and a HorizontalPodAutoscaler (HPA).

In `values.yaml`:

```yaml
replicaCount: 2

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

In the Deployment template, render `replicas` only when autoscaling is disabled:

```gotemplate
{{- if not .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```

Render the HPA in a separate template when autoscaling is enabled.

**Why it is useful:** Avoids having Helm continuously set a fixed replica count that conflicts with the HPA's scaling decisions.

CPU-based autoscaling usually requires resource requests and a functioning metrics pipeline.

### Pattern 16 — Use a Schema to Validate User Configuration

Create `values.schema.json` to specify what your chart accepts. For example, you can require an image repository and restrict a replica count to a valid range.

Simplified example:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": {
          "type": "string",
          "minLength": 1
        }
      }
    }
  },
  "required": ["image"]
}
```

**Why it is useful:** Users receive understandable validation errors instead of discovering configuration mistakes after the workload fails.

This is especially valuable when multiple teams consume the same chart.

---

## 4. Reliable Chart Development

### Pattern 17 — Test Rendered Output Before Deploying

Do not rely only on a successful template render. Use several checks:

```bash
# Check chart structure and conventions
helm lint ./mychart

# Render manifests locally
helm template myapp ./mychart

# Render using production-like values
helm template myapp ./mychart -f values-prod.yaml

# Save rendered output
helm template myapp ./mychart -f values-prod.yaml > rendered.yaml

# Validate against the target cluster
kubectl apply --dry-run=server -f rendered.yaml
```

Server-side dry-run requires access to the target cluster and compatible API resources. It validates submitted resources but does not guarantee that the application will work at runtime.

**Why it is useful:** Catches template errors, missing values, invalid Kubernetes fields, and many configuration mistakes before deployment.

### Pattern 18 — Use `NOTES.txt` to Guide the User

Create `templates/NOTES.txt`:

```text
Your application has been deployed.

Check the workload status:
  kubectl get pods -n {{ .Release.Namespace }}

Check the Service:
  kubectl get svc -n {{ .Release.Namespace }}
```

Helm displays this output after installation or upgrade.

**Why it is useful:** Users get useful next steps directly in the terminal instead of having to search through documentation.

### Pattern 19 — Use Subcharts and Dependencies for Reusable Components

If an application requires components such as Redis or PostgreSQL, declare chart dependencies in `Chart.yaml` rather than embedding all their templates in your own chart.

Conceptual example:

```yaml
dependencies:
  - name: redis
    version: "A_COMPATIBLE_VERSION"
    repository: "https://charts.example.com"
    condition: redis.enabled
```

The example repository and version are placeholders.

**Why it is useful:** Keeps the chart modular and allows independently maintained components to be reused.

Do not automatically bundle every dependency. In production, a managed database or an independently operated service may be a better choice.

### Pattern 20 — Use Helm Hooks Only When Lifecycle Ordering Is Necessary

Helm hooks can run Jobs at specific release lifecycle points, such as before an upgrade or after installation.

For example, a pre-upgrade Job might perform a database migration.

**Why hooks can be useful:** Some operations must happen at a particular point in a deployment.

**Cautions:**

- Hooks have special lifecycle and cleanup behavior.
- A failed migration or a hook Job that never finishes can block an upgrade.
- Database migrations should tolerate rolling deployments and application-version compatibility.
- Do not use hooks for ordinary resources that Helm can manage directly.

---

## 5. Quick-Reference Cheat Sheet

| Priority | Pattern | Main benefit |
|---:|---|---|
| 1 | ConfigMap checksum | Roll out pods when configuration changes |
| 2 | Configurable environment variables | Customize applications without editing templates |
| 3 | Named templates in `_helpers.tpl` | Reuse naming and labels |
| 4 | `with`, `if`, and `range` | Render optional and repeated configuration |
| 5 | `toYaml` + `nindent` | Handle complex YAML values cleanly |
| 6 | `required` + `values.schema.json` | Validate user input |
| 7 | Fixed image tags or digests | Make releases reproducible |
| 8 | Health probes | Manage startup, readiness, and failures |
| 9 | Resource requests and limits | Control scheduling and resource consumption |
| 10 | Stable selector labels | Avoid breaking workload updates |
| 11 | Secrets and least-privilege RBAC | Improve security |
| 12 | Optional resources and autoscaling | Support different environments |
| 13 | Extra volumes and mounts | Extend charts without editing templates |
| 14 | `helm lint` and rendered-manifest tests | Catch mistakes before deployment |
| 15 | Subcharts, hooks, and `NOTES.txt` | Organize and operate more complex charts |

For underlying conventions, refer to the official [Helm Chart Best Practices Guide](https://helm.sh/docs/chart_best_practices/).

---
