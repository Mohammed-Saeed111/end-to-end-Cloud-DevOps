# 🌐 Lab 01: AWS VPC, Subnets, Gateways & Route Tables

## 🧠 Concept & Architecture ("What & Why")

### What is an AWS Virtual Private Cloud (VPC)?
An **AWS Virtual Private Cloud (VPC)** is a logically isolated virtual network dedicated to your AWS account. It gives you complete control over your virtual networking environment, including selection of your own IP address range, creation of subnets, and configuration of route tables and network gateways.

### Public vs. Private Subnet Topology Architecture
Production enterprise cloud architecture mandates strict network boundary isolation:
- **Public Subnets**: Connected directly to an **Internet Gateway (IGW)**. Resources placed here (such as Application Load Balancers or Bastion Hosts) have public IP addresses and can receive inbound internet traffic.
- **Private Subnets**: Isolated from direct public internet access. Compute workloads (such as EKS Worker Nodes and RDS PostgreSQL databases) reside here. Outbound internet access for software patching or container pulls is routed securely through a **NAT Gateway** located in a Public Subnet.

```text
                                  AWS REGION (us-east-1)
                                  
  +-----------------------------------------------------------------------------------+
  |  VPC: nti-devops-vpc (CIDR: 10.0.0.0/16)                                          |
  |                                                                                   |
  |  +-----------------------------------------------------------------------------+  |
  |  |  PUBLIC SUBNETS (Connected to Internet Gateway)                            |  |
  |  |  +-------------------------------+     +---------------------------------+  |  |
  |  |  | Public Subnet A (us-east-1a)   |     | Public Subnet B (us-east-1b)    |  |  |
  |  |  | CIDR: 10.0.1.0/24               |     | CIDR: 10.0.2.0/24               |  |  |
  |  |  | - AWS Application LB (ALB)    |     | - NAT Gateway (Elastic IP)      |  |  |
  |  |  | - Jenkins EC2 (Public IP)     |     |                                 |  |  |
  |  |  +---------------+---------------+     +----------------+----------------+  |  |
  |  +------------------|--------------------------------------|-------------------+  |
  |                     | Routing                              | Outbound Egress     |
  |                     v                                      v                     |
  |  +-----------------------------------------------------------------------------+  |
  |  |  PRIVATE SUBNETS (No Public IPs — Routed via NAT Gateway)                   |  |
  |  |  +-------------------------------+     +---------------------------------+  |  |
  |  |  | Private Subnet A (us-east-1a)  |     | Private Subnet B (us-east-1b)   |  |  |
  |  |  | CIDR: 10.0.10.0/24              |     | CIDR: 10.0.11.0/24              |  |  |
  |  |  | - EKS Managed Worker Nodes    |     | - RDS PostgreSQL Database       |  |  |
  |  |  +-------------------------------+     +---------------------------------+  |  |
  |  +-----------------------------------------------------------------------------+  |
  +-----------------------------------------------------------------------------------+
```

---

## 🎯 Objectives

- Write modular Infrastructure as Code (IaC) using Terraform (`>= 1.6`).
- Provision a custom AWS VPC (`10.0.0.0/16`) across 2 Availability Zones (`us-east-1a`, `us-east-1b`).
- Create 2 Public Subnets (`10.0.1.0/24`, `10.0.2.0/24`) and 2 Private Subnets (`10.0.10.0/24`, `10.0.11.0/24`).
- Attach an Internet Gateway (`aws_internet_gateway`), allocate an Elastic IP (`aws_eip`), and configure a NAT Gateway (`aws_nat_gateway`).
- Establish distinct Public and Private Route Tables (`aws_route_table`) with automatic subnet associations.

---

## 📂 Project Structure

```text
01-Infrastructure-Terraform/Lab01-AWS-VPC-Networking/
├── main.tf                 # Root Terraform manifest calling VPC module
├── outputs.tf              # Exported VPC IDs, subnet IDs, and NAT gateway IPs
├── provider.tf             # AWS Provider & version constraints
├── README.md               # Exhaustive step-by-step lab documentation
├── terraform.tfvars        # Input variable value specifications
├── variables.tf            # Input variable definitions
└── modules/
    └── vpc/                # Reusable Terraform VPC module
        ├── main.tf         # Module resources (VPC, Subnets, IGW, NAT, Routes)
        ├── outputs.tf      # Module outputs
        └── variables.tf    # Module input variables
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Terraform Provider Configuration (`provider.tf`)

```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = "Production"
      Project     = "NTI-DevOps-Final-Project"
      ManagedBy   = "Terraform"
    }
  }
}
```

#### Line-by-Line Breakdown:
- **`required_version = ">= 1.6.0"`**: Enforces a minimum Terraform binary version.
- **`source = "hashicorp/aws"`**: Downloads official AWS provider from HashiCorp Registry.
- **`default_tags`**: Automatically applies standard metadata tags (`Environment`, `Project`, `ManagedBy`) to every AWS resource created in this module.

---

### 2. VPC Module Resource Definition (`modules/vpc/main.tf`)

```hcl
# 1. AWS VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# 2. Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

