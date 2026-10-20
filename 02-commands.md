# 02 · CLI Commands

## Repositories
```bash
helm repo add <name> <url>
helm repo list
helm repo update
helm repo remove <name>
helm search repo <keyword>            # search local repo indexes
helm search repo <chart> --versions   # list all versions
helm search hub <keyword>             # search Artifact Hub
```

## Inspecting charts
```bash
helm show chart  <chart>      # Chart.yaml
helm show values <chart>      # default values.yaml
helm show readme <chart>
helm show all    <chart>
helm pull <chart> --untar     # download and unpack locally
```

## Installing
```bash
helm install <release> <chart> [flags]

# common flags
-n, --namespace <ns>
--create-namespace
-f, --values <file>           # can repeat; later files win
--set key=value               # override single values
--set-string key=value        # force string type
--set-file key=path           # value from file contents
--version <x.y.z>             # chart version
--wait                        # wait until resources are ready
--timeout 5m
--atomic                      # roll back automatically if install fails
--dry-run                     # render + validate, don't install
--generate-name               # let Helm pick a release name
```

## Upgrading & rolling back
```bash
helm upgrade <release> <chart> -f values.yaml
helm upgrade --install <release> <chart>   # idempotent: install if missing (great for CI)

--reuse-values      # keep previously set values, merge new --set on top
--reset-values      # discard previous values, use chart defaults + new ones
--atomic            # roll back on failure
--cleanup-on-fail   # delete new resources created in a failed upgrade

helm history <release>
helm rollback <release> [revision]         # no revision = previous one
```

> `--reuse-values` can hide new defaults added in a newer chart version. Prefer passing your full values files every time.

## Inspecting releases
```bash
helm list                    # current namespace
helm list -A                 # all namespaces
helm list --all              # include failed/uninstalled
helm status <release>
helm get values <release>            # user-supplied values
helm get values <release> --all      # computed values
helm get manifest <release>          # rendered YAML that was applied
helm get notes <release>
helm get hooks <release>
```

## Uninstalling
```bash
helm uninstall <release> -n <ns>
helm uninstall <release> --keep-history   # keep record so you can roll back
```

## Chart development
```bash
helm create <name>
helm lint <chart-dir>
helm template <release> <chart-dir> [-f values.yaml] [--debug]
helm template <release> <chart-dir> -s templates/deployment.yaml   # render one file
helm package <chart-dir>              # -> name-version.tgz
helm dependency update <chart-dir>    # download deps into charts/
helm dependency build  <chart-dir>    # rebuild from Chart.lock
helm test <release>                   # run chart tests
```

## Misc
```bash
helm env          # Helm's environment variables / paths
helm version
helm plugin list
helm completion bash|zsh|fish
```
