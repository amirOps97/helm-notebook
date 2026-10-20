# 04 · Templating

Helm uses Go's `text/template` plus the **Sprig** function library and a few Helm-specific functions.

---

## Built-in objects

| Object | Examples |
|--------|----------|
| `.Values` | `.Values.image.repository` |
| `.Release` | `.Release.Name`, `.Release.Namespace`, `.Release.Revision`, `.Release.IsInstall`, `.Release.IsUpgrade`, `.Release.Service` |
| `.Chart` | `.Chart.Name`, `.Chart.Version`, `.Chart.AppVersion` |
| `.Capabilities` | `.Capabilities.KubeVersion`, `.Capabilities.APIVersions.Has "networking.k8s.io/v1"` |
| `.Template` | `.Template.Name`, `.Template.BasePath` |
| `.Files` | `.Files.Get "config.ini"`, `.Files.Glob "conf/*"` |

### `.Values` — what's in values.yaml
values.yaml
```yaml
image:
  repository: nginx
```
template
```yaml
image: {{ .Values.image.repository }}
```
output
```yaml
image: nginx
```

### `.Release` — info about this install
template
```yaml
release:    {{ .Release.Name }}
namespace:  {{ .Release.Namespace }}
revision:   {{ .Release.Revision }}
isInstall:  {{ .Release.IsInstall }}
isUpgrade:  {{ .Release.IsUpgrade }}
managedBy:  {{ .Release.Service }}
```
output (first install)
```yaml
release:    demo
namespace:  dev
revision:   1
isInstall:  true
isUpgrade:  false
managedBy:  Helm
```

### `.Chart` — fields from Chart.yaml
template
```yaml
chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app:   {{ .Chart.AppVersion }}
```
output
```yaml
chart: mychart-0.1.0
app:   1.27.0
```

### `.Capabilities` — what the cluster supports
template
```yaml
kube: {{ .Capabilities.KubeVersion.Version }}
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1" }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: networking.k8s.io/v1beta1
{{- end }}
```
output
```yaml
kube: v1.30.0
apiVersion: networking.k8s.io/v1
```

### `.Template` — the current template file
template (in `templates/info.yaml`)
```yaml
file: {{ .Template.Name }}
dir:  {{ .Template.BasePath }}
```
output
```yaml
file: mychart/templates/info.yaml
dir:  mychart/templates
```

### `.Files` — non-template files in the chart
chart file `mychart/config.ini`
```ini
port=8080
```
template
```yaml
config.ini: {{ .Files.Get "config.ini" | quote }}
```
output
```yaml
config.ini: "port=8080\n"
```

---

## Syntax basics

```yaml
{{ .Values.name }}            # output a value
{{- .Values.name }}           # trim whitespace on the left
{{ .Values.name -}}           # trim whitespace on the right
{{/* a comment */}}
```

### Whitespace trimming
template **without** `-`
```yaml
labels:
  app: web
  {{ if true }}
  tier: frontend
  {{ end }}
```
output — ugly blank lines
```yaml
labels:
  app: web
  
  tier: frontend
  
```
template **with** `{{-`
```yaml
labels:
  app: web
  {{- if true }}
  tier: frontend
  {{- end }}
```
output
```yaml
labels:
  app: web
  tier: frontend
```
> `{{-` eats the newline and spaces **before** the tag. `-}}` eats them **after**. Using `{{-` on its own line is the safe habit.

### Comments
template
```yaml
{{/* This text never shows up in the output */}}
# This YAML comment DOES show up in the output
replicas: 1
```
output
```yaml
# This YAML comment DOES show up in the output
replicas: 1
```

---

## Pipelines and functions

`|` passes the result on the left as the **last** argument of the function on the right.

values.yaml
```yaml
name: myapp
replicaCount: "3"
image:
  tag: ""
```
template
```yaml
name: {{ .Values.name | upper | quote }}
tag:  {{ .Values.image.tag | default .Chart.AppVersion }}
replicas: {{ .Values.replicaCount | int }}
```
output
```yaml
name: "MYAPP"
tag:  1.27.0
replicas: 3
```

