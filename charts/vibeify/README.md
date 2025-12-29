# rancher-argocd-stack

Umbrella Helm chart that deploys **Rancher** and **Argo CD** into the same Kubernetes cluster (e.g. OKE on OCI).

## Prereqs

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

## Using this chart

1. Edit `values.yaml` and set:
   - `rancher.hostname` to something like `rancher.vibeify.co`
   - `argo-cd.global.domain` and `argo-cd.server.ingress.hosts[0]` to something like `argocd.vibeify.co`
   - TLS secret names or LetsEncrypt options as appropriate.

2. Build chart dependencies:

```bash
helm dependency build ./rancher-argocd-umbrella
```

3. Install / upgrade:

```bash
helm upgrade --install rancher-argocd ./rancher-argocd-umbrella   --namespace cattle-system --create-namespace
```

(You can choose a different namespace; Rancher usually uses `cattle-system`, Argo CD often uses `argocd`. If you prefer
separate namespaces, deploy them as separate releases instead of a single umbrella.)
