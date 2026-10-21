# 📘 NTI DevOps Platform — Master End-to-End Deployment Guide

This guide provides the complete, step-by-step master sequence for deploying the entire enterprise Cloud DevOps platform from zero on AWS.

---

## 📑 Deployment Sequence Overview

1. [Prerequisites & Workstation Setup](#1-prerequisites--workstation-setup)
2. [Phase 1: Infrastructure as Code (Terraform)](#phase-1-infrastructure-as-code-terraform)
3. [Phase 2: Configuration Management (Ansible)](#phase-2-configuration-management-ansible)
4. [Phase 3: Containerization & Local Development (Docker)](#phase-3-containerization--local-development-docker)
5. [Phase 4: Kubernetes Cluster Workloads (kubectl)](#phase-4-kubernetes-cluster-workloads-kubectl)
6. [Phase 5: Helm Package Management & Add-ons (Helm)](#phase-5-helm-package-management--add-ons-helm)
7. [Phase 6: CI/CD Pipeline Setup (Jenkins)](#phase-6-cicd-pipeline-setup-jenkins)
8. [Phase 7: Monitoring & Observability Stack (Prometheus)](#phase-7-monitoring--observability-stack-prometheus)

---

## 1. Prerequisites & Workstation Setup

Ensure your local workstation has the following tools installed and configured:

```bash
# Verify AWS CLI installation
aws --version

# Verify Terraform installation
terraform version

# Verify Ansible installation
ansible --version

# Verify Docker installation
docker --version

# Verify kubectl installation
kubectl version --client

# Verify Helm installation
helm version
```

Configure AWS CLI credentials with administrator permissions:
```bash
aws configure
```

---

## Phase 1: Infrastructure as Code (Terraform)

Deploy VPC, IAM roles, Security Groups, ECR repository, RDS PostgreSQL instance, AWS Secrets Manager, EKS Cluster, Jenkins EC2 instance, Elastic IP, and AWS Backup Vault.

```bash
cd 01-Infrastructure-Terraform/Lab06-Jenkins-EC2-AWSBackup

# Initialize S3 backend and download AWS provider
terraform init

# Validate configuration syntax
terraform validate

# Preview resources to be created
terraform plan -out=tfplan

# Apply infrastructure creation
terraform apply tfplan
```

Export generated outputs:
```bash
JENKINS_IP=$(terraform output -raw jenkins_public_ip)
EKS_CLUSTER_NAME=$(terraform output -raw eks_cluster_name)
ECR_REPO_URL=$(terraform output -raw ecr_repository_url)

echo "Jenkins Public IP: $JENKINS_IP"
echo "EKS Cluster Name: $EKS_CLUSTER_NAME"
echo "ECR Repo URL: $ECR_REPO_URL"
```

Configure local `kubectl` context to connect to the new EKS cluster:
```bash
aws eks update-kubeconfig --region us-east-1 --name $EKS_CLUSTER_NAME
kubectl get nodes -o wide
```

---

## Phase 2: Configuration Management (Ansible)

Automate remote server setup on the Jenkins EC2 host: install Docker, OpenJDK 17, Git, Jenkins daemon, AWS CLI v2, kubectl, Helm 3, and CloudWatch agent.

```bash
cd 02-Configuration-Management-Ansible/Lab08-Jenkins-Server-Automation

# Update Ansible inventory with Jenkins Public IP
./../Lab07-Ansible-Inventory-Setup/scripts/update-inventory.sh $JENKINS_IP

# Verify SSH connectivity to target host
ansible jenkins_servers -m ping -i ../Lab07-Ansible-Inventory-Setup/inventory/hosts.ini

# Run site playbook to provision all 8 Ansible roles
ansible-playbook -i ../Lab07-Ansible-Inventory-Setup/inventory/hosts.ini playbooks/site.yml

# Execute verification playbook
ansible-playbook -i ../Lab07-Ansible-Inventory-Setup/inventory/hosts.ini playbooks/verify.yml
```

---

## Phase 3: Containerization & Local Development (Docker)

Verify local multi-container orchestration using Docker Compose (Django, PostgreSQL, Redis, Nginx).

```bash
cd 03-Containerization-Docker/Lab10-Nginx-DockerCompose-Stack

# Build and launch local stack in detached mode
docker compose up -d --build

# Verify running container status
docker compose ps

# Test Nginx reverse proxy endpoint
curl http://localhost:80
```

---

## Phase 4: Kubernetes Cluster Workloads (kubectl)

Deploy core workloads to AWS EKS: Namespace, ConfigMap, Secret, Deployment, Service, ALB Ingress, NetworkPolicies, and HPA.

```bash
cd 04-Kubernetes-Orchestration/Lab11-K8s-Workloads-Config
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml -f secret.yaml
kubectl apply -f deployment.yaml -f service.yaml

cd ../Lab12-K8s-Ingress-ALB-Controller
kubectl apply -f ingress.yaml -n nti-devops

cd ../Lab13-K8s-Security-Autoscaling
kubectl apply -f networkpolicy.yaml -n nti-devops
kubectl apply -f horizontalpodautoscaler.yaml -n nti-devops

# Verify running pods and ingress endpoint
kubectl get pods,svc,ingress,hpa -n nti-devops
```

---

## Phase 5: Helm Package Management & Add-ons (Helm)

Deploy AWS Load Balancer Controller and Metrics Server using Helm.

```bash
cd 05-Helm-Package-Management/Lab15-Helm-Cluster-Addons

# Add EKS official Helm repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install AWS Load Balancer Controller
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller   -f releases/aws-load-balancer-controller-values.yaml   -n kube-system

# Install Metrics Server
helm upgrade --install metrics-server eks/metrics-server   -f releases/metrics-server-values.yaml   -n kube-system
```

---

## Phase 6: CI/CD Pipeline Setup (Jenkins)

Access Jenkins Web UI at `http://<JENKINS_IP>:8080`.
1. Retrieve initial admin password:
   ```bash
   ssh ubuntu@$JENKINS_IP "sudo cat /var/lib/jenkins/secrets/initialAdminPassword"
   ```
2. Install recommended plugins.
3. Configure multibranch pipeline pointing to your GitHub repository.
4. Push code changes to trigger the declarative `Jenkinsfile` pipeline execution.

---

## Phase 7: Monitoring & Observability Stack (Prometheus)

Deploy `kube-prometheus-stack`, custom Prometheus alert rules, and Grafana dashboards.

```bash
cd 07-Monitoring-Observability/Lab17-Prometheus-Grafana-Monitoring-Stack
chmod +x install/install.sh
./install/install.sh

cd ../Lab18-Alerts-Grafana-Dashboards
kubectl apply -f alerts/nti-devops-rules.yaml -n monitoring

# Port-forward Grafana interface
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```
Access Grafana at `http://localhost:3000` (User: `admin`, Password: `prom-operator`).
