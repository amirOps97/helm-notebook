# ⎈ Helm Notes

Personal reference notes for [Helm](https://helm.sh), the package manager for Kubernetes.

## Quick cheat sheet

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami   # add a repo
helm repo update                                           # refresh indexes
helm search repo nginx                                     # search added repos
helm show values bitnami/nginx > values.yaml               # grab default values

helm install my-app ./mychart -n dev --create-namespace    # install
helm upgrade --install my-app ./mychart -f values-dev.yaml # install or upgrade
helm list -A                                               # releases in all namespaces
helm status my-app                                         # release status
helm history my-app                                        # revisions
helm rollback my-app 2                                     # roll back to revision 2
helm uninstall my-app                                      # remove release

helm create mychart                                        # scaffold a chart
helm lint ./mychart                                        # check chart
helm template my-app ./mychart --debug                     # render locally
helm get manifest my-app                                   # what's actually deployed
```
