# 🗄️ Lab 04: Amazon RDS PostgreSQL & AWS Secrets Manager Integration

## 🧠 Concept & Architecture ("What & Why")

### What are Amazon RDS and AWS Secrets Manager?
- **Amazon RDS (Relational Database Service)**: A fully managed relational database service. It handles hardware provisioning, database setup, patching, Multi-AZ high availability replication, and continuous automated backups.
- **AWS Secrets Manager**: A secure secret management service that allows you to replace hardcoded credentials in code with an API call to dynamically retrieve secrets. It automatically generates, encrypts, and rotates database credentials using **AWS KMS (Key Management Service)**.

```text
                                ZERO-TRUST DATABASE ARCHITECTURE
                                
  +-----------------------------------------------------------------------------------+
  |  AWS SECRETS MANAGER                                                              |
  |  Generates & Encrypts Credentials (KMS) -> DB Username, Password, Endpoint       |
  +----------------------------------------+------------------------------------------+
                                           |
                                           | Dynamic Secret Fetch via AWS API
                                           v
  +-----------------------------------------------------------------------------------+
  |  AWS EKS CLUSTER (Private Subnets)                                                |
  |  Django Application Pods (Injects DB Password dynamically into memory)            |
  |                                                                                   |
  |                                        |                                          |
  |                                        | Encrypted TCP Connection (Port 5432)    |
  |                                        v                                          |
  |  +-----------------------------------------------------------------------------+  |
  |  |  AMAZON RDS POSTGRESQL 16 (Private DB Subnet Group — No Public Access)       |  |
  |  |  - Primary Node (us-east-1a) <---> Standby Replica (us-east-1b Multi-AZ)       |  |
  |  +-----------------------------------------------------------------------------+  |
  +-----------------------------------------------------------------------------------+
```

---

## 🎯 Objectives

- Provision a Multi-AZ Amazon RDS PostgreSQL 16 database instance (`modules/rds/main.tf`).
- Enforce strict private network placement (`publicly_accessible = false`) in a dedicated DB Subnet Group (`aws_db_subnet_group`).
- Generate dynamic, random 16-character database passwords using Terraform `random_password`.
- Store database credentials securely in **AWS Secrets Manager** (`modules/secrets-manager/main.tf`).

---

## 📂 Project Structure

```text
01-Infrastructure-Terraform/Lab04-RDS-SecretsManager/
├── main.tf                 # Root Terraform manifest calling RDS & Secrets Manager modules
├── outputs.tf              # Exported Secrets Manager ARN and RDS Endpoint
├── provider.tf             # AWS Provider configuration
├── README.md               # Exhaustive step-by-step lab documentation
└── modules/
    ├── rds/                # Amazon RDS PostgreSQL 16 module
    │   └── main.tf
    └── secrets-manager/    # AWS Secrets Manager module
        └── main.tf
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Amazon RDS PostgreSQL Module (`modules/rds/main.tf`)

```hcl
# 1. DB Subnet Group
resource "aws_db_subnet_group" "main" {
  name        = "${var.project_name}-db-subnet-group"
  subnet_ids  = var.private_subnet_ids
  description = "Subnet group for private RDS database instance"

  tags = {
    Name = "${var.project_name}-db-subnet-group"
  }
}

# 2. Database Parameter Group
resource "aws_db_parameter_group" "pg16" {
  name        = "${var.project_name}-pg16-params"
  family      = "postgres16"
  description = "Custom parameter group for PostgreSQL 16"

  parameter {
    name  = "rds.force_ssl"
    value = "1"
  }
}

# 3. RDS PostgreSQL Instance
resource "aws_db_instance" "postgres" {
  identifier                  = "${var.project_name}-postgres"
  engine                      = "postgres"
  engine_version              = "16.1"
  instance_class              = var.db_instance_class
  allocated_storage           = 20
  max_allocated_storage       = 100
  storage_type                = "gp3"
  storage_encrypted           = true
  db_name                     = var.db_name
  username                    = var.db_username
  password                    = var.db_password
  port                        = 5432
  multi_az                    = true
  publicly_accessible         = false
  db_subnet_group_name        = aws_db_subnet_group.main.name
  vpc_security_group_ids      = [var.rds_security_group_id]
  parameter_group_name        = aws_db_parameter_group.pg16.name
  backup_retention_period     = 7
  deletion_protection         = false
  skip_final_snapshot         = true

  tags = {
    Name = "${var.project_name}-postgres-db"
  }
}
```

#### Line-by-Line Breakdown:
- **`publicly_accessible = false`**: Prevents the database from receiving a public IP address, ensuring it can only be reached internally from inside the VPC.
- **`multi_az = true`**: Provisions a synchronous standby replica in a second Availability Zone for automatic failover.
- **`storage_encrypted = true`**: Encrypts database storage at rest using AWS KMS.
- **`rds.force_ssl = 1`**: Requires all client application connections to use SSL/TLS encryption in transit.

---

### 2. AWS Secrets Manager Module (`modules/secrets-manager/main.tf`)

```hcl
# 1. Random Password Generation
resource "random_password" "db_password" {
  length           = 16
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

# 2. Secrets Manager Secret Container
resource "aws_secretsmanager_secret" "db_credentials" {
  name                    = "${var.project_name}/rds/credentials"
  description             = "Production RDS PostgreSQL database credentials"
  recovery_window_in_days = 0

  tags = {
    Name = "${var.project_name}-db-secret"
  }
}

# 3. Secret Version Content
resource "aws_secretsmanager_secret_version" "db_credentials_val" {
  secret_id = aws_secretsmanager_secret.db_credentials.id

  secret_string = jsonencode({
    engine   = "postgres"
    host     = var.db_endpoint
    port     = 5432
    dbname   = var.db_name
    username = var.db_username
    password = random_password.db_password.result
  })
}
```

---

## 📋 Step-by-Step Execution Guide

### Step 1: Deploy RDS & Secrets Manager
```bash
cd 01-Infrastructure-Terraform/Lab04-RDS-SecretsManager
terraform init
terraform apply -auto-approve
```

### Step 2: Retrieve Secret Values via AWS CLI
```bash
aws secretsmanager get-secret-value   --secret-id nti-devops/rds/credentials   --query SecretString   --output text | jq .
```

---

## 🧪 Verification & Expected Terminal Output

```bash
aws secretsmanager get-secret-value --secret-id nti-devops/rds/credentials
```

**Expected Output:**
```json
{
    "engine": "postgres",
    "host": "nti-devops-postgres.c123456789.us-east-1.rds.amazonaws.com",
    "port": 5432,
    "dbname": "nti_db",
    "username": "nti_admin",
    "password": "SuperSecretRandomPassword123!"
}
```

---

## 🔗 Connection to Next Lab
In **Lab 05**, we provision the Kubernetes cluster layer (Amazon EKS 1.34 and Managed Node Groups)!
