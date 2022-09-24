# gratefuldog2
aws implementation in us-west2 using karepenter and oauth2

## Required Utilities
Install these tools before proceeding:

1. AWS CLI
  + manual installation  
    - https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions
    - `curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"`
    - `sudo installer -pkg AWSCLIV2.pkg -target /`
  + homebrew: `brew install awscli`
  + setup:
    - `For general use, the aws configure command in your preferred terminal is the fastest way to set up your AWS CLI installation`
    - AWS cli will prompt for pieces of information:
      * acces key ID
      * secret access key
      * AWS Region
      * output format
    - The AWS CLI stores this information in a profile (a collection of settings) named default in the credentials file
    - by default, the information in this profile is used when you run an AWS CLI command that doesn't explicitly specify a profile to use
    - by default, credentials are stored in: ~/.aws
    - The AWS CLI stores sensitive credential information that you specify with aws configure in a local file named credentials, in a folder named .aws in your home directory
    - The less sensitive configuration options that you specify with aws configure are stored in a local file named config, also stored in the .aws folder in your home directory.
2. kubectl - the Kubernetes CLI
  + **note**: You must use a kubectl version that is within one minor version difference of your Amazon EKS cluster control plane
  + brew:  `brew install kubectl`
  + verify current installation: `kubectl version | grep Client | cut -d : -f 5`
  + In order for kubectl to find and access a Kubernetes cluster, it needs a kubeconfig file
  + created automatically when you create a cluster using kube-up.sh or successfully deploy a Minikube cluster. 
  + By default, kubectl configuration is located at ~/.kube/config.
  + *when working with eks*: https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html
    - `aws sts get-caller-identity`
    - `aws eks update-kubeconfig --region <REGION> --name <CLUSTER-NAME>`
3. terraform - infrastructure-as-code tool made by HashiCorp
  + brew: `brew install terraform`
4. helm - the package manager for Kubernetes
  + `brew install helm`
5. eskctl
  + brew: `brew install eksctl`
  + `aws --version`

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

## Application load balancing on Amazon EKS
- When you create a Kubernetes ingress, an AWS Application Load Balancer (ALB) is provisioned that load balances application traffic.
- What is an application load balancer? https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html
- What is a Kubernetes ingress? https://kubernetes.io/docs/concepts/services-networking/ingress/
- ALBs can be used with pods that are deployed to nodes or to AWS Fargate.
- Application traffic is balanced at L7 of the OSI model
- To load balance network traffic at L4, you deploy a Kubernetes service of the LoadBalancer type
- At least two subnets in different Availability Zones.
- the AWS Load Balancer Controller creates ALBs and the necessary supporting AWS resources whenever a Kubernetes ingress resource is created on the cluster with the kubernetes.io/ingress.class: alb annotation.
- The ingress resource configures the ALB to route HTTP or HTTPS traffic to different pods within the cluster. 
- To ensure that your ingress objects use the AWS Load Balancer Controller, add the following annotation to your Kubernetes ingress specification:
```
annotations:
    kubernetes.io/ingress.class: alb
```
- The AWS Load Balancer Controller supports the following traffic modes:
  + instance: 
    - Registers nodes within your cluster as targets for the ALB
    - Traffic reaching the ALB is routed to NodePort for your **service** and then proxied to your pods
  + IP: 
    _ uses annotation: `alb.ingress.kubernetes.io/target-type: ip`
- Deploy a sample application



## links 
https://www.golinuxcloud.com/setup-kubernetes-cluster-on-aws-ec2/
https://docs.aws.amazon.com/eks/latest/userguide/choosing-instance-type.html
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html#AvailableInstanceTypes
https://karpenter.sh/v0.16.0/provisioner/
https://www.ianlewis.org/en/almighty-pause-container
https://aws.github.io/aws-eks-best-practices/karpenter/
https://aws.amazon.com/blogs/containers/using-amazon-ec2-spot-instances-with-karpenter/
https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html
https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html
https://kubernetes.io/docs/concepts/services-networking/ingress/
