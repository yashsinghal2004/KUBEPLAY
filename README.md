# 🚀 KUBEPLAY - Deploying 2048 Game on Amazon EKS with Fargate

A comprehensive guide to deploy the popular 2048 game application on Amazon EKS (Elastic Kubernetes Service) using Fargate for serverless compute, with Application Load Balancer (ALB) for external access.


## 🎯 Overview

- **Managed Control Plane**: EKS handles master nodes automatically
- **Serverless Compute**: Fargate manages worker nodes without EC2 overhead
- **External Access**: Application Load Balancer for internet-facing access
- **High Availability**: Multi-AZ deployment with automatic scaling
- **Security**: Private subnets for application pods, public subnets for ALB

---

## 🎥 Demo
Here is the demo of complete project: [Demo Video](https://drive.google.com/file/d/1FopLpLvo4tW01Shg97rCHDPSlfwcxD24/view)

---

## Architecture

```
Internet → ALB → Ingress Controller → Service → Pods (Fargate)
                                    ↓
                              Private Subnets (VPC)
```

This project demonstrates:
- **EKS Cluster Management**: Creating and managing Kubernetes clusters on AWS
- **Fargate Integration**: Using serverless compute for Kubernetes workloads
- **Load Balancer Setup**: Configuring ALB for external access
- **IAM Integration**: Setting up proper permissions for EKS workloads
- **Helm Charts**: Using Helm for Kubernetes application deployment
- **Ingress Controllers**: Managing external traffic routing

## Prerequisites

Before starting, ensure you have the following tools installed:

| Tool | Installation Link | Purpose |
|------|------------------|---------|
| **kubectl** | [Install kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) | Kubernetes command-line tool |
| **eksctl** | [Install eksctl](https://eksctl.io/introduction/) | EKS cluster management tool |
| **AWS CLI** | [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) | AWS command-line interface |
| **Helm** | [Install Helm](https://helm.sh/docs/intro/install/) | Kubernetes package manager |

**Additional Requirements:**
- AWS account with appropriate permissions
- AWS credentials configured (`aws configure`)
- Sufficient IAM permissions for EKS, IAM, and EC2 operations

## ✅ Step-by-Step Deployment

### Step 1: Create EKS Cluster

```bash
eksctl create cluster --name 2048-cluster --region us-east-1 --fargate
```

**What this does:**
- Creates EKS cluster with managed control plane
- Sets up VPC with public/private subnets
- Configures Fargate for serverless compute
- Takes 10-20 minutes to complete

### Step 2: Configure kubectl

```bash
aws eks update-kubeconfig --name 2048-cluster --region us-east-1
```

### Step 3: Create Fargate Profile

```bash
eksctl create fargateprofile \
    --cluster 2048-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

**Note:** Fargate requires profiles for all namespaces where pods will run.

### Step 4: Deploy 2048 Application

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

This creates:
- `game-2048` namespace
- Deployment with 5 replicas
- NodePort service
- Ingress resource (ALB class)

### Step 5: Setup IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider --cluster 2048-cluster --approve
```

### Step 6: Install ALB Controller

#### 6.1 Download IAM Policy
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

#### 6.2 Create IAM Policy
```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

#### 6.3 Create IAM Role
```bash
eksctl create iamserviceaccount \
  --cluster=2048-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<YOUR-AWS-ACCOUNT-ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

**Replace `<YOUR-AWS-ACCOUNT-ID>` with your actual AWS account ID.**

#### 6.4 Install ALB Controller via Helm
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=2048-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<YOUR-VPC-ID>
```

**Replace `<YOUR-VPC-ID>` with your cluster's VPC ID (found in EKS console → Networking tab).**

## Verification

### Check Pod Status
```bash
kubectl get pods -n game-2048
kubectl get pods -n kube-system
```

### Check Services
```bash
kubectl get svc -n game-2048
```

### Check Ingress
```bash
kubectl get ingress -n game-2048
```

**Output:**
```
NAME           CLASS   HOSTS   ADDRESS                                                                  PORTS   AGE
ingress-2048   alb     *       k8s-game2048-ingress2-xxxxx.us-east-1.elb.amazonaws.com   80      1m
```

### Check ALB Controller
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

**Output:**
```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           2m
```

## Access Your Application

1. Wait for the ALB to become active (check EC2 → Load Balancers in AWS Console)
2. Copy the DNS name from the Ingress ADDRESS field
3. Open the URL in your browser
4. Enjoy playing 2048! 🎯

## Cleanup

### Delete the Cluster
```bash
eksctl delete cluster --name 2048-cluster --region us-east-1
```

**Note:** This will delete all resources including VPC, subnets, and load balancers.

## 🚨 Troubleshooting

### Common Issues

#### 1. ALB Controller Pods Not Ready
```bash
# Check pod logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Check deployment status
kubectl describe deployment aws-load-balancer-controller -n kube-system
```

#### 2. Ingress Address Not Showing
- Ensure ALB Controller is running (2/2 replicas)
- Check that IAM OIDC provider is configured
- Verify VPC ID and region in Helm installation

#### 3. Pods Stuck in Pending
- Verify Fargate profile exists for the namespace
- Check namespace configuration
- Ensure cluster has sufficient resources

### Debug Commands
```bash
# Check cluster status
kubectl cluster-info

# Check nodes
kubectl get nodes

# Check namespaces
kubectl get namespaces

# Check all resources in game-2048 namespace
kubectl get all -n game-2048
```

## 📚 Additional Resources

- [EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Fargate for EKS](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

## 📄 License

MIT License © 2025 [Yash Singhal]
