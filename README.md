# gratefuldog2
aws implementation in us-west2 using karepenter and oauth2

## Required Utilities
Install these tools before proceeding:

1. AWS CLI
2. kubectl - the Kubernetes CLI
3. terraform - infrastructure-as-code tool made by HashiCorp
4. helm - the package manager for Kubernetes


## cluster setup
1. `export AWS_DEFAULT_REGION="us-west-2"`
2. update Terraform code:
  + region
  + ec2 instance type
3. create ec2 spot instance linked role
  + `aws iam create-service-linked-role --aws-service-name spot.amazonaws.com`
```
{
    "Role": {
        "Path": "/aws-service-role/spot.amazonaws.com/",
        "RoleName": "AWSServiceRoleForEC2Spot",
        "RoleId": "AROATP3GKDEDTFBPTAZPD",
        "Arn": "arn:aws:iam::240195868935:role/aws-service-role/spot.amazonaws.com/AWSServiceRoleForEC2Spot",
        "CreateDate": "2022-08-29T12:25:20+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Action": [
                        "sts:AssumeRole"
                    ],
                    "Effect": "Allow",
                    "Principal": {
                        "Service": [
                            "spot.amazonaws.com"
                        ]
                    }
                }
            ]
        }
    }
}
```
4. Configure the KarpenterNode IAM Role 
  + The EKS module creates an IAM role for the EKS managed node group nodes
  + We’ll use that for Karpenter so we don’t have to reconfigure the aws-auth ConfigMap
  + we will need to create an instance profile we can reference.

5. Create the KarpenterController IAM Role
  + Karpenter requires permissions like launching instances, which means it needs an IAM role that grants it access.
  + create an AWS IAM Role, attach a policy, and authorize the Service Account to assume the role using IRSA
6. Install Karpenter Helm Chart
  + use the helm_release Terraform resource to do the deploy and pass in the cluster details and IAM role Karpenter needs to assume.
7. setup the karpenter provisioner
  + A single Karpenter provisioner is capable of handling many different pod shapes.
  + Karpenter makes scheduling and provisioning decisions based on pod attributes such as labels and affinity
  + thus Karpenter eliminates the need to manage many different node groups.
  + This provisioner will configure instances to connect to your cluster’s endpoint and discover resources like subnets and security groups using the cluster’s name.
  + The `ttlSecondsAfterEmpty` value configures Karpenter to terminate empty nodes. This behavior can be disabled by leaving the value undefined
  + **Note:** *This provisioner will create capacity as long as the sum of all created capacity is less than the specified limit.*
8. update kubectl:   `aws eks update-kubeconfig --name karpenter-demo`
```
Added new context arn:aws:eks:us-west-2:240195868935:cluster/karpenter-demo to /Users/robert/.kube/config
```
9. **setup complete**

## Automatic Node Provisioning
```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              cpu: 1
EOF
kubectl scale deployment inflate --replicas 5
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller

```

## links 
https://www.golinuxcloud.com/setup-kubernetes-cluster-on-aws-ec2/
https://docs.aws.amazon.com/eks/latest/userguide/choosing-instance-type.html
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html#AvailableInstanceTypes
https://karpenter.sh/v0.16.0/provisioner/
https://www.ianlewis.org/en/almighty-pause-container
https://aws.github.io/aws-eks-best-practices/karpenter/
https://aws.amazon.com/blogs/containers/using-amazon-ec2-spot-instances-with-karpenter/
