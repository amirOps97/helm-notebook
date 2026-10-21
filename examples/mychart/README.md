# mychart

Minimal example chart demonstrating helpers, labels, `with`/`range`, `toYaml | nindent`, a config checksum annotation, NOTES.txt and a chart test.

```bash
helm lint .
helm template demo . --set env.LOG_LEVEL=debug
helm install demo . -n demo --create-namespace
helm test demo -n demo
```
