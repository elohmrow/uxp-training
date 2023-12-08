### **Lab 04 [Azure]: Deploying a Managed Resource with Family Providers**

[Managed resources](https://docs.crossplane.io/master/concepts/managed-resources/) are the Crossplane representation of the cloud provider resources. In this lab, we will be deploying an Azure Resource Group with Crossplane.

**Prerequisites:**

- Ensure that you have completed [Lab 03](../../lab03/azure/lab03.md), or you have a running UXP instance and provider-family-azure is installed.

**Steps:**

1. Login to Azure with CLI

```
$ az login

A web browser has been opened at https://login.microsoftonline.com/organizations/oauth2/v2.0/authorize. Please continue the login in the web browser. If no web browser is available or if the web browser fails to open, use device code flow with `az login --use-device-code`.
[
  {
    "cloudName": "AzureCloud",
    "homeTenantId": "<redacted>",
    "id": "<redacted>",
    "isDefault": true,
    "managedByTenants": [],
    "name": "Upbound Azure Subscription",
    "state": "Enabled",
    "tenantId": "<redacted>",
    "user": {
      "name": "bradley@upbound.io",
      "type": "user"
    }
  }
]

```

2. Create an Azure Service principal. Note that this service principal needs to include fields for the SDK.

```
$ az ad sp create-for-rbac --sdk-auth --role Contributor --scopes /subscriptions/$(az account show --query 'id' --output=tsv) > "creds.json"

WARNING: Creating 'Contributor' role assignment under scope '/subscriptions/<redacted>'
WARNING: The output includes credentials that you must protect. Be sure that you do not include these credentials in your code or check the credentials into your source control. For more information, see https://aka.ms/azadsp-cli
```

3. Next, we need to create a Kubernetes secret that contains the credentials we created in step #1.

```
$ kubectl create secret generic azure-account-creds -n upbound-system --from-file=credentials=./creds.json

secret/azure-account-creds created
```

4. Create a **_ProviderConfig_** object referring to the credentials secret created in the previous step:

```
$ vi providerconfig.yaml 

apiVersion: azure.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: upbound-system
      name: azure-account-creds
      key: credentials

$ kubectl apply -f providerconfig.yaml

providerconfig.azure.upbound.io/default created
```

5. Now, we are ready to provision Azure resources. Create a **_ResourceGroup_** object:

```
$ vi resourcegroup.yaml

apiVersion: azure.upbound.io/v1beta1
kind: ResourceGroup
metadata:
  name: xp-training-rg-${USER} # ${USER} can be anything. 
spec:
  forProvider:
    location: West Europe

$ kubectl apply -f resourcegroup.yaml

resourcegroup.azure.upbound.io/xp-training-rg-${USER} created
```

6. Check the status the **_ResourceGroup_** object and verify that it is **_SYNCED_** and **_READY_**:

```
$ kubectl get resourcegroup.azure.upbound.io

NAME                     READY   SYNCED   EXTERNAL-NAME            AGE
xp-training-rg-${USER}   True    True     xp-training-rg-${USER}   2m52s
```

7. You can always check the events on the object to find the problems if something goes wrong:

```
$ kubectl describe resourcegroup.azure.upbound.io xp-training-rg-${USER}

...
Spec:
  Deletion Policy:  Delete
  For Provider:
    Location:  West Europe
  Init Provider:
  Management Policies:
    *
  Provider Config Ref:
    Name:  default
Status:
  At Provider:
    Id:        /subscriptions/<redacted>/resourceGroups/xp-training-rg-${USER}
    Location:  westeurope
  Conditions:
    Last Transition Time:  2023-11-05T12:45:03Z
    Reason:                Available
    Status:                True
    Type:                  Ready
    Last Transition Time:  2023-11-05T12:44:57Z
    Reason:                ReconcileSuccess
    Status:                True
    Type:                  Synced
    Last Transition Time:  2023-11-05T12:45:00Z
    Reason:                Success
    Status:                True
    Type:                  LastAsyncOperation
    Last Transition Time:  2023-11-05T12:45:00Z
    Reason:                Finished
    Status:                True
    Type:                  AsyncOperation
Events:
  Type     Reason                       Age    From                                                  Message
  ----     ------                       ----   ----                                                  -------
  Normal   CreatedExternalResource      3m38s  managed/azure.upbound.io/v1beta1, kind=resourcegroup  Successfully requested creation of external resource

```

8. Go to the [Resource Groups in the Azure Console](https://portal.azure.com/#blade/HubsExtension/BrowseResourceGroups) and verify that the corresponding Resource Group is created there.

**Cleanup:**

1. Delete the Resource group object from the cluster. Then confirm that it has been deleted in the Azure console:

```
$ kubectl delete resourcegroup.azure.upbound.io xp-training-rg-${USER}

resourcegroup.azure.upbound.io "xp-training-rg-${USER}" deleted
```

**Bonus Exercise**
Investigate the [Marketplace Documentation](https://marketplace.upbound.io/providers/upbound/provider-azure-network/v0.37.1) and deploy a [VirtualNetwork](https://marketplace.upbound.io/providers/upbound/provider-azure-network/v0.37.1/resources/network.azure.upbound.io/VirtualNetwork/v1beta1) associated with your Resource Group.

Please use version `v.0.37.1`!

**Lab 04 Complete.**

- Continue to [Lab 05](../../lab05/azure/lab05.md).
