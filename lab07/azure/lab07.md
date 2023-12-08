### **Lab 07 [Azure]: XR Status Patching with Family Providers**

In this lab we will extend the existing composition from [Lab 06](../../lab06/azure/lab06.md) with XRD status patching

**Prerequisites:**

- Ensure that you have completed [Lab 04](../../lab04/azure/lab04.md), or you have a running UXP instance, provider-azure is installed, and a default ProviderConfig pointing to Azure credentials secret exists and you have a working, tested composition from [Lab 06](../../lab06/azure/lab06.md).

**Tasks**:

1. Expose your composed PostgreSQLInstance subnet to the XR status. Use `x-kubernetes-preserve-unknown-fields: true` to propagate arbitrary fields
2. Extend a Composition with a FirewallRule that uses a subnet range to populate the start and end IP addresses of the rule spec
3. Use a pipeline of string transforms to construct the IP addresses

**Steps:**

1. Get the files from lab06: XRD, Composition, Claim

```
$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/configuration/definition.yaml

$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/configuration/composition-azure.yaml

$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/claim-azure.yaml
```

Remember to randomize the db name in the Claim, as in Lab 05 … `my-db-azure` should be something like `my-db-azure-myName` instead.

```
$ wget -q https://raw.githubusercontent.com/elohmrow/uxp-training/family-providers/lab05/family-providers/azure/provider-azure-network.yaml

$ wget -q https://raw.githubusercontent.com/elohmrow/uxp-training/family-providers/lab05/family-providers/azure/provider-azure-dbforpostgresql.yaml
```

2. Open `definition.yaml` and extend it with a new `status` section just after the last `required` section. Pay attention to indentation. The `status` section should be at the same level as `spec`

```
            required:
              - parameters
          status:
            description: A Status represents the observed state
            properties:
              instance:
                description: Freeform field containing status information from PostgreSQLInstance
                type: object
                x-kubernetes-preserve-unknown-fields: true
            type: object

```

Notice `x-kubernetes-preserve-unknown-fields: true.`

This allows the propagation of the set of API fields without explicitly naming them.

3. Open `composition-azure.yaml` and add a new patch of type `ToCompositeFieldPath` to the composed Subnet.

```
      patches:
      - type: ToCompositeFieldPath
        fromFieldPath: status.atProvider.addressPrefixes[0]
        toFieldPath: status.instance.subnet
```

4. Apply new XRD, Composition and Claim

```
$ kubectl apply -f definition.yaml

compositeresourcedefinition.apiextensions.crossplane.io/compositepostgresqlinstances.database.example.org configured

$ kubectl apply -f composition-azure.yaml

composition.apiextensions.crossplane.io/xpostgresqlinstances.azure.database.example.org configured

$ kubectl apply -f claim-azure.yaml

postgresqlinstance.database.example.org/my-db-azure-bandersen created
```

5. Check that the address range from the Subnet status was properly propagated to the Claim status. It should be available immediately after the composed Subnet reaches `Ready` state

```
$ kubectl describe postgresqlinstance my-db-azure
…
Status:
  At Provider:
    Address Prefixes:
      192.168.1.0/24
```

6. Open `composition-azure.yaml` and add a new FirewallRule Managed Resource with a set of transform string patches to the end of the file

```
  - name: firewallrule
    base:
      apiVersion: dbforpostgresql.azure.upbound.io/v1beta1
      kind: FirewallRule
      spec:
        forProvider:
          resourceGroupNameSelector:
            matchControllerRef: true
          serverNameSelector:
            matchControllerRef: true
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: status.instance.subnet
      toFieldPath: spec.forProvider.startIpAddress
      transforms:
      - type: string
        string:
          type: TrimSuffix
          trim: "0/24"
      - type: string
        string:
          fmt: "%s1" #192.168.1.1
    - type: FromCompositeFieldPath
      fromFieldPath: status.instance.subnet
      toFieldPath: spec.forProvider.endIpAddress
      transforms:
      - type: string
        string:
          type: TrimSuffix
          trim: "0/24"
      - type: string
        string:
          fmt: "%s2" #192.168.1.2
```

7. Notice how we are using `status.instance.subnet `from XR status field to dynamically template FirewallRule and with the help of additional transforms dynamically create desired IP range

8. Update the Composition

```
$ kubectl apply -f composition-azure.yaml

composition.apiextensions.crossplane.io/xpostgresqlinstances.azure.database.example.org configured
```

9. Find the new FirewallRule resource and check that all the data was properly propagated. You should see that `spec.forProvider.startIpAddress` and `spec.forProvider.endIpAddress` is dynamically patched with the data from `status.instance.subnet`.

```
$ kubectl get managed -l crossplane.io/claim-name=my-db-azure|grep -i firewallrule

firewallrule.dbforpostgresql.azure.upbound.io/my-db-azure-bandersen-wgtg4-shrht   True    True     my-db-azure-bandersen-wgtg4-shrht   68s

$ kubectl get firewallrule.dbforpostgresql.azure.upbound.io/my-db-azure-bandersen-wgtg4-shrht -o yaml

...
status:
  atProvider:
    endIpAddress: 192.168.1.2
    id: /subscriptions/038f2b7c-3265-43b8-8624-c9ad5da610a8/resourceGroups/my-db-azure-bandersen-wgtg4-zdj2t/providers/Microsoft.DBforPostgreSQL/servers/my-db-azure-bandersen-wgtg4-postgresql/firewallRules/my-db-azure-bandersen-wgtg4-shrht
    resourceGroupName: my-db-azure-bandersen-wgtg4-zdj2t
    serverName: my-db-azure-bandersen-wgtg4-postgresql
    startIpAddress: 192.168.1.1

```

**Cleanup:**

1. Delete the **_PostgreSQLInstance_** object from the cluster:

```
$ kubectl -n default delete PostgreSQLInstance my-db-azure-bandersen

postgresqlinstance.database.example.org "my-db-azure-bandersen" deleted
```

2. Verify that corresponding managed resources are being deleted and eventually disappear:

```
$ kubectl get managed -l crossplane.io/claim-name=my-db-azure-bandersen

No resources found
```

**Lab 7 Complete.**

- Continue to [Lab 08](../../lab08/azure/lab08.md).

