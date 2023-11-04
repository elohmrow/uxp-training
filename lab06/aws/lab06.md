### **Lab 06 [AWS]: Extending a Composition**

In this lab we will extend the existing composition from Lab 05.

**Prerequisites:**

- Ensure that you have completed Lab 04, or you have a running UXP instance, provider-aws is installed and a default ProviderConfig pointing to AWS credentials secret exists and you have a working, tested composition from Lab 05.

**Tasks**:

1. Extend `CompositePostgreSQLInstance` XRD parameters with a new field called `spec.parameters.class`. The new field should accept only values of `small`, `medium`, and `large`. Other values should be automatically rejected by the API server.

Consult with [https://swagger.io/docs/specification/data-models/](https://swagger.io/docs/specification/data-models/) for implementation. (Hint, look for an `enum` type).

2. Extend the `vpcpostgresqlinstances.aws.database.example.org` Composition with a `transform` patch that automatically converts “small/medium/large” input values from `spec.parameters.class` to supported RDS instance types [https://aws.amazon.com/rds/instance-types/](https://aws.amazon.com/rds/instance-types/) and patch `spec.forProvider.instanceClass` of composed instance Managed Resource. The target instance type selection is up to you. Consult with [https://docs.crossplane.io/latest/concepts/patch-and-transform/#transform-a-patch](https://docs.crossplane.io/latest/concepts/patch-and-transform/#transform-a-patch) for implementation.
3. Extend the PostgreSQLInstance Claim with the new field and test the implementation end-to-end.

**Steps:**

1. Get the files from lab05: XRD, Composition, Claim

```
$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/configuration/definition.yaml

$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/configuration/composition.yaml

$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab05/claim.yaml
```

2. Open `definition.yaml` and extend it with a new `class` field together with the [Enum](https://swagger.io/docs/specification/data-models/enums/) based validation under the `parameters` `properties` section.

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

3. Next, we need a way of mapping `[small, medium, large]` classes to AWS instance types, like `db.t2.small`. We can do this by using a Crossplane [Map transformation](https://docs.crossplane.io/latest/concepts/patch-and-transform/#map-transforms). By using a Map you can provide users a generic interface that can be customized for each cloud provider.

Note that the composition.yaml file contains multiple Managed Resources. It’s good practice to name each resource in a Composition, and the resource we want to change can be found by looking for <strong><code>name: rdsinstance</code></strong> in the composition.yaml file.

While many values in the Composition are hard-coded, Crossplane [Patches](https://docs.crossplane.io/latest/concepts/patch-and-transform/) allow us to dynamically update values based on the user request.

Edit `composition.yaml`, find the rdsinstance, go to the patches section and extend it with the addition of a transform patch of map type.

```
          - fromFieldPath: "spec.parameters.class"
            toFieldPath: "spec.forProvider.instanceClass"
            transforms:
            - type: map
              map:
                small: db.t2.small
                medium: db.t2.medium
                large: db.t3.medium

```

4. Apply new XRD and Composition

```
$ kubectl apply -f definition.yaml

compositeresourcedefinition.apiextensions.crossplane.io/compositepostgresqlinstances.database.example.org configured

$ kubectl apply -f composition.yaml

composition.apiextensions.crossplane.io/vpcpostgresqlinstances.aws.database.example.org configured

```

5. Modify `claim.yaml` adding the new field into Claim definition

```
spec:
  parameters:
    storageGB: 20
    passwordSecretName: getting-started-db-pass
    class: wrongclass
```

6. Test that API validation works

```
$ kubectl apply -f claim.yaml

The PostgreSQLInstance "my-db" is invalid: spec.parameters.class: Unsupported value: "wrongclass": supported values: "small", "medium", "large"
```

7. Fix the Claim by specifying the valid class name and test the instantiation

```
$ vi claim.yaml # Change to `class: small`

$ kubectl apply -f claim.yaml

postgresqlinstance.database.example.org/my-db configured
```

8. Check that Instance object was patched properly

```
$ kubectl get managed -l crossplane.io/claim-name=my-db | grep instance
instance.rds.aws.upbound.io/my-db-q9pm7-27l6f   True    True     my-db-q9pm7-27l6f   43m

$ kubectl get instance.rds.aws.upbound.io/my-db-q9pm7-27l6f -o yaml | grep instanceClass

    instanceClass: db.t2.small

```

9. If time allows wait for the full <code>PostgreSQLInstance </code>deployment and validate proper creation of connection secret


**Cleanup (skip if you are proceeding with Lab 07):**

1. Delete the **_PostgreSQLInstance_** object from cluster:

```
$ kubectl -n default delete PostgreSQLInstance my-db
```

2. Verify that corresponding managed resources are being deleted and eventually disappeared:

```
$ kubectl get managed -l crossplane.io/claim-name=my-db
```

**Lab 06 Complete.**

- Continue to [Lab 07](../../lab07/aws/lab07.md).
