# terraform-eks-setup

Terraform code to provision an EKS cluster on AWS for the payment-service 
GitOps pipeline project.

## What it creates

- VPC with 2 public subnets across 2 availability zones
- Internet gateway and route tables
- EKS cluster (Kubernetes 1.31)
- Managed node group (t3.small)
- IAM roles for control plane and worker nodes

## Prerequisites

- AWS CLI configured with appropriate permissions
- Terraform >= 1.3

## Usage

```bash
terraform init
terraform plan
terraform apply
```

## After apply

Run these in order after the cluster is up:

```bash
# 1. configure kubectl
aws eks update-kubeconfig \
  --region ap-south-2 \
  --name payment-service-cluster

# 2. verify nodes
kubectl get nodes

# 3. install metrics server (required for HPA)
kubectl apply -f metrics-server.yaml

# 4. install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 5. expose ArgoCD UI
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'
```

## Teardown

```bash
# delete ArgoCD apps first
argocd app delete payment-service --yes

# destroy all infrastructure
terraform destroy -auto-approve
```

Note: terraform destroy does not delete the ArgoCD-created LoadBalancer. 
Delete the ArgoCD app first, otherwise the LB remains and keeps billing.

## Resources created

| Resource | Details |
|----------|---------|
| VPC | 10.0.0.0/16 |
| Subnets | 10.0.1.0/24, 10.0.2.0/24 |
| EKS version | 1.31 |
| Node type | t3.small |
| Node count | 1 (min) - 2 (max) |
| Region | ap-south-2 |

## Related repos

- [payment-service](https://github.com/vinayak432/payment-service) — application code and Jenkinsfile
- [payment-service-gitops](https://github.com/vinayak432/payment-service-gitops) — Helm chart and ArgoCD config
