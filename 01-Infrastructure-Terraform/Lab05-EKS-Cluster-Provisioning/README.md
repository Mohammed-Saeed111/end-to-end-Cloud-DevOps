# ☸️ Lab 05: Amazon EKS Cluster & Managed Node Group Provisioning

## 🧠 Concept & Architecture ("What & Why")

### What is Amazon EKS?
**Amazon EKS (Elastic Kubernetes Service)** is a managed Kubernetes service that eliminates the operational burden of deploying and maintaining the Kubernetes Control Plane (API Server, `etcd`, Controller Manager, Scheduler) across multi-AZ availability zones.

- **Managed Node Groups**: AWS handles provisioning, updating, and auto-repairing EC2 instances acting as Kubernetes worker nodes.
- **OIDC Provider**: Enables IRSA (IAM Roles for Service Accounts) so Kubernetes pods can assume IAM roles natively.

```text
                                AMAZON EKS CLUSTER ARCHITECTURE
                                
  +-----------------------------------------------------------------------------------+
  |  AWS MANAGED KUBERNETES CONTROL PLANE (Managed by AWS)                            |
  |  - API Server (kube-apiserver)                                                    |
  |  - High-Availability etcd Key-Value Store                                         |
  |  - Controller Manager & Scheduler                                                 |
  +----------------------------------------+------------------------------------------+
                                           |
                                           | Secure Control Plane Communication
                                           v
  +-----------------------------------------------------------------------------------+
  |  PRIVATE SUBNETS (VPC) — EKS MANAGED NODE GROUP                                   |
  |                                                                                   |
  |  +-------------------------------+     +---------------------------------+        |
  |  | Worker Node 1 (us-east-1a)    |     | Worker Node 2 (us-east-1b)      |        |
  |  | Type: t3.medium               |     | Type: t3.medium                 |        |
  |  | - Kubelet & Containerd        |     | - Kubelet & Containerd          |        |
  |  | - AWS VPC CNI Plugin          |     | - AWS VPC CNI Plugin            |        |
  |  +-------------------------------+     +---------------------------------+        |
  +-----------------------------------------------------------------------------------+
```

---

## 🎯 Objectives

- Provision Amazon EKS Cluster v1.34 (`modules/eks/main.tf`).
- Deploy EKS Managed Node Group with 2 worker nodes (`t3.medium`) scaling up to 6.
- Enable AWS IAM OpenID Connect (OIDC) provider for IRSA authentication.

---

## 📂 Project Structure

```text
01-Infrastructure-Terraform/Lab05-EKS-Cluster-Provisioning/
├── main.tf                 # Root Terraform manifest calling EKS module
├── outputs.tf              # Exported EKS Cluster Endpoint, Name, and OIDC Provider ARN
├── provider.tf             # AWS Provider configuration
├── README.md               # Step-by-step lab documentation
└── modules/
    └── eks/                # EKS Cluster & Managed Node Group module
        └── main.tf
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Amazon EKS Cluster & Node Group (`modules/eks/main.tf`)

```hcl
# 1. EKS Control Plane
resource "aws_eks_cluster" "main" {
  name     = "${var.project_name}-eks"
  role_arn = var.eks_cluster_role_arn
  version  = "1.34"

  vpc_config {
    subnet_ids              = var.private_subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true
    security_group_ids      = [var.eks_cluster_security_group_id]
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]
}

# 2. OIDC Provider for IRSA
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

# 3. EKS Managed Node Group
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-managed-nodes"
  node_role_arn   = var.eks_node_role_arn
  subnet_ids      = var.private_subnet_ids

  scaling_config {
    desired_size = 2
    min_size     = 2
    max_size     = 6
  }

  update_config {
    max_unavailable = 1
  }

  instance_types = ["t3.medium"]
  capacity_type  = "ON_DEMAND"

  labels = {
    role = "general-workloads"
  }

  tags = {
    Name = "${var.project_name}-node-group"
  }
}
```

---

## 📋 Step-by-Step Execution Guide

### Step 1: Deploy EKS Cluster (~15-20 Minutes Creation Time)
```bash
cd 01-Infrastructure-Terraform/Lab05-EKS-Cluster-Provisioning
terraform init
terraform apply -auto-approve
```

### Step 2: Configure Local `kubectl` Context
```bash
aws eks update-kubeconfig --name nti-devops-eks --region us-east-1
```

---

## 🧪 Verification & Expected Terminal Output

```bash
kubectl get nodes
```

**Expected Output:**
```text
NAME                             STATUS   ROLES    AGE     VERSION
ip-10-0-10-42.ec2.internal       Ready    <none>   3m20s   v1.34.0
ip-10-0-11-89.ec2.internal       Ready    <none>   3m18s   v1.34.0
```

---

## 🔗 Connection to Next Lab
In **Lab 06**, we provision the Jenkins CI/CD EC2 instance, state backends, and AWS Backup Vault!
