# Deploying a Game on AWS EKS (Elastic Kubernetes Service)
This will be a step-by-step guide to create a Kubernetes cluster using EKS through AWS CLI and deploying a 2048 game on this cluster which then can be accessed by anyone through a URL.

## Tools Used

* **[EKS (Elastic Kubernetes Service)](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html/):** A managed service that you can use to run Kubernetes on AWS without needing to install, operate, and maintain your own Kubernetes control plane or nodes.

* **[Fargate](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html):** AWS Fargate is a technology that you can use with Amazon ECS to run containers without having to manage servers or clusters of Amazon EC2 instances.

* **[VPC (Virtual Private Cloud)](https://docs.aws.amazon.com/toolkit-for-visual-studio/latest/user-guide/vpc-tkv.html):** Amazon Virtual Private Cloud (Amazon VPC) enables you to launch Amazon Web Services resources into a virtual network you've defined. This virtual network resembles a traditional network that you'd operate in your own data center, with the benefits of using the scalable infrastructure of AWS.

* **[IAM (Identity Access Manager)](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html):** AWS Identity and Access Management (IAM) is a web service that helps you securely control access to AWS resources. With IAM, you can centrally manage permissions that control which AWS resources users can access.

---

# Pre-Requisite
**kubectl** – A command line tool for working with Kubernetes clusters.

**eksctl** – A command line tool for working with EKS clusters that automates many individual tasks.

**AWS CLI** – A command line tool for working with AWS services, including Amazon EKS.

---

# Tutorial

## Step-1: Configuring AWS and the terminal
We need to create a new access key by going to `AWS account -> Security Credentials -> Access Key`.

```
aws configure
```

## Step-2: Creating the EKS Cluster
```
eksctl create cluster --name <cluster-name> --region <region-name> --fargate
```

## Step-3: Update the KubeConfig file
```
aws eks update-kubeconfig --name <cluster-name> --region <region-name>
```

## Step-4: Creating a Fargate Profile

```
eksctl create fargateprofile \
    --cluster <cluster-name> \
    --region <region-name> \
    --name alb-sample-app \
    --namespace game-2048
```

## Step-5: Deploy Deployment, Service and Ingress

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

## Step-6: Setup Automatic Load Balancer

### Download the IAM Policy
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```
### Create IAM Policy
```
aws iam create-policy 
    --policy-name AWSLoadBalancerControllerIAMPolicy 
    --policy-document file://iam_policy.json
```
### Create IAM Role (Edit your aws account id)
```
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

## Step-7: Deploy the ALB (Automatic Load Balanacer) Controller

### Add Helm Repo
```
helm repo add eks https://aws.github.io/eks-charts
```
### Update the Repo
```
helm repo update eks
```
### Install the ALB Controller
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region-name> \
  --set vpcId=<vpc-name>
```

## Step-8: Verify the Deployments

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

## Step-9: Get your Address for the deployed Game

```
kubectl get ingress -n game-2048
```

---
# Cleaning Up
```
eksctl delete cluster --name <cluster-name> --region <region-name>
```

















