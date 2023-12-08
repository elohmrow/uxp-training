### **Lab 04 [AWS]: Deploying a Managed Resource**

[Managed resources](https://docs.crossplane.io/latest/concepts/managed-resources/) are the Crossplane representation of the cloud provider resources. In this lab, we will be deploying an AWS VPC with Crossplane.

**Prerequisites:**

- Ensure that you have completed [Lab 03](../../lab03/aws/lab03.md), or you have a running UXP instance and provider-aws is installed.

**Steps:**

1. Create a Kubernetes secret with AWS credentials:

```
$ AWS_PROFILE=default && echo -e "[default]\naws_access_key_id = $(aws configure get aws_access_key_id --profile $AWS_PROFILE)\naws_secret_access_key = $(aws configure get aws_secret_access_key --profile $AWS_PROFILE)" > creds.conf

$ cat creds.conf

[default]
aws_access_key_id=<access_key>
aws_secret_access_key=<secret_key>


$ kubectl -n upbound-system create secret generic aws-creds --from-file=creds=./creds.conf

secret/aws-creds created
```

2. Create a **_ProviderConfig_** object referring to the credentials secret created in the previous step:

```
$ vi providerconfig.yaml

apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: upbound-system
      name: aws-creds
      key: creds

$ kubectl apply -f providerconfig.yaml

providerconfig.aws.upbound.io/default created
```

3. Now, we are ready to provision AWS resources. Create a **_VPC_** object:

```
$ vi vpc.yaml

apiVersion: ec2.aws.upbound.io/v1beta1
kind: VPC
metadata:
  name: training-vpc
spec:
  forProvider:
    region: us-west-1
    cidrBlock: 172.16.0.0/16
    tags:
      Name: TrainingVpc

$ kubectl apply -f vpc.yaml

vpc.ec2.aws.upbound.io/training-vpc created
```

4. Check the status the **_VPC_** object and verify that it is **_SYNCED_** and **_READY_**:

```
$ kubectl get vpc.ec2.aws.upbound.io

NAME           READY   SYNCED   EXTERNAL-NAME           AGE
training-vpc   True    True     vpc-05cb46b298319ccc0   55s
```

5. You can always check the events on the object to find the problems if something goes wrong:

```
$ kubectl describe vpc.ec2.aws.upbound.io training-vpc

...
Status:
  At Provider:
    Arn:                                   arn:aws:ec2:us-west-1:609897127049:vpc/vpc-05cb46b298319ccc0
    assignGeneratedIpv6CidrBlock:          false
    Cidr Block:                            172.16.0.0/16
    Default Network Acl Id:                acl-0a043ecd17a9af807
    Default Route Table Id:                rtb-08e5f2e720014d69c
    Default Security Group Id:             sg-02f7d5a3bc990bea7
    Dhcp Options Id:                       dopt-8e0b0fe9
    Enable Classiclink:                    false
    Enable Classiclink Dns Support:        false
    Enable Dns Hostnames:                  false
    Enable Dns Support:                    true
    Enable Network Address Usage Metrics:  false
    Id:                                    vpc-05cb46b298319ccc0
    Instance Tenancy:                      default
    ipv6AssociationId:
    ipv6CidrBlock:
    ipv6CidrBlockNetworkBorderGroup:
    ipv6IpamPoolId:
    ipv6NetmaskLength:                     0
    Main Route Table Id:                   rtb-08e5f2e720014d69c
    Owner Id:                              609897127049
    Tags:
      Name:                         TrainingVpc
      Crossplane - Kind:            vpc.ec2.aws.upbound.io
      Crossplane - Name:            training-vpc
      Crossplane - Providerconfig:  default
    Tags All:
      Name:                         TrainingVpc
      Crossplane - Kind:            vpc.ec2.aws.upbound.io
      Crossplane - Name:            training-vpc
      Crossplane - Providerconfig:  default
  Conditions:
    Last Transition Time:  2023-11-04T12:44:11Z
    Reason:                Available
    Status:                True
    Type:                  Ready
    Last Transition Time:  2023-11-04T12:44:01Z
    Reason:                ReconcileSuccess
    Status:                True
    Type:                  Synced
    Last Transition Time:  2023-11-04T12:44:05Z
    Reason:                Success
    Status:                True
    Type:                  LastAsyncOperation
    Last Transition Time:  2023-11-04T12:44:05Z
    Reason:                Finished
    Status:                True
    Type:                  AsyncOperation
Events:
  Type     Reason                       Age   From                                          Message
  ----     ------                       ----  ----                                          -------
  Normal   CreatedExternalResource      82s   managed/ec2.aws.upbound.io/v1beta1, kind=vpc  Successfully requested creation of external resource

```

6. Verify that the VPC was created either through the aws cli or the console

   1. `kubectl get vpc`

   Copy the external name and use it with the following command

   `aws --profile default ec2 describe-vpcs --region us-west-1 --vpc-ids <PASTE_EXTERNAL_NAME_HERE>`

   2. Go to the [VPCs on the AWS console](https://console.aws.amazon.com/vpc/home?region=us-east-1#vpcs:) and verify that the corresponding VPC is created there. You can find **_VPC ID_** under the **_EXTERNAL-NAME_** column of the **_kubectl get vpc_** output in step 4.

**Cleanup:**

1. Delete the VPC object from cluster:

```
$ kubectl delete vpc.ec2.aws.upbound.io training-vpc

vpc.ec2.aws.upbound.io "training-vpc" deleted
```

**Lab 04 Complete.**

- Continue to [Lab 04 - Komoplane](lab04-komoplane.md).