### Handy functions

| Function | Use |
|----------|-----|
| `quote` / `squote` | wrap in quotes |
| `default` | fallback value |
| `required "msg" .Values.x` | fail rendering if missing |
| `toYaml` | dump a map/list as YAML |
| `nindent N` / `indent N` | indent (nindent adds a leading newline) |
| `trunc 63` / `trimSuffix "-"` | DNS-safe names |
| `upper`, `lower`, `title`, `replace` | strings |
| `b64enc` / `b64dec` | for Secrets |
| `sha256sum` | checksum (e.g. force pod restart on config change) |
| `tpl` | render a string from values as a template |
| `lookup` | read live objects from the cluster (empty in `helm template`/dry-run) |
| `fail "msg"` | abort rendering |

#### `quote` / `squote`
values.yaml
```yaml
version: 1.10
enabled: true
```
template
```yaml
a: {{ .Values.enabled | quote }}
b: {{ .Values.enabled | squote }}
c: {{ "1.10" | quote }}
```
output
```yaml
a: "true"
b: 'true'
c: "1.10"
```
>  `version: 1.10` in values.yaml is already the **float** `1.1`. Quote it in values.yaml: `version: "1.10"`.

#### `default`
values.yaml
```yaml
logLevel: ""
```
template
```yaml
level: {{ .Values.logLevel | default "info" }}
```
output
```yaml
level: info
```

#### `required`
values.yaml
```yaml
image:
  repository: ""
```
template
```yaml
image: {{ required "image.repository is required" .Values.image.repository }}
```
output
```text
Error: execution error at (mychart/templates/deployment.yaml:1:10): image.repository is required
```

#### `toYaml`
values.yaml
```yaml
labels:
  team: backend
  tier: api
```
template
```yaml
metadata:
  labels:
{{ toYaml .Values.labels | indent 4 }}
```
output
```yaml
metadata:
  labels:
    team: backend
    tier: api
```

#### `indent` vs `nindent`
Both produce the same output; `nindent` adds a newline first, so the tag can sit at the end of the line above with `{{-`.
```yaml
# indent: tag must start at column 0
  labels:
{{ toYaml .Values.labels | indent 4 }}

# nindent: tag can be indented like the YAML around it (preferred)
  labels:
    {{- toYaml .Values.labels | nindent 4 }}
```

#### `trunc` / `trimSuffix`
Kubernetes names must be ≤ 63 characters. Truncating can leave a trailing `-`, so trim it.

template
```yaml
a: {{ "my-release-name" | trunc 11 }}
b: {{ "my-release-name" | trunc 11 | trimSuffix "-" }}
```
output
```yaml
a: my-release-
b: my-release
```

#### `upper`, `lower`, `title`, `replace`
template
```yaml
a: {{ "hello world" | upper }}
b: {{ "HELLO" | lower }}
c: {{ "hello world" | title }}
d: {{ "hello world" | replace " " "-" }}
```
output
```yaml
a: HELLO WORLD
b: hello
c: Hello World
d: hello-world
```

#### `b64enc` / `b64dec`
values.yaml
```yaml
db:
  password: password
```
template
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db
data:
  password: {{ .Values.db.password | b64enc }}
# decode example: {{ "cGFzc3dvcmQ=" | b64dec }}
```
output
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db
data:
  password: cGFzc3dvcmQ=
# decode example: password
```
> Tip: use `stringData:` in Secrets and you can skip `b64enc` entirely.

