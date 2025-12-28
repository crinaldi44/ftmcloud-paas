# Multipass + K3s + Rancher + Longhorn

This README walks you through setting up a **single-node K3s cluster inside a Multipass VM**, with:

* **Longhorn** as the persistent storage layer
* **Rancher** as the Kubernetes management UI

The steps are intentionally ordered and written so that **almost everything is a copy‑paste shell command**.

---

## Prerequisites (on your local machine)

* `multipass`
* `kubectl`
* `helm`

Verify:

```bash
multipass version
kubectl version --client
helm version
```

---

## 1. Create the Multipass VM

Create a VM with enough resources for Rancher + Longhorn:

```bash
multipass launch \
  --name k3s-rancher \
  --cpus 4 \
  --memory 8G \
  --disk 80G
```

Shell into it:

```bash
multipass shell k3s-rancher
```

---

## 2. Install K3s

Install K3s with default components (Traefik included):

```bash
curl -sfL https://get.k3s.io | sh -
```

Verify:

```bash
sudo kubectl get nodes
sudo kubectl get pods -A
```

---

## 3. Configure kubectl for convenience

Inside the VM:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

Test:

```bash
kubectl get nodes
```

---

## 4. Install Longhorn prerequisites

Longhorn requires iSCSI and NFS support:

```bash
sudo apt update
sudo apt install -y open-iscsi nfs-common
sudo systemctl enable --now iscsid
```

Verify:

```bash
lsmod | grep iscsi || true
```

---

## 5. Install Longhorn

Add the Helm repo:

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

Install Longhorn:

```bash
kubectl create namespace longhorn-system

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system
```

Wait until all pods are running:

```bash
kubectl get pods -n longhorn-system
```

---

## 6. Make Longhorn the default StorageClass

Check storage classes:

```bash
kubectl get storageclass
```

Set Longhorn as default:

```bash
kubectl patch storageclass longhorn \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

(Optional) Disable the local-path default:

```bash
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

---

## 7. Install cert-manager (required by Rancher)

Add repo:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

Install:

```bash
kubectl create namespace cert-manager

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set installCRDs=true
```

Verify:

```bash
kubectl get pods -n cert-manager
```

---

## 8. Install Rancher

### Choose a hostname

Pick the hostname you will eventually use (even before DNS exists):

```
rancher.example.com
```

### Install Rancher

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update

kubectl create namespace cattle-system

helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --set replicas=1
```

Wait for Rancher to come up:

```bash
kubectl get pods -n cattle-system
```

---

## 9. Access Rancher (temporary local access)

Port-forward Rancher:

```bash
kubectl port-forward -n cattle-system svc/rancher 8443:443
```

Open:

```
https://localhost:8443
```

Complete:

* Admin password setup
* Initial cluster confirmation

---

## 10. Access Longhorn UI (temporary)

```bash
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
```

Open:

```
http://localhost:8080
```

Verify:

* Node is healthy
* Disk is available
* No replica errors

---

## 11. Final State Checklist

You now have:

* ✅ Multipass VM running K3s
* ✅ Longhorn installed and default
* ✅ cert-manager installed
* ✅ Rancher installed and functional

This is the **correct foundation** before exposing anything externally.

---

## 12. "Chart of charts" (one command install)

Instead of installing each chart by hand, use **Helmfile** to install everything in the right order.

Why Helmfile?

* It *is* effectively a “chart of charts” (orchestrates multiple Helm charts)
* It avoids embedding cert-manager as a sub-chart (cert-manager’s docs recommend installing it standalone) ([cert-manager.io](https://cert-manager.io/docs/installation/helm/?utm_source=chatgpt.com))

### Create a bootstrap repo (inside the VM)

```bash
mkdir -p ~/bootstrap && cd ~/bootstrap

cat > helmfile.yaml <<'YAML'
repositories:
  - name: jetstack
    url: https://charts.jetstack.io
  - name: longhorn
    url: https://charts.longhorn.io
  - name: rancher-latest
    url: https://releases.rancher.com/server-charts/latest
  - name: cloudflare
    url: https://cloudflare.github.io/helm-charts

