### **Lab 05 [Azure]: Building a Composition with Family Providers**

[Compositions](https://docs.crossplane.io/latest/concepts/compositions/) allow platform builders to define new custom resources that are composed of managed resources. In this lab, we will be building a composition for a new type, **_CompositePostgreSQLInstance,_** which is composed of a couple of managed resources, namely **_ResourceGroup, Subnet, VirtualNetwork, VirtualNetworkRule, and Server._**

**Prerequisites:** 

* Ensure that you have completed [Lab 04](../../lab04/azure/lab04.md), or you have a running UXP instance, provider-azure is installed, and a default ProviderConfig pointing to azure credentials secret exists.

**Steps:**

1. The first step of building a composition is defining a new kind of [Composite Resource](https://docs.crossplane.io/latest/concepts/composite-resource-definitions/). Composite resources are defined by a **_CompositeResourceDefinition (XRD)._**

    Inspect and create this [CompositeResourceDefinition](https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/configuration/definition.yaml) on the cluster, which will:

* Define a **_CompositePostgreSQLInstance _** as a **_CompositeResource(XR)_**
* Offer a **_PostgreSQLInstance claim (XRC)_** for that XR.

```
$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/configuration/definition.yaml

$ kubectl apply -f definition.yaml

compositeresourcedefinition.apiextensions.crossplane.io/compositepostgresqlinstances.database.example.org created
```


2. Check the status of the **_CompositeResourceDefinition_** and verify that it is **_ESTABLISHED_** and **_OFFERED_**:

```
$ kubectl get xrd

NAME                                                ESTABLISHED   OFFERED   AGE
compositepostgresqlinstances.database.example.org   True          True      21s
```


3. Verify that the new types are defined on the cluster:

```
$ kubectl get crd | grep postgresqlinstance

compositepostgresqlinstances.database.example.org          2023-11-05T13:07:33Z
postgresqlinstances.database.example.org                   2023-11-05T13:07:33Z
```


4. [Specify how this resource should be composed](https://docs.crossplane.io/latest/concepts/compositions/). This is done by another custom resource called **Composition**

   Inspect and create this [Composition](https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/configuration/composition.yaml) on the cluster:

```
$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/configuration/composition.yaml

$ kubectl apply -f composition.yaml

composition.apiextensions.crossplane.io/vpcpostgresqlinstances.aws.database.example.org created
```


5. Create a Secret for database password

```
$ kubectl -n upbound-system create secret generic getting-started-db-pass --from-literal=password="Upb0ndR0cks"

secret/getting-started-db-pass created
```


6. As in [Lab 3](../../lab03/azure/lab03.md), we need to add a couple of Providers. This is because the Provider we have already installed does not have all the things we need. In this case, we need to add [provider-azure-dbforpostgresql](https://marketplace.upbound.io/providers/upbound/provider-azure-dbforpostgresql/v0.37.1) and [provider-azure-network](https://marketplace.upbound.io/providers/upbound/provider-azure-network/v0.37.1). You can search for them as you searched in Lab 3 for provider-family-azure, or you can do the following:

    For provider-azure-network:


```
$ vi provider-azure-network.yaml  

apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-network
spec:
  package: xpkg.upbound.io/upbound/provider-azure-network:v0.37.1

$ kubectl apply -f provider-azure-network.yaml

provider.pkg.crossplane.io/provider-azure-network created
```


For provider-azure-dbforpostgresql:


```
$vi provider-azure-dbforpostgresql.yaml

apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-dbforpostgresql
spec:
  package: xpkg.upbound.io/upbound/provider-azure-dbforpostgresql:v0.37.1

$ kubectl apply -f provider-azure-dbforpostgresql.yaml

provider.pkg.crossplane.io/provider-azure-dbforpostgresql created
```


Make sure everything is there, INSTALLED and HEALTHY:


```
$ kubectl get provider.pkg

NAME                             INSTALLED   HEALTHY   PACKAGE                                                          AGE
provider-azure-dbforpostgresql   True        True      xpkg.upbound.io/upbound/provider-azure-dbforpostgresql:v0.37.1   97s
provider-azure-network           True        True      xpkg.upbound.io/upbound/provider-azure-network:v0.37.1           2m4s
provider-family-azure            True        True      xpkg.upbound.io/upbound/provider-family-azure:v0.37.1            58m

$ kubectl get providers
```


7. Now we are ready to claim our new Infrastructure resource. Create the following [Composite Resource Claim (XRC)](https://docs.crossplane.io/latest/concepts/claims/) object:

```
$ vi claim.yaml

apiVersion: database.example.org/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: my-db-azure-${USER} # ${USER} can be anything
  namespace: default
spec:
  parameters:
    storageGB: 20
    passwordSecretName: getting-started-db-pass
  compositionSelector:
    matchLabels:
      uxp-guide: getting-started
      provider: azure
  writeConnectionSecretToRef:
    name: db-conn-azure

$ kubectl apply -f claim.yaml

postgresqlinstance.database.example.org/my-db-azure-${USER} created
```


8. Check all the managed resources created for this XRC object:

```
$ kubectl get managed -l crossplane.io/claim-name=my-db-azure-${USER}

NAME                                                             READY   SYNCED   EXTERNAL-NAME                     AGE
resourcegroup.azure.upbound.io/my-db-azure-bradley-7hht2-d2z9n   True    True     my-db-azure-bradley-7hht2-d2z9n   12m
NAME                                                                           READY   SYNCED   EXTERNAL-NAME                          AGE
server.dbforpostgresql.azure.upbound.io/my-db-azure-bradley-7hht2-postgresql   True    True     my-db-azure-bradley-7hht2-postgresql   12m
NAME                                                                                  READY   SYNCED   EXTERNAL-NAME                     AGE
virtualnetworkrule.dbforpostgresql.azure.upbound.io/my-db-azure-bradley-7hht2-8d9pn   True    True     my-db-azure-bradley-7hht2-8d9pn   12m
NAME                                                                      READY   SYNCED   EXTERNAL-NAME                     AGE
virtualnetwork.network.azure.upbound.io/my-db-azure-bradley-7hht2-fpvhb   True    True     my-db-azure-bradley-7hht2-fpvhb   12m
NAME                                                              READY   SYNCED   EXTERNAL-NAME                     AGE
subnet.network.azure.upbound.io/my-db-azure-bradley-7hht2-c68k8   True    True     my-db-azure-bradley-7hht2-c68k8   12m

```


9. Check the status of XRC resource:

```
$ kubectl -n default get postgresqlinstance

NAME          READY   CONNECTION-SECRET   AGE
my-db-azure   False   db-conn             2m52s
```


10. Wait for enough time until all managed resources are **_READY_** and **_SYNCED._** You can check the output of the commands in the previous two steps. In the meantime, you can verify that the resources are actually created on the Azure side by navigating through the Azure portal.
11. Eventually (in around 10 mins) you should see our XRC is ready.

```
$ kubectl -n default get postgresqlinstance

NAME          READY   CONNECTION-SECRET   AGE
my-db-azure   True    db-conn             13m
```


and its [connection details](https://github.com/upbound/uxp-training/blob/main/lab05/configuration/composition.yaml#L200-L206) are available in a k8s secret named **_db-conn_**:

	


```
$ kubectl -n default describe secrets db-conn-azure

Name:         db-conn-azure
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  connection.crossplane.io/v1alpha1

Data
====
endpoint:  63 bytes
password:  11 bytes
username:  10 bytes

```


**Cleanup (skip if you are proceeding with Lab 06):**


1. Delete the **_PostgreSQLInstance_** object from cluster:

```
$ kubectl -n default delete PostgreSQLInstance my-db-azure
```


2. Verify that corresponding managed resources are being deleted and eventually disappeared:

```
$ kubectl get managed -l crossplane.io/claim-name=my-db-azure
```

**Lab 5 Complete.**

- Continue to [Lab 06](file:///Users/ghost/upbound/uxp-training/lab06/azure/lab06.md).