# 3. Public Subnets
resource "aws_subnet" "public" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name                                           = "${var.project_name}-public-subnet-${count.index + 1}"
    "kubernetes.io/role/elb"                        = "1"
    "kubernetes.io/cluster/${var.project_name}-eks" = "shared"
  }
}

# 4. Elastic IP for NAT Gateway
resource "aws_eip" "nat" {
  domain     = "vpc"
  depends_on = [aws_internet_gateway.main]

  tags = {
    Name = "${var.project_name}-nat-eip"
  }
}

# 5. NAT Gateway
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id

  tags = {
    Name = "${var.project_name}-nat-gw"
  }

  depends_on = [aws_internet_gateway.main]
}

# 6. Private Subnets
resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name                                           = "${var.project_name}-private-subnet-${count.index + 1}"
    "kubernetes.io/role/internal-elb"               = "1"
    "kubernetes.io/cluster/${var.project_name}-eks" = "shared"
  }
}

# 7. Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

# 8. Private Route Table
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }

  tags = {
    Name = "${var.project_name}-private-rt"
  }
}

# 9. Public Route Table Associations
resource "aws_route_table_association" "public" {
  count          = length(var.public_subnet_cidrs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# 10. Private Route Table Associations
resource "aws_route_table_association" "private" {
  count          = length(var.private_subnet_cidrs)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

#### Line-by-Line Breakdown:
- **`enable_dns_hostnames = true`**: Enables public DNS hostname assignment for EC2 instances in this VPC.
- **`map_public_ip_on_launch = true`**: Ensures instances launched into public subnets automatically get public IP addresses.
- **`kubernetes.io/role/elb = 1`**: Mandatory AWS EKS tag letting the AWS Load Balancer Controller auto-discover public subnets for external ALBs.
- **`kubernetes.io/role/internal-elb = 1`**: Mandatory EKS tag for discovering private subnets for internal load balancers.
- **`domain = "vpc"`**: Allocates an Elastic IP inside VPC scope for NAT Gateway egress.

---

## 📋 Step-by-Step Execution Guide

### Step 1: Initialize Terraform Working Directory
```bash
cd 01-Infrastructure-Terraform/Lab01-AWS-VPC-Networking
terraform init
```
*What happens under the hood?* Downloads the AWS Provider plugin (`~> 5.0`) into `.terraform/` and initializes local backend state.

### Step 2: Validate Syntax and Format
```bash
terraform fmt -check
terraform validate
```

### Step 3: Generate Execution Plan
```bash
terraform plan -out=tfplan
```

### Step 4: Apply Infrastructure Plan
```bash
terraform apply tfplan
```

---

## 🧪 Verification & Expected Terminal Output

```bash
terraform output
```

**Expected Terminal Output:**
```text
vpc_id              = "vpc-0a1b2c3d4e5f67890"
public_subnet_ids   = [
  "subnet-01111111111111111",
  "subnet-02222222222222222"
]
private_subnet_ids  = [
  "subnet-03333333333333333",
  "subnet-04444444444444444"
]
nat_gateway_ip      = "54.210.100.50"
```

---

## 💡 Best Practices & Common Troubleshooting

### Best Practices:
1. **Multi-AZ Subnets**: Spread public and private subnets across at least 2 Availability Zones for high availability.
2. **Explicit EKS Tags**: Always tag subnets with `kubernetes.io/role/elb` to prevent ingress provisioning failures.

### Troubleshooting Matrix:

| Error Symptom | Root Cause | Remediation Command |
|---------------|------------|---------------------|
| `InvalidSubnet.Conflict: CIDR overlaps` | Subnet CIDR block overlaps with an existing subnet | Adjust `public_subnet_cidrs` and `private_subnet_cidrs` in `terraform.tfvars`. |
| `K8s ALB Controller fails to find subnets` | Missing `kubernetes.io/role/elb` tags | Add required Kubernetes tags to public and private subnets in `main.tf`. |

---

## 🔗 Connection to Next Lab
With the underlying network topology established, **Lab 02** defines security groups and IAM roles for EKS, Jenkins, RDS, and IRSA!
