### **Lab 02: Installing UXP**

In this lab, we will be installing [Upbound Universal Crossplane (UXP)](https://github.com/upbound/universal-crossplane) to a Kubernetes Cluster using the **_up_** CLI tool.

**Prerequisites:**

- Ensure that you have completed [Lab 01](../lab01/lab01.md), or you have a Kubernetes cluster available.

**Steps:**

1. Install the **_up_** CLI following the instructions at [https://docs.upbound.io/reference/cli/](https://docs.upbound.io/reference/cli/)

   The easiest way is to download and run the install script:

```
$ curl -sL "https://cli.upbound.io" | sh

Up downloaded successfully!
By proceeding, you are accepting to comply with terms and conditions in https://licenses.upbound.io/upbound-software-license.html

Run the following commands to finish installation:

sudo mv up /usr/local/bin/
up --version

Visit https://upbound.io to get started. 🚀
Have a nice day! 👋


$ sudo mv up /usr/local/bin/
```

2. Verify that **_up_** has been installed to your system, and you are running a minimum version of v0.19.0

```
$ up --version

v0.21.0
```

3. Ensure that your kubectl context is set to the correct cluster. If not, [use the right context](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/):

```
$ kubectl config current-context

kind-kind
```

4. Install UXP using **_up uxp_**. This will create the **_upbound-system_** namespace and install UXP components:

```
$ up uxp install

UXP 1.13.2-up.3 installed
```

5. Finally, validate the UXP install. All the pods, deployments and replicasets should be **_READY_** or **_AVAILABLE_**:

```
$ kubectl get all -n upbound-system

NAME                                          READY   STATUS    RESTARTS   AGE
pod/crossplane-5d49b9cb9b-t5xfg               1/1     Running   0          31s
pod/crossplane-rbac-manager-dd95c68f6-lscqr   1/1     Running   0          31s

NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/crossplane-webhooks   ClusterIP   10.96.74.241   <none>        9443/TCP   31s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/crossplane                1/1     1            1           31s
deployment.apps/crossplane-rbac-manager   1/1     1            1           31s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/crossplane-5d49b9cb9b               1         1         1       31s
replicaset.apps/crossplane-rbac-manager-dd95c68f6   1         1         1       31s
```

6. Things to note about a UXP install:
   1. UXP installs to a cluster’s **_upbound-system_** namespace instead of **_crossplane-system_** like upstream Crossplane.
   2. UXP can also be installed via [Helm 3](https://github.com/upbound/universal-crossplane#installation-with-helm-3)
   3. The **_up_** CLI accepts helm options for configuring an install.
   4. The **_up_** CLI can be used to upgrade an existing Crossplane installation to UXP, keeping the install in the crossplane-system namespace. See [up uxp usage](https://docs.upbound.io/reference/cli/command-reference/).

**Lab 02 Complete.** 

- If you are using AWS, Continue to [AWS Lab 03](../lab03/aws/lab03.md).
- If you are using Azure, Continue to [Azure Lab 03]().
- If you are using GCP, Continue to [GCP Lab 03]().
