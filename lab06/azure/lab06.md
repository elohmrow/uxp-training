### **Lab 06 [Azure]: Extending a Composition with Family Providers**

In this lab we will extend the existing composition from Lab 05.

**Prerequisites:**

- Ensure that you have completed [Lab 04](../../lab04/azure/lab04.md), or you have a running UXP instance, provider-family-azure is installed, and a default ProviderConfig pointing to Azure credentials secret exists, and you have a working, tested composition from [Lab 05](../../lab05/azure/lab05.md).

**Tasks**:

1. Extend your CompositePostgreSQLInstance XRD parameters with a new field called `spec.parameters.class`. This new field should accept only values of `small`, `medium`, and `large`. Other values should be automatically rejected by the API server. Consult with [Swagger Data Models](https://swagger.io/docs/specification/data-models/) for implementation details.

2. Extend your `xpostgresqlinstances.azure.database.example.org` Composition with a `transform` patch that automatically converts “small/medium/large” input values from `spec.parameters.class` to supported Server [`skuName`s](https://marketplace.upbound.io/providers/upbound/provider-azure-dbforpostgresql/v0.37.1/resources/dbforpostgresql.azure.upbound.io/Server/v1beta1#doc:spec-forProvider-skuName) and patch the `spec.forProvider.instanceClass` of your composed instance Managed Resource. The target instance type selection is up to you. Consult with [Transform a Patch](https://docs.crossplane.io/latest/concepts/patch-and-transform/#transform-a-patch) for implementation details.

3. Extend your PostgreSQLInstance Claim with the new field and test the implementation end-to-end.

**Steps:**

1. Get the files from lab05: XRD, Composition, Claim

```
$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/configuration/definition.yaml

$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/configuration/composition-azure.yaml

$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/claim-azure.yaml
```

Remember to randomize the db name in the Claim, as in Lab 05 … `my-db-azure` should be something like `my-db-azure-myName` instead.

```
$ wget -q https://github.com/elohmrow/uxp-training/blob/family-providers/lab05/family-providers/azure/provider-azure-dbforpostgresql.yaml

$ wget -q https://github.com/elohmrow/uxp-training/blob/family-providers/lab05/family-providers/azure/provider-azure-network.yaml
```

2. Open `definition.yaml` and extend it with a new `class` field together with the [Enum](https://swagger.io/docs/specification/data-models/enums/) based validation under the `parameters` section.

```
                  class:
                    type: string
                    enum: [small, medium, large]
```

Pay attention to indentation. The `class` property should be at the same level as `storageGB` and `passwordSecretName` as follows:

```
             parameters:
                type: object
                properties:
                  class:
                    type: string
                    enum: [small, medium, large]
                  storageGB:
                    type: integer
                  passwordSecretName:
                    type: string
                required:
                  - storageGB
                  - passwordSecretName

```

3. Next, we need a way of mapping `[small, medium, large]` classes to Azure instance types, like `GP_Gen5_2`. We can do this by using a Crossplane [Map transformation](https://docs.crossplane.io/latest/concepts/patch-and-transform/#map-transforms). By using a Map you can provide users a generic interface that can be customized for each cloud provider.

Note that the composition-azure.yaml file contains multiple Managed Resources. It’s a good practice to name each resource in a Composition, and the resource we want to change can be found by looking for `name: postgresqlserver` in the composition-azure.yaml file.

While many values in the Composition are hard-coded, Crossplane [Patches](https://docs.crossplane.io/latest/concepts/patch-and-transform/#types-of-patches) allow us to dynamically update values based on user request.

Edit `composition-azure.yaml`, find the postgresqlserver, go to the patches section and extend it with the addition of a transform patch of map type.

```
          - fromFieldPath: spec.parameters.class
            toFieldPath: spec.forProvider.skuName
            transforms:
            - type: map
              map:
                small: GP_Gen5_2
                medium: GP_Gen5_4
                large: MO_Gen5_4

```

4. Apply the new XRD and Composition

```
$ kubectl apply -f definition.yaml

compositeresourcedefinition.apiextensions.crossplane.io/compositepostgresqlinstances.database.example.org configured

$ kubectl apply -f composition-azure.yaml

composition.apiextensions.crossplane.io/xpostgresqlinstances.azure.database.example.org created
```

5. Modify `claim-azure.yaml` adding the new field into Claim definition

```
spec:
  parameters:
    storageGB: 20
    passwordSecretName: getting-started-db-pass
    class: wrongclass
```

6. Test that API validation works

```
$ kubectl apply -f claim-azure.yaml

The PostgreSQLInstance "my-db-azure-bda" is invalid: spec.parameters.class: Unsupported value: "wrongclass": supported values: "small", "medium", "large"
```

7. Fix the Claim by specifying the valid class name and test the instantiation

```
$ vi claim-azure.yaml # Change to `class: small`

$ kubectl apply -f claim-azure.yaml

postgresqlinstance.database.example.org/my-db-azure-bda created
```

8. Check that Instance object was patched properly

```
$ kubectl get managed -l crossplane.io/claim-name=my-db-azure | grep server
server.dbforpostgresql.azure.upbound.io/my-db-azure-sn67r-postgresql   False   True     my-db-azure-sn67r-postgresql   2m37

$ kubectl get server.dbforpostgresql.azure.upbound.io/my-db-azure-sn67r-postgresql -o yaml | grep skuName
    skuName: GP_Gen5_2

```

9. If time allows wait for the full <code>PostgreSQLInstance </code>deployment and validate proper creation of connection secret


**Cleanup (skip if you are proceeding with Lab 07):**


1. Delete the **_PostgreSQLInstance_** object from cluster:

```
$ kubectl -n default delete PostgreSQLInstance my-db-azure
```


2. Verify that corresponding managed resources are being deleted and eventually disappeared:

```
$ kubectl get managed -l crossplane.io/claim-name=my-db-azure
```

**Lab 6 Complete.**

- Continue to [Lab 07](../../lab07/azure/lab07.md).
