### **Lab 01: Creating a Kind Cluster**

[Kind](https://kind.sigs.k8s.io) allows a developer to run a Kubernetes Cluster on their desktop using docker. This is an optional step. If you have an existing cluster you want to use, skip to Lab 02.

**Prerequisites:**

- Ensure that [Docker](https://www.docker.com/products/docker-desktop) is installed.

**Steps:**

1. Install Kind via the instructions in the [Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) guide.

2. If running an M1 Macintosh, ensure that a minimum version of Kind 0.11.0 is running:

```
$ kind version

kind v0.16.0 go1.19.1 darwin/arm64
```

3. Create a Kind cluster:

```
$ kind create cluster

Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.2) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊
```

4. Get the information about your cluster:

```
$ kubectl cluster-info --context kind-kind

Kubernetes control plane is running at https://127.0.0.1:54385
CoreDNS is running at https://127.0.0.1:54385/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

5. Get your current kubectl context:

```
$ kubectl config current-context

kind-kind
```

6. Check to see what pods are running:

```
$ kubectl get pods --all-namespaces

NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
kube-system          coredns-565d847f94-5ffn7                     1/1     Running   0          68s
kube-system          coredns-565d847f94-jz9bs                     1/1     Running   0          68s
kube-system          etcd-kind-control-plane                      1/1     Running   0          82s
kube-system          kindnet-mllf9                                1/1     Running   0          68s
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          82s
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          82s
kube-system          kube-proxy-7sbsx                             1/1     Running   0          68s
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          82s
local-path-storage   local-path-provisioner-684f458cdd-45l6q      1/1     Running   0          68s
```

**Lab 01 Complete.** 

- Continue to [Lab 02](../lab02/lab02.md).
