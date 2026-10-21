# 🌐 Lab 12: Application Routing with K8s Ingress & AWS Load Balancer Controller

## 🧠 Concept & Architecture ("What & Why")

### What is Kubernetes Ingress & AWS Load Balancer Controller?
- **Kubernetes Ingress**: An API object managing external HTTP/HTTPS access to services within a cluster.
- **AWS Load Balancer Controller (LBC)**: An operator running inside EKS that monitors Ingress resources and automatically provisions AWS Application Load Balancers (ALB), Target Groups, and Listener rules on AWS.

```text
                                INGRESS ROUTING ARCHITECTURE
                                
  Client Request (HTTP / Port 80)
         |
         v
  +-----------------------------------------------------------------------------------+
  |  AWS APPLICATION LOAD BALANCER (Provisioned automatically by AWS LBC)             |
  +----------------------------------------+------------------------------------------+
                                           |
                                           | Direct Target Group Routing (target-type: ip)
                                           v
  +-----------------------------------------------------------------------------------+
  |  AWS EKS CLUSTER (Private Subnets)                                                |
  |  Pods receive incoming traffic directly via Amazon VPC CNI                        |
  |                                                                                   |
  |  +-------------------------------+     +---------------------------------+        |
  |  | Pod 1 (IP: 10.0.10.42:8000)   |     | Pod 2 (IP: 10.0.11.89:8000)     |        |
  |  +-------------------------------+     +---------------------------------+        |
  +-----------------------------------------------------------------------------------+
```

---

## 🎯 Objectives

- Create Kubernetes Ingress manifest `ingress.yaml` with AWS ALB annotations.
- Configure path-based routing rules and target-type IP mode (`alb.ingress.kubernetes.io/target-type: ip`).
- Verify automatic provision of AWS Application Load Balancer.

---

## 📂 Project Structure

```text
04-Kubernetes-Orchestration/Lab12-K8s-Ingress-ALB-Controller/
├── ingress.yaml            # Kubernetes Ingress manifest with AWS ALB annotations
└── README.md               # Exhaustive step-by-step lab documentation
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Ingress Manifest (`ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nti-django-ingress
  namespace: nti-devops
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nti-django-svc
                port:
                  number: 8000
```

#### Line-by-Line Breakdown:
- **`scheme: internet-facing`**: Provisions a public-facing AWS Application Load Balancer attached to public subnets.
- **`target-type: ip`**: Routes traffic directly from ALB to Pod IP addresses via Amazon VPC CNI, bypassing NodePort latency.

---

## 📋 Step-by-Step Execution Guide

### Step 1: Apply Ingress Resource
```bash
cd 04-Kubernetes-Orchestration/Lab12-K8s-Ingress-ALB-Controller
kubectl apply -f ingress.yaml
```

---

## 🧪 Verification & Expected Terminal Output

```bash
kubectl get ingress -n nti-devops
```

**Expected Output:**
```text
NAME                 CLASS   HOSTS   ADDRESS                                                                  PORTS   AGE
nti-django-ingress   alb     *       k8s-ntidevops-ntidjang-a1b2c3d4e5-123456789.us-east-1.elb.amazonaws.com   80      2m
```

---

## 🔗 Connection to Next Lab
In **Lab 13**, we harden cluster security using zero-trust NetworkPolicies and configure HorizontalPodAutoscaler (HPA)!
