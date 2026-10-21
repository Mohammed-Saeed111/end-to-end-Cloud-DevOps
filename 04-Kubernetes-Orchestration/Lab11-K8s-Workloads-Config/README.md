# ☸️ Lab 11: Core Kubernetes Workloads, ConfigMaps & Secrets

## 🧠 Concept & Architecture ("What & Why")

### What are Core Kubernetes API Workloads?
Kubernetes abstracts application infrastructure into declarative API objects:
- **Namespace**: Provides logical resource isolation (`nti-devops`). Prevents naming collisions and acts as a boundary for RBAC and ResourceQuotas.
- **ConfigMap**: Decouples non-sensitive configuration artifacts (database hostnames, debug flags) from container images.
- **Secret**: Encodes sensitive credentials (passwords, tokens) in Base64 for injection into pods.
- **Deployment**: Declarative controller managing pod creation, scaling, self-healing pod replacements, and zero-downtime rolling updates.
- **Service (ClusterIP)**: Provides a stable, internal virtual IP and DNS name (`nti-django-svc.nti-devops.svc.cluster.local`) load-balancing traffic across dynamic pod IP addresses.

```text
                                KUBERNETES WORKLOAD ARCHITECTURE
                                
  +-----------------------------------------------------------------------------------+
  |  KUBERNETES CLUSTER (AWS EKS)                                                     |
  |  NAMESPACE: nti-devops                                                            |
  |                                                                                   |
  |  +-----------------------+     +-------------------+                              |
  |  |    ConfigMap          |     |      Secret       |                              |
  |  |  (app-config)         |     |   (app-secret)    |                              |
  |  +-----------+-----------+     +---------+---------+                              |
  |              |                           |                                        |
  |              +-------------+-------------+                                        |
  |                            | Injected as Env Vars                                 |
  |                            v                                                      |
  |  +-------------------------------------------------+                              |
  |  |  DEPLOYMENT: nti-django-app (Replicas: 2)       |                              |
  |  |  +--------------------+   +------------------+  |                              |
  |  |  | Pod 1 (Django App) |   | Pod 2 (Django)   |  |                              |
  |  |  +---------+----------+   +--------+---------+  |                              |
  |  +------------|-----------------------|------------+                              |
  |               +-----------+-----------+                                           |
  |                           |                                                       |
  |                           v                                                       |
  |            +------------------------------+                                       |
  |            |   SERVICE (ClusterIP)        |                                       |
  |            |   nti-django-svc:8000        |                                       |
  |            +------------------------------+                                       |
  +-----------------------------------------------------------------------------------+
```

---

## 🎯 Objectives

- Create custom namespace `nti-devops` (`namespace.yaml`).
- Decouple application variables using `ConfigMap` (`configmap.yaml`) and `Secret` (`secret.yaml`).
- Deploy a 2-replica `Deployment` (`deployment.yaml`) with CPU/Memory requests & limits and liveness/readiness health probes.
- Expose workload internally using a `ClusterIP` `Service` (`service.yaml`).

---

## 📂 Project Structure

```text
04-Kubernetes-Orchestration/Lab11-K8s-Workloads-Config/
├── configmap.yaml          # Non-sensitive application configuration
├── deployment.yaml         # Application deployment specification (2 replicas)
├── namespace.yaml          # K8s logical namespace definition (nti-devops)
├── README.md               # Exhaustive step-by-step lab documentation
├── secret.yaml             # Encoded Kubernetes secret data
└── service.yaml            # ClusterIP service definition
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Application Deployment (`deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nti-django-app
  namespace: nti-devops
  labels:
    app: nti-django-app
    app.kubernetes.io/part-of: nti-devops-platform
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nti-django-app
  template:
    metadata:
      labels:
        app: nti-django-app
    spec:
      containers:
        - name: django-app
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/nti-devops-ecr:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
              name: http
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secret
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /
              port: 8000
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5
```

#### Line-by-Line Breakdown:
- **`replicas: 2`**: Maintains 2 running pod instances across worker nodes.
- **`envFrom`**: Maps all key-value pairs from `app-config` and `app-secret` into container environment variables.
- **`resources.requests` vs `limits`**: `requests` reserves guaranteed Node resources (250m CPU, 256Mi RAM); `limits` caps maximum container consumption (500m CPU, 512Mi RAM).
- **`livenessProbe` & `readinessProbe`**: `readinessProbe` prevents unready pods from receiving traffic; `livenessProbe` restarts frozen containers.

---

## 📋 Step-by-Step Execution Guide

### Step 1: Connect to EKS Cluster & Apply Namespace
```bash
aws eks update-kubeconfig --name nti-devops-eks --region us-east-1
cd 04-Kubernetes-Orchestration/Lab11-K8s-Workloads-Config
kubectl apply -f namespace.yaml
```

### Step 2: Apply Workload Manifests
```bash
kubectl apply -f configmap.yaml -f secret.yaml
kubectl apply -f deployment.yaml -f service.yaml
```

---

## 🧪 Verification & Expected Terminal Output

```bash
kubectl get all -n nti-devops
```

**Expected Output:**
```text
NAME                                  READY   STATUS    RESTARTS   AGE
pod/nti-django-app-6d8b9f7c4d-x9y8z   1/1     Running   0          45s
pod/nti-django-app-6d8b9f7c4d-w1v2u   1/1     Running   0          45s

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/nti-django-svc   ClusterIP   10.100.150.42   <none>        8000/TCP   45s

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nti-django-app   2/2     2            2           45s
```

---

## 🔗 Connection to Next Lab
In **Lab 12**, we expose this ClusterIP Service to public internet users via Kubernetes Ingress and the AWS Load Balancer Controller!