releases:
  # cert-manager (required by Rancher)
  - name: cert-manager
    namespace: cert-manager
    createNamespace: true
    chart: jetstack/cert-manager
    set:
      - name: installCRDs
        value: true

  # Longhorn
  - name: longhorn
    namespace: longhorn-system
    createNamespace: true
    chart: longhorn/longhorn

  # Rancher
  - name: rancher
    namespace: cattle-system
    createNamespace: true
    chart: rancher-latest/rancher
    values:
      - hostname: rancher.vibeify.co
        replicas: 1

  # Cloudflare Tunnel (remotely managed)
  # NOTE: this chart expects a tunnel token (string) in Values.
  # See cloudflare-tunnel-remote templates (token key) ([github.com](https://github.com/cloudflare/helm-charts/blob/main/charts/cloudflare-tunnel-remote/templates/deployment.yaml?utm_source=chatgpt.com))
  - name: cloudflare-tunnel
    namespace: cloudflare
    createNamespace: true
    chart: cloudflare/cloudflare-tunnel-remote
    values:
      - cloudflare:
          tunnel_token: "${CLOUDFLARE_TUNNEL_TOKEN}"
YAML
```

### Install Helmfile

```bash
sudo apt update
sudo apt install -y wget tar

# Install helmfile (Linux x86_64)
HF_VER="0.166.0"
wget -qO- "https://github.com/helmfile/helmfile/releases/download/v${HF_VER}/helmfile_${HF_VER}_linux_amd64.tar.gz" \
  | sudo tar -xz -C /usr/local/bin helmfile

helmfile --version
```

### One command install

1. Export your Cloudflare tunnel token (you’ll create it in the next section):

```bash
export CLOUDFLARE_TUNNEL_TOKEN='PASTE_YOUR_TOKEN_HERE'
```

2. Run the install:

```bash
cd ~/bootstrap
helmfile apply
```

---

## 13. Cloudflare Tunnel for `vibeify.co`

You can expose Rancher (and anything else) *without opening inbound ports* by running `cloudflared` in the cluster and routing `vibeify.co` hostnames through Cloudflare.

Cloudflare’s Kubernetes guide for **remotely-managed tunnels** uses a **tunnel token stored in a Kubernetes secret / values** ([developers.cloudflare.com](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/deployment-guides/kubernetes/?utm_source=chatgpt.com)).

### 13.1 Prereqs (in Cloudflare)

1. `vibeify.co` is added to your Cloudflare account and using Cloudflare nameservers
2. In **Zero Trust** → **Networks** → **Tunnels**: create a new tunnel and copy the **token**

The token is a long string that starts like `eyJhIjoi...` ([developers.cloudflare.com](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/deployment-guides/kubernetes/?utm_source=chatgpt.com)).

### 13.2 Route traffic to your cluster

**Recommended pattern for K3s + Traefik:**

* Cloudflare Tunnel routes `rancher.vibeify.co` → **Traefik** in-cluster
* Kubernetes Ingress handles host-based routing

Traefik service in K3s is typically:

* `traefik.kube-system.svc.cluster.local:80`

In Cloudflare Zero Trust, add **Public Hostnames** like:

* `rancher.vibeify.co` → `http://traefik.kube-system.svc.cluster.local:80`

Then create (or rely on) an Ingress that routes that host to the Rancher service.

> Rancher’s install docs require setting the `hostname` when installing via Helm ([ranchermanager.docs.rancher.com](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster?utm_source=chatgpt.com)).

### 13.3 Create the Rancher Ingress (Traefik)

Rancher’s chart usually creates an ingress automatically. If you want to be explicit, apply this (works well with Traefik on K3s):

```bash
cat > rancher-ingress.yaml <<'YAML'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rancher
  namespace: cattle-system
spec:
  rules:
    - host: rancher.vibeify.co
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rancher
                port:
                  number: 80
YAML

kubectl apply -f rancher-ingress.yaml
```

### 13.4 Optional: Longhorn UI via tunnel

Longhorn also has a UI. You can expose it the same way:

1. Add a Cloudflare Public Hostname:

* `longhorn.vibeify.co` → `http://traefik.kube-system.svc.cluster.local:80`

2. Create an ingress:

```bash
cat > longhorn-ingress.yaml <<'YAML'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn
  namespace: longhorn-system
spec:
  rules:
    - host: longhorn.vibeify.co
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: longhorn-frontend
                port:
                  number: 80
YAML

kubectl apply -f longhorn-ingress.yaml
```

---

## 14. Quick validation commands

```bash
kubectl get nodes
kubectl get pods -A
kubectl get storageclass
kubectl get ingress -A
```

---

## Notes on TLS (important)

* You will get the smoothest experience if `rancher.vibeify.co` is served over HTTPS.
* With Cloudflare Tunnel you have a few good options:

  * **Cloudflare Access** in front of Rancher (recommended)
  * Terminate TLS at Cloudflare edge and use HTTP to origin (lab-friendly)
  * Or issue real origin certs via cert-manager + DNS-01 (more setup)

If you tell me which TLS approach you want (quick/lab vs. “proper”), I’ll tailor the exact commands/values for that.
