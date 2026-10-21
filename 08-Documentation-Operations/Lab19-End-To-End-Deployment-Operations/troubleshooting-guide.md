# 🛠️ NTI DevOps Platform — Production Troubleshooting Guide

This handbook provides diagnostic procedures, root cause analysis, and immediate fix commands for common operational errors across the DevOps stack.

---

## 📑 Diagnostic Index

1. [Terraform & Infrastructure Errors](#1-terraform--infrastructure-errors)
2. [Ansible & Server Automation Errors](#2-ansible--server-automation-errors)
3. [Docker & Container Build Errors](#3-docker--container-build-errors)
4. [Kubernetes & EKS Cluster Errors](#4-kubernetes--eks-cluster-errors)
5. [Jenkins & CI/CD Pipeline Errors](#5-jenkins--cicd-pipeline-errors)
6. [Prometheus & Grafana Observability Errors](#6-prometheus--grafana-observability-errors)

---

## 1. Terraform & Infrastructure Errors

### Error 1.1: `Error locking state: Error acquiring the state lock`
- **Symptom**: `terraform plan` or `terraform apply` fails stating state is locked by another process.
- **Root Cause**: A previous Terraform run was interrupted or crashed before releasing the DynamoDB lock table entry.
- **Remediation**:
  ```bash
  # Force unlock using the Lock ID printed in the error message
  terraform force-unlock <LOCK_ID>
  ```

### Error 1.2: `CIDR block overlaps with existing subnet`
- **Symptom**: Terraform fails during VPC subnet creation.
- **Root Cause**: The subnet CIDR range specified in `variables.tf` falls outside or overlaps with an existing subnet range inside `10.0.0.0/16`.
- **Remediation**: Update `public_subnet_cidrs` and `private_subnet_cidrs` in `terraform.tfvars` to non-overlapping `/24` blocks (e.g. `10.0.1.0/24`, `10.0.2.0/24`, `10.0.3.0/24`, `10.0.4.0/24`).

---

## 2. Ansible & Server Automation Errors

### Error 2.1: `Permission denied (publickey)`
- **Symptom**: Ansible fails to connect to target EC2 host over SSH.
- **Root Cause**: Wrong SSH key path in `ansible.cfg`, incorrect `ansible_user` (must be `ubuntu` for Ubuntu AMIs), or insecure SSH key permissions.
- **Remediation**:
  ```bash
  # Fix private key permissions
  chmod 400 ~/.ssh/id_rsa

  # Test manual SSH connection
  ssh -i ~/.ssh/id_rsa ubuntu@<JENKINS_IP>
  ```

### Error 2.2: `Failed to connect to the host via ssh: Connection refused`
- **Symptom**: SSH connection times out or is refused.
- **Root Cause**: Jenkins EC2 Security Group does not allow inbound SSH traffic on port 22 from your current public IP.
- **Remediation**: Update the Security Group ingress rule in Terraform or AWS Console to allow port 22 from your public IP (`curl ifconfig.me`).

---

## 3. Docker & Container Build Errors

### Error 3.1: `django.db.utils.OperationalError: could not connect to server: Connection refused`
- **Symptom**: Django container crashes during `entrypoint.sh` migration execution.
- **Root Cause**: Django started before PostgreSQL database container finished initialization and was ready to accept TCP connections on port 5432.
- **Remediation**: Add a wait-for-it database check loop inside `entrypoint.sh`:
  ```bash
  until nc -z -v -w30 $DB_HOST 5432; do
    echo "Waiting for PostgreSQL database connection..."
    sleep 2
  done
  ```

### Error 3.2: `Permission denied: '/app/staticfiles'`
- **Symptom**: `python manage.py collectstatic` fails inside container.
- **Root Cause**: Container is running as non-root user `appuser`, but `/app` directory ownership belongs to `root`.
- **Remediation**: Ensure `Dockerfile` includes `chown -R appuser:appuser /app` before switching to `USER appuser`.

---

## 4. Kubernetes & EKS Cluster Errors

### Error 4.1: `CrashLoopBackOff` on Pods
- **Symptom**: Application Pods continuously restart.
- **Root Cause**: Unhandled application runtime exception, missing environment variable secret, or failing liveness probe.
- **Remediation**:
  ```bash
  # Inspect container stdout logs
  kubectl logs -n nti-devops <POD_NAME> --previous

  # Describe pod lifecycle events
  kubectl describe pod -n nti-devops <POD_NAME>
  ```

### Error 4.2: `ImagePullBackOff` or `ErrImagePull`
- **Symptom**: Kubernetes cannot pull container image from ECR.
- **Root Cause**: Image tag does not exist in ECR or EKS worker nodes lack IAM policy `AmazonEC2ContainerRegistryReadOnly`.
- **Remediation**: Verify ECR IAM role attachment and check that the image tag exists in ECR:
  ```bash
  aws ecr list-images --repository-name nti-devops-app
  ```

---

## 5. Jenkins & CI/CD Pipeline Errors

### Error 5.1: `SonarQube Quality Gate Failure`
- **Symptom**: Jenkins pipeline fails at stage `SonarQube Quality Gate`.
- **Root Cause**: Code coverage fell below threshold or critical security vulnerabilities were discovered during static code analysis.
- **Remediation**: Review SonarQube dashboard report at `http://<JENKINS_IP>:9000`, fix code vulnerabilities, commit changes, and re-trigger pipeline.

### Error 5.2: `Trivy Vulnerability Scan Exit Code 1`
- **Symptom**: Jenkins pipeline halts during container vulnerability scanning.
- **Root Cause**: Container base image contains `CRITICAL` CVE vulnerabilities.
- **Remediation**: Upgrade base image in `Dockerfile` (e.g. update `python:3.13-slim` to latest patch version) or add verified exceptions to `.trivyignore`.

---

## 6. Prometheus & Grafana Observability Errors

### Error 6.1: `Grafana Dashboard Panel Shows No Data`
- **Symptom**: Custom Grafana panels display empty graphs.
- **Root Cause**: Prometheus `ServiceMonitor` failed to discover application metrics endpoint due to non-matching label selectors.
- **Remediation**: Verify that `ServiceMonitor` label selector matches application service labels:
  ```bash
  kubectl get servicemonitors -n monitoring -o yaml
  ```
