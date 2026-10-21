# 📦 Lab 03: Amazon ECR Container Registry & S3 Storage Buckets

## 🧠 Concept & Architecture ("What & Why")

### What are Amazon ECR and S3?
- **Amazon ECR (Elastic Container Registry)**: A managed AWS Docker container registry with image scanning on push (`scan_on_push = true`) and automated lifecycle policies.
- **Amazon S3**: Object storage used for ALB access logging and Terraform remote state backends.

---

## 📂 Project Structure

```text
01-Infrastructure-Terraform/Lab03-ECR-S3-Storage/
├── main.tf                 # Root Terraform manifest calling ECR & S3 modules
├── outputs.tf              # Exported ECR repository URLs and S3 bucket names
├── provider.tf             # AWS Provider configuration
├── README.md               # Step-by-step lab guide
└── modules/
    ├── alb-logs/           # Dedicated S3 bucket module for ALB access logs
    ├── ecr/                # Amazon ECR container registry module
    └── s3/                 # General S3 storage bucket module
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Amazon ECR Repository (`modules/ecr/main.tf`)

```hcl
resource "aws_ecr_repository" "app" {
  name                 = var.repository_name
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
  }
}

resource "aws_ecr_lifecycle_policy" "policy" {
  repository = aws_ecr_repository.app.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 10 images"
        selection = {
          tagStatus   = "any"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 30
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}
```

---

## 📋 Step-by-Step Execution Guide

### Step 1: Deploy ECR & S3 Infrastructure
```bash
cd 01-Infrastructure-Terraform/Lab03-ECR-S3-Storage
terraform init
terraform apply -auto-approve
```

---

## 🧪 Verification & Expected Terminal Output

```bash
aws ecr describe-repositories --repository-names nti-devops-ecr
```

**Expected Output:**
```json
{
    "repositories": [
        {
            "repositoryName": "nti-devops-ecr",
            "imageScanningConfiguration": {
                "scanOnPush": true
            }
        }
    ]
}
```

---

## 🔗 Connection to Next Lab
In **Lab 04**, we provision the relational database layer (Amazon RDS PostgreSQL 16) and integrate AWS Secrets Manager!
