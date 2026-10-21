# 🚀 Lab 06: Jenkins EC2 Instance, S3 Backend & AWS Backup Vault

## 🧠 Concept & Architecture ("What & Why")

### What is the Jenkins Host & AWS Backup Architecture?
- **Jenkins Host**: Dedicated EC2 host in Public Subnet A running Jenkins LTS, Docker CE, and CI/CD tools. Attached to a persistent Elastic IP (EIP).
- **AWS Backup Vault**: Fully managed enterprise backup service. Executes automated daily snapshots of the Jenkins EC2 EBS volume at 05:00 UTC with 30-day retention policies.
- **Remote State Backend**: S3 bucket with DynamoDB state locking to prevent concurrent state corruption during multi-developer terraform runs.

```text
                                JENKINS EC2 & DISASTER RECOVERY
                                
  +-----------------------------------------------------------------------------------+
  |  PUBLIC SUBNET A (VPC)                                                            |
  |  - Jenkins EC2 Instance (Elastic IP Attached)                                     |
  |  - EBS Root Volume (gp3, 30GB Encrypted)                                         |
  +----------------------------------------+------------------------------------------+
                                           |
                                           | Daily Automated Snapshot at 05:00 UTC
                                           v
  +-----------------------------------------------------------------------------------+
  |  AWS BACKUP VAULT (nti-devops-backup-vault)                                       |
  |  - Rule: Daily Snapshots                                                          |
  |  - Retention: 30 Days                                                             |
  |  - Encryption: KMS Key                                                            |
  +-----------------------------------------------------------------------------------+
```

---

## 🎯 Objectives

- Configure S3 remote state backend (`backend.tf`) with DynamoDB lock table.
- Provision Jenkins EC2 host (`modules/ec2-jenkins/main.tf`) with Elastic IP.
- Configure AWS Backup Vault and daily Backup Plan (`modules/backup/main.tf`).

---

## 📂 Project Structure

```text
01-Infrastructure-Terraform/Lab06-Jenkins-EC2-AWSBackup/
├── backend.tf              # Remote S3 state backend & DynamoDB state locking
├── main.tf                 # Master root Terraform manifest connecting all modules
├── outputs.tf              # Master exported outputs (Jenkins IP, EKS Endpoint, RDS Endpoint)
├── provider.tf             # AWS Provider setup
├── README.md               # Step-by-step lab documentation
├── terraform.tfvars        # Global input variable values
└── modules/
    ├── backup/             # AWS Backup Vault & Plan module
    └── ec2-jenkins/        # Jenkins EC2 Instance & EIP module
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Master Remote State Backend (`backend.tf`)

```hcl
terraform {
  backend "s3" {
    bucket         = "nti-devops-tfstate-12345"
    key            = "platform/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "nti-devops-tfstate-lock"
    encrypt        = true
  }
}
```

---

### 2. AWS Backup Vault & Daily Backup Plan (`modules/backup/main.tf`)

```hcl
# 1. AWS Backup Vault
resource "aws_backup_vault" "main" {
  name        = "${var.project_name}-backup-vault"
  kms_key_arn = var.kms_key_arn

  tags = {
    Name = "${var.project_name}-backup-vault"
  }
}

# 2. Daily Backup Plan
resource "aws_backup_plan" "daily" {
  name = "${var.project_name}-daily-backup-plan"

  rule {
    rule_name         = "daily-backup-rule"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 5 * * ? *)"

    lifecycle {
      delete_after_in_days = 30
    }
  }
}

# 3. Resource Selection (Jenkins EC2)
resource "aws_backup_selection" "jenkins" {
  iam_role_arn = var.backup_iam_role_arn
  name         = "${var.project_name}-jenkins-backup-selection"
  plan_id      = aws_backup_plan.daily.id

  resources = [
    var.jenkins_instance_arn
  ]
}
```

---

## 📋 Step-by-Step Execution Guide

### Step 1: Initialize S3 Remote Backend & Apply Master Infrastructure
```bash
cd 01-Infrastructure-Terraform/Lab06-Jenkins-EC2-AWSBackup
terraform init
terraform apply -auto-approve
```

---

## 🧪 Verification & Expected Terminal Output

```bash
terraform output
```

**Expected Output:**
```text
jenkins_public_ip   = "54.210.142.88"
eks_cluster_name    = "nti-devops-eks"
rds_endpoint        = "nti-devops-postgres.c123456789.us-east-1.rds.amazonaws.com"
```

---

## 🔗 Connection to Next Lab
In **Lab 07**, we configure Ansible to connect to `jenkins_public_ip` over SSH and set up automated inventory management!
