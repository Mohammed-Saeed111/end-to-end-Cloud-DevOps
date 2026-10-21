# 📦 Lab 14: Creating a Custom Helm Chart for Django Microservices

## 🧠 Concept & Architecture ("What & Why")

### What is Helm & Helm Charts?
**Helm** is the official package manager for Kubernetes (equivalent to `apt` on Ubuntu or `pip` on Python). Managing dozens of raw static Kubernetes YAML manifests across different environments (Development, Staging, Production) leads to code duplication and configuration drift.

A **Helm Chart** is a structured directory package containing Go-template files parameterized by a single configuration file: `values.yaml`.

- **Chart.yaml**: Contains metadata (Chart Name, Version, AppVersion, Description).
- **values.yaml**: Default configuration variables (image repository, tag, replica count, ingress host, secret keys).
- **templates/**: Go-templated Kubernetes manifests rendered dynamically at release time.
- **_helpers.tpl**: Reusable partial template functions (naming conventions, common labels).

```text
                                  HELM CHART ARCHITECTURE
                                  
  +----------------------------------+          +----------------------------------+
  |    VALUES FILE (values.yaml)     |          |   TEMPLATES (templates/*.yaml)   |
  |  replicaCount: 2                 |          |  replicas: {{ .Values.replica }} |
  |  image:                        | +======> |  image: {{ .Values.image.repo }} |
  |    repository: ecr-repo-url      |          |  ...                             |
  |    tag: "1.0.0"                  |          +-----------------+----------------+
  +----------------------------------+                            |
                                                                  | Render Engine
                                                                  v
                                                +----------------------------------+
                                                |  DYNAMIC MANIFESTS APPLIED TO    |
                                                |  AWS EKS CLUSTER (RELEASE)       |
                                                +----------------------------------+
```

---

## 🎯 Objectives

- Build a production-grade custom Helm Chart directory structure under `myapp/`.
- Parameterize `deployment.yaml`, `service.yaml`, `ingress.yaml`, `configmap.yaml`, `secret.yaml`, `networkpolicy.yaml`, and `hpa.yaml`.
- Define template helper functions in `templates/_helpers.tpl`.
- Perform chart linting (`helm lint`), dry-run rendering (`helm template`), installation (`helm install`), and release upgrades (`helm upgrade`).

---

## 📂 Project Structure

```text
05-Helm-Package-Management/Lab14-Custom-Helm-Chart-App/
├── myapp/
│   ├── Chart.yaml              # Helm chart metadata specification
│   ├── values.yaml             # Default configuration values
│   └── templates/              # Go-templated K8s manifests
│       ├── _helpers.tpl        # Template helper macros & label definitions
│       ├── configmap.yaml      # Parameterized ConfigMap
│       ├── deployment.yaml     # Parameterized Deployment
│       ├── hpa.yaml            # Parameterized HPA
│       ├── ingress.yaml        # Parameterized Ingress
│       ├── namespace.yaml      # Target namespace definition
│       ├── networkpolicy.yaml  # Parameterized NetworkPolicy
│       ├── secret.yaml         # Parameterized Secret
│       └── service.yaml        # Parameterized Service
└── README.md                   # Exhaustive step-by-step lab documentation
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Helm Chart Metadata (`myapp/Chart.yaml`)

```yaml
apiVersion: v2
name: myapp
description: A Helm chart for Django Microservice Platform on AWS EKS
type: application
version: 0.1.0
appVersion: "1.0.0"
maintainers:
  - name: NTI DevOps Team
    email: devops@nti.edu.eg
```

#### Line-by-Line Breakdown:
- **`apiVersion: v2`**: Indicates Helm 3 chart format.
- **`version: 0.1.0`**: Semantic versioning (SemVer) of the Helm chart package itself.
- **`appVersion: "1.0.0"`**: Version of the underlying Django container application being deployed.

---

### 2. Default Chart Values (`myapp/values.yaml`)

```yaml
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/nti-devops-ecr
  pullPolicy: Always
  tag: "latest"

nameOverride: ""
fullnameOverride: "nti-django-app"

serviceAccount:
  create: true
  annotations: {}
  name: "nti-django-sa"

podAnnotations: {}
podSecurityContext:
  fsGroup: 1000

securityContext:
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: false
  runAsNonRoot: true
  runAsUser: 1000

service:
  type: ClusterIP
  port: 8000

ingress:
  enabled: true
  className: "alb"
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
  hosts:
    - host: ""
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 6
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

env:
  APP_ENV: production
  DEBUG: "False"
  DB_HOST: nti-postgres.c123456789.us-east-1.rds.amazonaws.com
  DB_PORT: "5432"
  DB_NAME: nti_db
  DB_USER: nti_admin

secretEnv:
  DB_PASSWORD: "SuperSecretProductionPassword123!"
```

---

### 3. Template Helpers Macro (`myapp/templates/_helpers.tpl`)

```gotemplate
{{/*
Expand the name of the chart.
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

---

### 4. Templated Deployment Manifest (`myapp/templates/deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  namespace: nti-devops
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8000
              protocol: TCP
          envFrom:
            - configMapRef:
                name: {{ include "myapp.fullname" . }}-config
            - secretRef:
                name: {{ include "myapp.fullname" . }}-secret
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

---

## 📋 Step-by-Step Execution Guide

### Step 1: Lint Helm Chart Syntax
```bash
cd 05-Helm-Package-Management/Lab14-Custom-Helm-Chart-App
helm lint myapp/
```
*Expected Output:* `1 chart(s) linted, 0 chart(s) failed`

### Step 2: Dry-Run Template Rendering
```bash
helm template myapp myapp/ --debug
```

### Step 3: Install Helm Release to EKS
```bash
helm install myapp myapp/ -n nti-devops
```

### Step 4: Upgrade Release with Custom Image Tag
```bash
helm upgrade myapp myapp/ -n nti-devops --set image.tag="v2.0.0"
```

---

## 🧪 Verification & Expected Terminal Output

```bash
helm list -n nti-devops
```

**Expected Output:**
```text
NAME    NAMESPACE   REVISION    UPDATED                                 STATUS      CHART       APP VERSION
myapp   nti-devops  2           2026-07-27 00:00:00.000000000 +0000 UTC deployed    myapp-0.1.0 1.0.0
```

---

## 💡 Best Practices & Common Troubleshooting

### Enterprise Best Practices:
1. **Always Use `helm lint`**: Run `helm lint` in CI pipelines before deploying.
2. **Atomic Upgrades**: Use `helm upgrade --install --atomic --timeout 5m` to automatically rollback releases if pod health checks fail within 5 minutes.

---

## 🔗 Connection to Next Lab
In **Lab 15**, we manage production EKS cluster add-ons (AWS Load Balancer Controller & Metrics Server) using Helm releases!