#### `sha256sum`
template
```yaml
hash: {{ "hello" | sha256sum }}
```
output
```yaml
hash: 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
```
Same input → same hash. Different input → different hash. See [Restart pods when a ConfigMap changes](#restart-pods-when-a-configmap-changes) for the real use.

#### `tpl`
Lets **values** contain template code.

values.yaml
```yaml
greeting: "Hello from {{ .Release.Name }} in {{ .Release.Namespace }}"
```
template
```yaml
without: {{ .Values.greeting | quote }}
with:    {{ tpl .Values.greeting . | quote }}
```
output
```yaml
without: "Hello from {{ .Release.Name }} in {{ .Release.Namespace }}"
with:    "Hello from demo in dev"
```

#### `lookup`
Reads a live object from the cluster: `lookup "apiVersion" "Kind" "namespace" "name"`.
Classic use: keep a generated password stable across upgrades.

template
```yaml
{{- $existing := lookup "v1" "Secret" .Release.Namespace "app-secret" }}
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  {{- if $existing }}
  password: {{ index $existing.data "password" }}
  {{- else }}
  password: {{ randAlphaNum 16 | b64enc }}
  {{- end }}
```
output on first install (random value)
```yaml
data:
  password: Wk5xN2RqVjRMcU1ia0hwVA==
```
On every upgrade the **same** password is reused.
> `lookup` returns an empty result in `helm template` and client-side `--dry-run`, so you'll always see the `else` branch there.

#### `fail`
values.yaml
```yaml
ingress:
  enabled: true
  host: ""
```
template
```yaml
{{- if and .Values.ingress.enabled (not .Values.ingress.host) }}
{{- fail "ingress.host must be set when ingress.enabled is true" }}
{{- end }}
```
output
```text
Error: execution error at (mychart/templates/ingress.yaml:2:4): ingress.host must be set when ingress.enabled is true
```

---

## toYaml + nindent (the most used pattern)

values.yaml
```yaml
resources:
  limits:
    cpu: 100m
    memory: 128Mi
```
template
```yaml
    containers:
      - name: app
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```
output
```yaml
    containers:
      - name: app
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
```
> The number in `nindent N` = the column where the child keys should start.

---

## Control flow

### `if` / `else if` / `else`
values.yaml
```yaml
ingress:
  enabled: false
route:
  enabled: true
```
template
```yaml
{{- if .Values.ingress.enabled }}
expose: ingress
{{- else if .Values.route.enabled }}
expose: route
{{- else }}
expose: none
{{- end }}
```
output
```yaml
expose: route
```

### Falsy values
Falsy: `false`, `0`, `""`, `nil` (missing key), empty list `[]`, empty map `{}`.

values.yaml
```yaml
a: 0
b: ""
c: []
d: "false"
```
template
```yaml
a: {{ if .Values.a }}yes{{ else }}no{{ end }}
b: {{ if .Values.b }}yes{{ else }}no{{ end }}
c: {{ if .Values.c }}yes{{ else }}no{{ end }}
d: {{ if .Values.d }}yes{{ else }}no{{ end }}
e: {{ if .Values.missing }}yes{{ else }}no{{ end }}
```
output
```yaml
a: no
b: no
c: no
d: yes      # the STRING "false" is non-empty → truthy!
e: no
```

### Logic: `and`, `or`, `not`, `eq`, `ne`, `lt`, `gt`
Functions come **first** (prefix style): `eq A B`, not `A eq B`.

values.yaml
```yaml
mode: prod
replicas: 3
debug: false
```
template
```yaml
{{- if and (eq .Values.mode "prod") (gt (int .Values.replicas) 1) }}
ha: true
{{- end }}
{{- if or .Values.debug (ne .Values.mode "prod") }}
verbose: true
{{- end }}
{{- if not .Values.debug }}
debug: off
{{- end }}
```
output
```yaml
ha: true
debug: off
```

### `with` — changes scope (`.`)
Inside `with`, `.` **becomes** the value. If the value is empty, the whole block is skipped.

values.yaml
```yaml
nodeSelector:
  disktype: ssd
```
template
```yaml
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 2 }}
{{- end }}
```
output
```yaml
nodeSelector:
  disktype: ssd
```
With `nodeSelector: {}` the output is **nothing** — no empty `nodeSelector:` key.

#### Using `$` to reach the root
```yaml
{{- with .Values.nodeSelector }}
# .Release.Name would FAIL here, because . is now the nodeSelector map
release: {{ $.Release.Name }}
{{- end }}
```
output
```yaml
release: demo
```

### `range` — loops

Over a **list**:

values.yaml
```yaml
ports: [80, 443]
```
template
```yaml
ports:
{{- range .Values.ports }}
  - containerPort: {{ . }}
{{- end }}
```
output
```yaml
ports:
  - containerPort: 80
  - containerPort: 443
```

Over a **map** (keys come out sorted alphabetically):

values.yaml
```yaml
env:
  LOG_LEVEL: debug
  APP_MODE: prod
```
template
```yaml
env:
{{- range $key, $val := .Values.env }}
  - name: {{ $key }}
    value: {{ $val | quote }}
{{- end }}
```
output
```yaml
env:
  - name: APP_MODE
    value: "prod"
  - name: LOG_LEVEL
    value: "debug"
```

With an **index**:
```yaml
{{- range $i, $p := .Values.ports }}
port{{ $i }}: {{ $p }}
{{- end }}
```
output
```yaml
port0: 80
port1: 443
```

---

## Variables

Variables start with `$`, are set with `:=` and are handy inside `range`/`with`, where `.` has changed.

values.yaml
```yaml
hosts: [a.example.com, b.example.com]
```
template
```yaml
{{- $svc := printf "%s-web" .Release.Name }}
rules:
{{- range .Values.hosts }}
  - host: {{ . }}
    service: {{ $svc }}
{{- end }}
```
output
```yaml
rules:
  - host: a.example.com
    service: demo-web
  - host: b.example.com
    service: demo-web
```

---

## Named templates (`_helpers.tpl`)

`define` creates a reusable snippet. Files starting with `_` aren't rendered on their own.

`templates/_helpers.tpl`
```yaml
{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```
`templates/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
```
output
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo
  labels:
    app.kubernetes.io/name: mychart
    app.kubernetes.io/instance: demo
```

### Why `include` and not `template`
```yaml
labels:
  {{- include "mychart.labels" . | nindent 4 }}   # output can be piped → correctly indented
labels:
  {{ template "mychart.labels" . }}               #  can't pipe → second line lands at column 0
```

### Always pass the context `.`
```yaml
{{ include "mychart.labels" . }}    # app.kubernetes.io/instance: demo
{{ include "mychart.labels" }}      #  error / empty values — the template gets no data
```

---

## Restart pods when a ConfigMap changes

Changing only a ConfigMap does **not** restart pods. Put the ConfigMap's hash in the pod annotations: when the config changes, the hash changes, the pod spec changes, and Kubernetes rolls the pods.

`templates/configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  LOG_LEVEL: {{ .Values.logLevel | quote }}
```
`templates/deployment.yaml`
```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```
output
```yaml
      annotations:
        checksum/config: 6f1c0e...   # 64-char hash, changes whenever logLevel changes
```
Try it: `helm upgrade demo ./mychart --set logLevel=debug` → pods restart.

---

## Files

Chart layout:
```
mychart/
├── files/
│   ├── app.conf     # contents: port=8080
│   └── log.conf     # contents: level=info
└── templates/
    └── configmap.yaml
```

### One file: `.Files.Get`
template
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.conf: |-
    {{- .Files.Get "files/app.conf" | nindent 4 }}
```
output
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.conf: |-
    port=8080
```

### Many files: `.Files.Glob` + `.AsConfig` / `.AsSecrets`
template
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: all-configs
data:
  {{- (.Files.Glob "files/*").AsConfig | nindent 2 }}
```
output (one key per file, named after the file)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: all-configs
data:
  app.conf: |
    port=8080
  log.conf: |
    level=info
```
`.AsSecrets` does the same but base64-encodes the contents, for use under a Secret's `data:`.

> Files in `templates/` can't be read with `.Files`, and files listed in `.helmignore` are excluded.
