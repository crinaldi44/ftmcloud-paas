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

## Next Steps (not in this README)

Recommended next additions:

* Cloudflare Tunnel (cloudflared) running inside the cluster
* Secure Rancher at `https://rancher.example.com`
* Cloudflare Access in front of Rancher
* Optional: expose Longhorn UI read‑only
* Optional: GitOps with Argo CD

---

## Notes

* This setup is ideal for **labs, home clusters, and small environments**.
* Longhorn uses the node filesystem by default; adding a second disk later is supported.
* Rancher expects HTTPS to behave correctly once exposed—handle TLS carefully when adding a tunnel.

---

Happy clustering 🚀
