### **Lab 07 [AWS]: XR Status Patching**

In this lab we will extend the existing composition from Lab 06 with XRD status patching.

**Prerequisites:**

- Ensure that you have completed Lab 04, or you have a running UXP instance, provider-aws is installed and a default ProviderConfig pointing to AWS credentials secret exists and you have working, tested composition from Lab 06.
- We will need an additional provider for this lab: provider-aws-iam. You can get it here: [https://marketplace.upbound.io/providers/upbound/provider-aws-iam/v0.43.1](https://marketplace.upbound.io/providers/upbound/provider-aws-iam/v0.43.1)

```
$ vi provider-aws-iam.yaml

apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-iam
spec:
  package: xpkg.upbound.io/upbound/provider-aws-iam:v0.43.1

$ kubectl apply -f provider-aws-iam.yaml

provider.pkg.crossplane.io/provider-aws-iam created
```

**Tasks**:

1. Expose composed RDS Instance status to XR status. Use `x-kubernetes-preserve-unknown-fields: true` to propagate all the fields
2. Extend Composition with IAM Policy that uses Instance ARN from the XR status field

**Steps:**

1. Get the files from lab06(previous lab): XRD, Composition, Claim

```
$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab06-transform-patch/configuration/definition.yaml

$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab06-transform-patch/configuration/composition.yaml

$ wget -q https://raw.githubusercontent.com/upbound/uxp-training/main/lab06-transform-patch/claim.yaml
```

2. Open `definition.yaml` and extend it with a new `status` section just after the last `required` section. Pay attention to indentation. The `status` section should be at the same level as `spec`

```
            required:
              - parameters
          status:
            description: A Status represents the observed state
            properties:
              instance:
                description: Freeform field containing status information from RDS Instance
                type: object
                x-kubernetes-preserve-unknown-fields: true
            type: object

```

Notice `x-kubernetes-preserve-unknown-fields: true.`

    It allows the propagation of the set of API fields without explicitly naming them.

3. Open `composition.yaml` and add a new patch of type `ToCompositeFieldPath` to RDS Instance MR after the last patch and just before `connectionDetails`.


```
       …
          - type: ToCompositeFieldPath
            fromFieldPath: "status.atProvider"
            toFieldPath: "status.instance"
        connectionDetails:
          - name: username
            fromFieldPath: spec.forProvider.username
```

4. Apply the new XRD, Composition, and Claim.

```
$ kubectl apply -f definition.yaml

compositeresourcedefinition.apiextensions.crossplane.io/compositepostgresqlinstances.database.example.org configured

$ kubectl apply -f composition.yaml

composition.apiextensions.crossplane.io/vpcpostgresqlinstances.aws.database.example.org configured

$ kubectl apply -f claim.yaml

postgresqlinstance.database.example.org/my-db unchanged

```

5. Check that the Instance status was properly propagated to the Claim

```
$ kubectl describe postgresqlinstance my-db

Status:
  Conditions:
    Last Transition Time:  2023-11-04T15:46:16Z
    Reason:                ReconcileSuccess
    Status:                True
    Type:                  Synced
    Last Transition Time:  2023-11-04T15:54:18Z
    Reason:                Available
    Status:                True
    Type:                  Ready
  Connection Details:
    Last Published Time:  2023-11-04T15:54:18Z
  Instance:
    Address:                                my-db-q9pm7-27l6f.c0cmytscebma.us-east-1.rds.amazonaws.com
    Allocated Storage:                      20
    Apply Immediately:                      true
    Arn:                                    arn:aws:rds:us-east-1:656321468224:db:my-db-q9pm7-27l6f
    Auto Minor Version Upgrade:             true
    Availability Zone:                      us-east-1c
    Backup Retention Period:                0
    Backup Window:                          09:04-09:34
    Ca Cert Identifier:                     rds-ca-2019
    Character Set Name:                     
    Copy Tags To Snapshot:                  false
    Custom Iam Instance Profile:            
    Customer Owned Ip Enabled:              false
    Db Name:                                
    Db Subnet Group Name:                   my-db-q9pm7-8fr7t
    Delete Automated Backups:               true
    Deletion Protection:                    false
    Domain:                                 
    Domain Iam Role Name:                   
    Endpoint:                               my-db-q9pm7-27l6f.c0cmytscebma.us-east-1.rds.amazonaws.com:5432
    Engine:                                 postgres
    Engine Version:                         12.15
    Engine Version Actual:                  12.15
    Hosted Zone Id:                         Z2R2ITUGPM61AM
    Iam Database Authentication Enabled:    false
    Id:                                     my-db-q9pm7-27l6f
    Instance Class:                         db.t2.small
    Iops:                                   0
    Kms Key Id:                             
    Latest Restorable Time:                 
    License Model:                          postgresql-license
    Maintenance Window:                     tue:03:49-tue:04:19
    Max Allocated Storage:                  0
    Monitoring Interval:                    0
    Monitoring Role Arn:                    
    Multi Az:                               false
    Name:                                   
    Nchar Character Set Name:               
    Network Type:                           IPV4
    Option Group Name:                      default:postgres-12
    Parameter Group Name:                   default.postgres12
    Performance Insights Enabled:           false
    Performance Insights Kms Key Id:        
    Performance Insights Retention Period:  0
    Port:                                   5432
    Publicly Accessible:                    true
    Replica Mode:                           
    Replicate Source Db:                    
    Resource Id:                            db-TUW2AZ664JAZCUXJPLKEKGIHKA
    Skip Final Snapshot:                    true
    Status:                                 available
    Storage Encrypted:                      false
    Storage Throughput:                     0
    Storage Type:                           gp2
    Tags:
      Crossplane - Kind:            instance.rds.aws.upbound.io
      Crossplane - Name:            my-db-q9pm7-27l6f
      Crossplane - Providerconfig:  default
    Tags All:
      Crossplane - Kind:            instance.rds.aws.upbound.io
      Crossplane - Name:            my-db-q9pm7-27l6f
      Crossplane - Providerconfig:  default

```

6. Open `composition.yaml` and add a new IAM Policy managed resource with a set of transform string patches to the end of the file

```
      - name: iampolicy
        base:
          apiVersion: iam.aws.upbound.io/v1beta1
          kind: Policy
        patches:
          - fromFieldPath: "metadata.name"
            toFieldPath: "spec.forProvider.name"
            transforms:
              - type: string
                string:
                  fmt: "%s-postgresql"
          - fromFieldPath: "status.instance.arn"
            toFieldPath: "spec.forProvider.policy"
            transforms:
              - type: string
                string:
                  fmt: |
                    {
                       "Version": "2012-10-17",
                       "Statement": [
                          {
                             "Effect": "Allow",
                             "Action": [
                                 "rds-db:connect"
                             ],
                             "Resource": [
                                 "%s"
                             ]
                          }
                       ]
                    }

```

7. Notice how we are using `status.instance.arn `from XR status field to dynamically template IAM policy with RDS Instance ARN

8. Update the Composition

```
$ kubectl apply -f composition.yaml

composition.apiextensions.crossplane.io/vpcpostgresqlinstances.aws.database.example.org configured
```

9. Find the new Policy resource and check that all the data was properly propagated. You should see that `spec.forProvider.policy` is dynamically patched with the `status.instance.arn`

```
$ kubectl get managed -l crossplane.io/claim-name=my-db | grep policy

policy.iam.aws.upbound.io/my-db-q9pm7-dqstt   True    True     my-db-q9pm7-dqstt   39s

$ kubectl get policy.iam.aws.upbound.io/my-db-q9pm7-dqstt -o yaml

...
...
...
spec:
  deletionPolicy: Delete
  forProvider:
    path: /
    policy: |
      {
         "Version": "2012-10-17",
         "Statement": [
            {
               "Effect": "Allow",
               "Action": [
                   "rds-db:connect"
               ],
               "Resource": [
                   "arn:aws:rds:us-east-1:656321468224:db:my-db-q9pm7-27l6f"
               ]
            }
         ]
      }

```

**Cleanup:**

1. Delete the **_PostgreSQLInstance_** object from cluster:

```
$ kubectl -n default delete PostgreSQLInstance my-db

postgresqlinstance.database.example.org "my-db" deleted
```

2. Verify that corresponding managed resources are being deleted and eventually disappeared:

```
$ kubectl get managed -l crossplane.io/claim-name=my-db

No resources found
```

**Lab 07 Complete.**

- Continue to [Lab 08](../../lab08/aws/lab08.md).
