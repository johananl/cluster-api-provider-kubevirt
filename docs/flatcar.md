# Running Flatcar Container Linux workload clusters on CAPK

This guide explains how to run [Flatcar Container Linux](https://www.flatcar.org/) workload
clusters on the Cluster API (CAPI) provider for KubeVirt.

>NOTE: Parts of this guide are based on the [official CAPI quickstart](https://cluster-api.sigs.k8s.io/user/quick-start).

## General

The [kind](https://kind.sigs.k8s.io/) project allows running k8s clusters on a dev machine using
containers, without having to provision "real" cloud infrastructure, while KubeVirt brings virtual
machines to the k8s world by running VMs inside k8s pods.

Together, kind and KubeVirt allow running an entire CAPI deployment, including both a management
cluster and workload clusters, inside a single top-level container on a dev machine.

While KubeVirt's intended use is primarily running workloads which can't be containerized easily on
k8s, the way we use KubeVirt here is relevant mainly for experimenting with CAPI without using
cloud infrastructure as well as for developing CAPI components.

Specifically, what prompted writing this guide was the need for running Flatcar CAPI workload
clusters locally on a dev machine to test Ignition functionality in
[CABPK](https://cluster-api.sigs.k8s.io/tasks/bootstrap/kubeadm-bootstrap/) during development.

While this guide uses kind instead of cloud infrastructure, KubeVirt supports many infrastructure
providers. It should be entirely possible to deploy CAPK on any supported cloud provider and create
Flatcar workload clusters in a similar way.

## Requirements

- Make (usually installed as a system package)
- The `qemu-system-x86_64` binary (usually installed as a system package)
- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Docker](https://docs.docker.com/engine/install/)
- [kind](https://kind.sigs.k8s.io/#installation-and-usage)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [clusterctl](https://cluster-api.sigs.k8s.io/user/quick-start#install-clusterctl)
- [virtctl](https://kubevirt.io/user-guide/user_workloads/virtctl_client_tool/)
- The `remote-viewer` system package or an equivalent VNC client (optional - needed for graphical console access via VNC)

## Steps

### Build a Flatcar KubeVirt container disk

KubeVirt runs QEMU/KVM virtual machines inside k8s pods. We need to create a suitable machine image
that's packaged inside a special container image (called
[container disk](https://kubevirt.io/user-guide/storage/disks_and_volumes/#containerdisk)) which
KubeVirt can consume to create containerized VMs. We'll use the Cluster API
[image-builder](https://github.com/kubernetes-sigs/image-builder/) subproject for this.

```shell
git clone https://github.com/kubernetes-sigs/image-builder.git
cd image-builder/images/capi
```

Build the image:

```shell
# Replace with the desired k8s version
export K8S_VERSION="v1.30.1"
# Can replace with an explicit version (e.g. if an older version is needed)
export FLATCAR_VERSION="$(curl -L -s \
     "https://www.flatcar.org/releases-json/releases-stable.json" \
    | jq -r 'to_entries[] | "\(.key)"' \
    | grep -v "current" \
    | sort --version-sort \
    | tail -n1
)"
# Required for Ignition to function properly on KubeVirt
export OEM_ID=kubevirt

PACKER_FLAGS="--var 'kubernetes_semver=$K8S_VERSION'" make build-kubevirt-qemu-flatcar
```

Push the image to a container registry:

```shell
# Replace with your registry and repository
export REGISTRY="ghcr.io"
export REPOSITORY="johananl/flatcar-container-disk"

export TAG="stable-$FLATCAR_VERSION-$K8S_VERSION"

docker tag flatcar-stable-$FLATCAR_VERSION-container-disk $REGISTRY/$REPOSITORY:$TAG
docker push $REGISTRY/$REPOSITORY:$TAG
```

>NOTE: You need to ensure the image repository is **public** or provide credentials using the
>`imagePullSecrets` field of the `KubeVirt` custom resource for KubeVirt to be able to pull the
>container disk image.

### Create a CAPI management cluster

>NOTE: Because we use kind rather than a cloud provider, our workload cluster nodes will run as
>pods on our management cluster. In order to interact with the workload cluster, we'll need to
>expose its API server using a k8s service of type `LoadBalancer`. To achieve that, we need to
>replace the default kind CNI with Calico and install MetalLB as a load balancer (see below).

>NOTE: KubeVirt needs to be able to pull the container disk. In the instructions below we mount
>your Docker config file inside kind so that the kubelet can pull container disks even if they are
>hosted in a private repository. This also helps avoid Docker Hub rate limiting.

CAPI needs a *management cluster* which is a Kubernetes cluster running the CAPI components
required for managing workload clusters.

Create a custom kind configuration with the default CNI disabled and the Docker config file
mounted:

```shell
mkdir flatcar-capk && cd flatcar-capk

export DOCKER_CONFIG_FILE="$HOME/.docker/config.json"

cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
nodes:
- role: control-plane
  extraMounts:
   - containerPath: /var/lib/kubelet/config.json
     hostPath: $DOCKER_CONFIG_FILE
EOF
```

Create a kind cluster with the created config file:

```shell
kind create cluster --config=kind-config.yaml
```

Install Calico:

```shell
kubectl create -f  https://raw.githubusercontent.com/projectcalico/calico/v3.24.4/manifests/calico.yaml
```

Install MetalLB and wait for it to become ready:

```shell
METALLB_VER=$(curl "https://api.github.com/repos/metallb/metallb/releases/latest" | jq -r ".tag_name")
kubectl apply -f "https://raw.githubusercontent.com/metallb/metallb/${METALLB_VER}/config/manifests/metallb-native.yaml"
kubectl wait pods -n metallb-system -l app=metallb,component=controller --for=condition=Ready --timeout=10m
kubectl wait pods -n metallb-system -l app=metallb,component=speaker --for=condition=Ready --timeout=2m
```

Apply MetalLB layer 2 configuration:

```shell
GW_IP=$(docker network inspect -f '{{range .IPAM.Config}}{{.Gateway}}{{end}}' kind)
NET_IP=$(echo ${GW_IP} | sed -E 's|^([0-9]+\.[0-9]+)\..*$|\1|g')
cat <<EOF | sed -E "s|172.19|${NET_IP}|g" | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: capi-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.19.255.200-172.19.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
EOF
```

Install KubeVirt and wait for it to become ready (can take a few minutes):

```shell
# get KubeVirt version
KV_VER=$(curl "https://api.github.com/repos/kubevirt/kubevirt/releases/latest" | jq -r ".tag_name")
# deploy required CRDs
kubectl apply -f "https://github.com/kubevirt/kubevirt/releases/download/${KV_VER}/kubevirt-operator.yaml"
# deploy the KubeVirt custom resource
kubectl apply -f "https://github.com/kubevirt/kubevirt/releases/download/${KV_VER}/kubevirt-cr.yaml"
kubectl wait -n kubevirt kv kubevirt --for=condition=Available --timeout=10m
```

Initialize CAPI with Ignition support enabled and the KubeVirt infrastructure provider:

```shell
export EXP_KUBEADM_BOOTSTRAP_FORMAT_IGNITION=true
clusterctl init --infrastructure kubevirt
```

### Create a workload cluster

Generate a workload cluster manifest:

```shell
export CRI_PATH="/var/run/containerd/containerd.sock"
export NODE_VM_IMAGE_TEMPLATE="$REGISTRY/$REPOSITORY:$TAG"
# TODO: Update URL after merging https://github.com/kubernetes-sigs/cluster-api-provider-kubevirt/pull/302.
export CLUSTER_TEMPLATE="https://raw.githubusercontent.com/johananl/cluster-api-provider-kubevirt/refs/heads/flatcar-template/templates/cluster-template-lb-flatcar.yaml"
export SSH_AUTHORIZED_KEY="ssh-ed25519 AAAAC3N..."

clusterctl generate cluster capi-quickstart \
  --from $CLUSTER_TEMPLATE \
  --kubernetes-version $K8S_VERSION \
  --control-plane-machine-count=1 \
  --worker-machine-count=1 > cluster.yaml
```

Apply the manifest to create a workload cluster:

```shell
kubectl apply -f cluster.yaml
```

Watch the VM containers get created inside the kind container (on another shell):

```shell
docker exec -it kind-control-plane watch crictl ps
```

Watch the console of the control plane VM (can take a while until the container disk is downloaded
and the VM is running):

```shell
export VM=$(kubectl get vm \
    -l cluster.x-k8s.io/role=control-plane \
    -ojsonpath='{.items[0].metadata.name}'
)

# Serial console
virtctl console $VM

# Graphical console
virtctl vnc $VM
```

Get the workload cluster's kubeconfig and check the cluster's status:

```shell
clusterctl get kubeconfig capi-quickstart > kubeconfig
export KUBECONFIG=$(pwd)/kubeconfig
kubectl get nodes
```

In order for the workload cluster nodes to become ready, we need to deploy a CNI. We'll use Calico
again -- with some modifications required for Calico to work with our kind-based setup. Run the
following against the **workload** cluster (i.e. using the same shell with the `KUBECONFIG` env var
above):

```shell
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.4/manifests/calico.yaml -o calico-workload.yaml

sed -i -E 's|^( +)# (- name: CALICO_IPV4POOL_CIDR)$|\1\2|g;'\
's|^( +)# (  value: )"192.168.0.0/16"|\1\2"10.243.0.0/16"|g;'\
'/- name: CLUSTER_TYPE/{ n; s/( +value: ").+/\1k8s"/g };'\
'/- name: CALICO_IPV4POOL_IPIP/{ n; s/value: "Always"/value: "Never"/ };'\
'/- name: CALICO_IPV4POOL_VXLAN/{ n; s/value: "Never"/value: "Always"/};'\
'/# Set Felix endpoint to host default action to ACCEPT./a\            - name: FELIX_VXLANPORT\n              value: "6789"' \
calico-workload.yaml

kubectl create -f calico-workload.yaml
```

Ensure all workload cluster nodes are ready:

```shell
kubectl get nodes
```

Sample output:

```
NAME                                  STATUS   ROLES           AGE     VERSION
capi-quickstart-control-plane-q876l   Ready    control-plane   3m15s   v1.30.1
capi-quickstart-md-0-z9f9f-m4zs4      Ready    <none>          2m15s   v1.30.1
```

Ensure all pods are running:

```shell
kubectl get pods -A
```

Sample output:

```shell
NAMESPACE     NAME                                                          READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-7469789669-ffpj2                      1/1     Running   0          2m48s
kube-system   calico-node-lzgsh                                             1/1     Running   0          2m48s
kube-system   calico-node-x78m9                                             1/1     Running   0          2m41s
kube-system   coredns-7db6d8ff4d-kdgcr                                      1/1     Running   0          3m23s
kube-system   coredns-7db6d8ff4d-qtrtd                                      1/1     Running   0          3m23s
kube-system   etcd-capi-quickstart-control-plane-q876l                      1/1     Running   0          3m38s
kube-system   kube-apiserver-capi-quickstart-control-plane-q876l            1/1     Running   0          3m38s
kube-system   kube-controller-manager-capi-quickstart-control-plane-q876l   1/1     Running   0          3m38s
kube-system   kube-proxy-dvt9d                                              1/1     Running   0          3m23s
kube-system   kube-proxy-wwblz                                              1/1     Running   0          2m41s
kube-system   kube-scheduler-capi-quickstart-control-plane-q876l            1/1     Running   0          3m38s
```

You can SSH into any cluster node by adding the private key matching the public key specified in
`SSH_AUTHORIZED_KEY` above to your ssh-agent and running the following (get the VM name using
`kubectl get vm`):

```shell
virtctl ssh core@$VM
```

## Cleanup

To delete the workload cluster, run the following against the **management** cluster (you may have
to unset the `KUBECONFIG` environment variable):

```shell
kubectl delete -f cluster.yaml
```

Delete the management cluster:

```shell
kind delete cluster
```
