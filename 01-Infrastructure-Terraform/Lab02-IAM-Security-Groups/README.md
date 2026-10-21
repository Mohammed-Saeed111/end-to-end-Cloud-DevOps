# 🔐 Lab 02: IAM Roles, IRSA Policies & Security Groups

## 🧠 Concept & Architecture ("What & Why")

### What are IAM Roles & Security Groups?
- **AWS IAM Roles**: Identity permissions granting temporary security credentials to AWS services (such as EKS Worker Nodes or EC2 instances) without embedding static credentials.
- **IRSA (IAM Roles for Service Accounts)**: OIDC integration allowing individual Kubernetes pods to assume specific IAM roles.
- **Security Groups**: Stateful virtual firewalls controlling inbound and outbound traffic at the instance/ENI level.

```text
                                SECURITY GROUP CHAINING TOPOLOGY
                                
  [ Internet Traffic ]
           |
           | HTTP / Port 80
           v
  +-----------------------------------------------------------------------------------+
  |  ALB SECURITY GROUP (sg-alb)                                                      |
  |  - Ingress: 0.0.0.0/0 on Port 80 / 443                                            |
  +----------------------------------------+------------------------------------------+
                                           |
                                           | Ingress: ALLOW sg-alb on Port 8000
                                           v
  +-----------------------------------------------------------------------------------+
  |  EKS WORKER NODES SECURITY GROUP (sg-eks-nodes)                                   |
  |  - Ingress: Allow Port 8000 from sg-alb                                           |
  |  - Ingress: Allow Port 22 from Jenkins SG (sg-jenkins)                            |
  +----------------------------------------+------------------------------------------+
                                           |
                                           | Ingress: ALLOW sg-eks-nodes on Port 5432
                                           v
  +-----------------------------------------------------------------------------------+
  |  RDS DATABASE SECURITY GROUP (sg-rds)                                             |
  |  - Ingress: Allow Port 5432 ONLY from sg-eks-nodes                                |
  +-----------------------------------------------------------------------------------+
```

---

## 🎯 Objectives

- Create least-privilege IAM Roles for EKS Cluster, EKS Node Group, and Jenkins EC2.
- Configure Security Group chaining: ALB ➔ EKS Worker Nodes ➔ RDS PostgreSQL.

---

## 📂 Project Structure

```text
01-Infrastructure-Terraform/Lab02-IAM-Security-Groups/
├── main.tf                 # Root Terraform manifest calling IAM & Security Group modules
├── outputs.tf              # Exported IAM Role ARNs and Security Group IDs
├── provider.tf             # AWS Provider configuration
├── README.md               # Step-by-step lab guide
└── modules/
    ├── iam/                # IAM Roles & OIDC Trust Policies
    └── security-groups/    # Security Group firewall rules
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Security Group Definitions (`modules/security-groups/main.tf`)

```hcl
# 1. ALB Security Group
resource "aws_security_group" "alb" {
  name        = "${var.project_name}-alb-sg"
  description = "Security Group for Application Load Balancer"
  vpc_id      = var.vpc_id

  ingress {
    description = "Allow HTTP inbound"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# 2. EKS Worker Nodes Security Group
resource "aws_security_group" "eks_nodes" {
  name        = "${var.project_name}-eks-nodes-sg"
  description = "Security Group for EKS Worker Nodes"
  vpc_id      = var.vpc_id

  ingress {
    description     = "Allow traffic from ALB"
    from_port       = 8000
    to_port         = 8000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# 3. RDS Database Security Group
resource "aws_security_group" "rds" {
  name        = "${var.project_name}-rds-sg"
  description = "Security Group for RDS PostgreSQL Database"
  vpc_id      = var.vpc_id

  ingress {
    description     = "Allow PostgreSQL access from EKS Nodes"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## 📋 Step-by-Step Execution Guide

### Step 1: Deploy IAM & Security Groups
```bash
cd 01-Infrastructure-Terraform/Lab02-IAM-Security-Groups
terraform init
terraform apply -auto-approve
```

---

## 🧪 Verification & Expected Terminal Output

```bash
terraform output
```

**Expected Terminal Output:**
```text
alb_security_group_id       = "sg-0a1b2c3d4e5f67890"
eks_nodes_security_group_id = "sg-01111111111111111"
rds_security_group_id       = "sg-02222222222222222"
```

---

## 🔗 Connection to Next Lab
In **Lab 03**, we create container storage infrastructure (Amazon ECR) and encrypted S3 buckets!
