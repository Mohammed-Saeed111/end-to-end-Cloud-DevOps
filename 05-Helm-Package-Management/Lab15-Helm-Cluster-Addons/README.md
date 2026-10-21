# ⛵ Lab 15: Managing EKS Cluster Add-ons with Helm & IRSA

## 🧠 Concept & Architecture ("What & Why")

### What are EKS Cluster Add-ons and IRSA?
- **EKS Add-ons**: Core system services running inside the EKS cluster required for infrastructure functionality (such as load balancer automation and pod autoscaling).
- **IRSA (IAM Roles for Service Accounts)**: IAM Roles for Service Accounts allows Kubernetes pods to securely assume AWS IAM roles using OpenID Connect (OIDC) federation. Pods obtain short-lived AWS credentials automatically without embedding AWS access keys inside container environments.

```text
                                IRSA OIDC AUTHENTICATION FLOW
                                
  +-----------------------------------------------------------------------------------+
  |  EKS SERVICE ACCOUNT (aws-load-balancer-controller)                               |
  |  Annotation: eks.amazonaws.com/role-arn = arn:aws:iam::123456789:role/lbc-role     |
  +----------------------------------------+------------------------------------------+
                                           |
                                           | Requests OIDC Token
                                           v
  +-----------------------------------------------------------------------------------+
  |  AWS IAM & OIDC PROVIDER                                                          |
  |  Validates ServiceAccount Token -> Issues Short-Lived AWS STS Credentials         |
  +----------------------------------------+------------------------------------------+
                                           |
                                           | Authorizes AWS API Calls
                                           v
  +-----------------------------------------------------------------------------------+
  |  AWS EC2 / Elastic Load Balancing API                                             |
  |  Provisions / Modifies AWS Application Load Balancer                              |
  +-----------------------------------------------------------------------------------+
```

---

## 🎯 Objectives

- Provision IAM Role for AWS Load Balancer Controller using Terraform IRSA module (`irsa/lbc-irsa.tf`).
- Deploy **Metrics Server** add-on via Helm (`releases/metrics-server-values.yaml`).
- Deploy **AWS Load Balancer Controller** add-on via Helm (`releases/aws-load-balancer-controller-values.yaml`).

---

## 📂 Project Structure

```text
05-Helm-Package-Management/Lab15-Helm-Cluster-Addons/
├── irsa/
│   └── lbc-irsa.tf                                 # IRSA Terraform IAM role definition
├── releases/
│   ├── aws-load-balancer-controller-values.yaml    # Helm values override for LBC
│   └── metrics-server-values.yaml                 # Helm values override for Metrics Server
└── README.md                                       # Exhaustive step-by-step lab documentation
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. AWS Load Balancer Controller Helm Values (`releases/aws-load-balancer-controller-values.yaml`)

```yaml
clusterName: nti-devops-eks

region: us-east-1

vpcId: vpc-0a1b2c3d4e5f67890

serviceAccount:
  create: true
  name: aws-load-balancer-controller
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/nti-devops-lbc-irsa-role

replicaCount: 2

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

#### Line-by-Line Breakdown:
- **`clusterName`**: Registers the target EKS cluster name for AWS API calls.
- **`eks.amazonaws.com/role-arn`**: Attaches the IRSA IAM role granting permissions to manage AWS Application Load Balancers.

---

## 📋 Step-by-Step Execution Guide

### Step 1: Deploy IRSA IAM Role with Terraform
```bash
cd 05-Helm-Package-Management/Lab15-Helm-Cluster-Addons/irsa
terraform init
terraform apply -auto-approve
```

### Step 2: Add Official Helm Repositories
```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

### Step 3: Deploy Metrics Server & AWS LBC Add-ons
```bash
cd ..
helm upgrade --install metrics-server metrics-server/metrics-server -n kube-system -f releases/metrics-server-values.yaml
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system -f releases/aws-load-balancer-controller-values.yaml
```

---

## 🧪 Verification & Expected Terminal Output

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

**Expected Output:**
```text
NAME                                            READY   STATUS    RESTARTS   AGE
aws-load-balancer-controller-7b89c6d4f5-x9z8y   1/1     Running   0          2m
aws-load-balancer-controller-7b89c6d4f5-v1w2u   1/1     Running   0          2m
```

---

## 💡 Best Practices & Common Troubleshooting

### Troubleshooting Matrix:

| Error Symptom | Root Cause | Remediation Command |
|---------------|------------|---------------------|
| `AccessDenied: User is not authorized to perform: elasticloadbalancing` | Missing IRSA ServiceAccount annotation | Check `serviceAccount.annotations` in Helm values and verify IRSA trust policy in AWS IAM. |

---

## 🔗 Connection to Next Lab
In **Lab 16**, we construct the enterprise multibranch CI/CD pipeline in Jenkins with SonarQube quality gates and Trivy container security scans!
