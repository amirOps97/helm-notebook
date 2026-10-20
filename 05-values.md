# 05 · Values

## Precedence (lowest → highest)
1. `values.yaml` in the chart
2. Parent chart's `values.yaml` (for subcharts)
3. `-f / --values` files, in order given (last wins)
4. `--set` / `--set-string` / `--set-file` (last wins)

```bash
helm upgrade --install app ./mychart \
  -f values.yaml -f values-prod.yaml \
  --set image.tag=1.2.3
```

## `--set` syntax
```bash
--set a.b=c                         # nested
--set list={a,b,c}                  # list
--set servers[0].port=80            # list item field
--set name=value1\,value2           # escape comma
--set annotations."example\.com/x"=y  # escape dots in keys
--set-string tag=1.10               # keep as string (not float!)
--set key=null                      # delete a default key
```

## Typical per-environment layout
```
values.yaml           # sane defaults
values-dev.yaml       # only what differs in dev
values-prod.yaml      # only what differs in prod
```

## Values schema validation
`values.schema.json` is checked on install, upgrade, lint and template:
```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image"],
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1 },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": { "type": "string" },
        "tag": { "type": "string" }
      }
    }
  }
}
```

## Tips
- Document every value with a comment in `values.yaml` — it's your chart's API.
- Use camelCase keys; avoid hyphens (they're awkward in templates: `index .Values "my-key"`).
- Quote strings that look like numbers/booleans: `tag: "1.10"`, `enabled: "true"` vs `true`.
- `helm get values <release> --all` shows the final merged values.
