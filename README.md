# ftmcloud PaaS Infrastructure
Contains all helm charts and overrides for server-side PaaS infrastructure,
including the Rancher dashboard.

## Provisioning the VMs for the Cluster
First, provision the VMs:

```commandline
multipass launch --name k3s-master --cpus 2 --memory 8G --disk 300G --network Ethernet
multipass launch --name k3s-worker1 --cpus 2 --memory 8G --disk 300G --network Ethernet
multipass launch --name k3s-worker2 --cpus 2 --memory 8G --disk 300G --network Ethernet
```

Deploy k3s on master:

```commandline
 multipass exec k3s-master -- bash -c "curl -sfL https://get.k3s.io | sh -"
```

Get the master node IP:

```commandline
IP=$(multipass info k3s-master | grep IPv4 | awk '{print $2}')
```

Retrieve the token:

```commandline
TOKEN=$(multipass exec k3s-master sudo cat /var/lib/rancher/k3s/server/node-token)
```

Connect the nodes using:

```commandline
for f in 1 2; do
    multipass exec k3s-worker$f -- bash -c "curl -sfL https://get.k3s.io | K3S_URL=\"https://$IP:6443\" K3S_TOKEN=\"$TOKEN\" sh -"
done
```

Once all nodes are connected, will need to ensure the network is bridged with the host network. If you've not done 
this, stop all VM instances within the Multipass UI and enable.

Retrieve the public IP of the master node, then update the kubeconfig to reflect the new IP. Verify connectivity and
proceed to provisioning the PaaS infrastructure within the cluster.


## PaaS Installation (Helm Subcharts Combo)
First, update your kubeconfig with the file in `conf/kubeconf` using a tool like kubecm. Then,

Provision CRDS:
helm template paas . --set installCRDs=true | kubectl apply -f -

Install chart:
helm install paas .


## Paas Installation (Manually Provision Cluster)
To manually provision without using the chart-of-charts setup (can be a headache when trying to ensure each
component is installed in its own respective namespace:

### cert-manager
helm repo add jetstack https://charts.jetstack.io --force-update


### ingress-nginx

### longhorn

### argocd
TODO.
