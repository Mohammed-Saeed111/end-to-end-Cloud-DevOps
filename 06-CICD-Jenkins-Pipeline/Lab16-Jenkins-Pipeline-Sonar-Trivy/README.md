# 🔄 Lab 16: Enterprise Multibranch CI/CD Pipeline with SonarQube & Trivy

## 🧠 Concept & Architecture ("What & Why")

### What is an Automated Enterprise CI/CD Pipeline?
Continuous Integration and Continuous Delivery (CI/CD) automates software delivery stages from initial developer code check-in to production deployment on EKS:

1. **Checkout & Unit Testing**: Fetches code from GitHub and runs Pytest unit tests.
2. **SonarQube Quality Gate**: Executes static code analysis to detect bugs, code smells, and security vulnerabilities. Blocks deployment if quality thresholds fail.
3. **Docker Multi-Stage Build**: Compiles application image.
4. **Trivy Vulnerability Scan**: Scans container layers for OS and dependency CVE vulnerabilities. Fails build if `CRITICAL` issues exist.
5. **Amazon ECR Push**: Tags image with Git SHA and `latest`, pushing artifact to ECR.
6. **Helm EKS Deployment**: Triggers rolling update on EKS via Helm.

```text
                                AUTOMATED MULTIBRANCH CI/CD PIPELINE
                                
  Developer (git push)
         |
         | GitHub Webhook Trigger
         v
  +-----------------------------------------------------------------------------------+
  |  JENKINS MULTIBRANCH PIPELINE (Jenkinsfile)                                       |
  |                                                                                   |
  |  [ Stage 1: Checkout Code ]                                                       |
  |           |                                                                       |
  |           v                                                                       |
  |  [ Stage 2: Pytest Unit Tests ]                                                   |
  |           |                                                                       |
  |           v                                                                       |
  |  [ Stage 3: SonarQube Static Analysis & Quality Gate ]                              |
  |           | (Quality Gate Passed ✅)                                               |
  |           v                                                                       |
  |  [ Stage 4: Multi-Stage Docker Build & Trivy CVE Security Scan ]                  |
  |           | (Zero CRITICAL CVEs ✅)                                                |
  |           v                                                                       |
  |  [ Stage 5: Tag & Push Container Image to Amazon ECR ]                            |
  |           |                                                                       |
  |           v                                                                       |
  |  [ Stage 6: Helm Upgrade Rolling Deployment to Amazon EKS ]                       |
  +-----------------------------------------------------------------------------------+
```

---

## 🎯 Objectives

- Write declarative `Jenkinsfile` multibranch pipeline script with 6 automated stages.
- Integrate SonarQube static code quality analysis and quality gate checks (`sonar-project.properties`).
- Integrate Trivy container security scanner (`trivy.yaml`, `.trivyignore`) to block images with `HIGH` or `CRITICAL` vulnerabilities.
- Automate Amazon ECR login, image tagging, and zero-downtime Helm rolling deployments on EKS.

---

## 📂 Project Structure

```text
06-CICD-Jenkins-Pipeline/Lab16-Jenkins-Pipeline-Sonar-Trivy/
├── .trivyignore                # Ignored CVE vulnerability exceptions
├── docker-compose.test.yml     # Local pipeline integration testing environment
├── Jenkinsfile                 # Declarative multibranch pipeline specification
├── README.md                   # Exhaustive step-by-step lab documentation
├── sonar-project.properties    # SonarQube scanner properties
└── trivy.yaml                  # Trivy security scanner configuration
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Declarative Jenkins Pipeline (`Jenkinsfile`)

```groovy
pipeline {
    agent any

    environment {
        AWS_REGION     = 'us-east-1'
        ECR_REPO_NAME  = 'nti-devops-ecr'
        APP_NAME       = 'myapp'
        K8S_NAMESPACE  = 'nti-devops'
        IMAGE_TAG      = "${BUILD_NUMBER}"
        AWS_ACCOUNT_ID = credentials('aws-account-id')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ Code checked out successfully from Git."
            }
        }

        stage('Pytest Unit Tests') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r docker/requirements.txt pytest
                    pytest --junitxml=result.xml
                '''
            }
            post {
                always {
                    junit 'result.xml'
                }
            }
        }

        stage('SonarQube Analysis & Quality Gate') {
            steps {
                withSonarQubeEnv('SonarServer') {
                    sh 'sonar-scanner -Dsonar.projectKey=nti-django-app -Dsonar.sources=.'
                }
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build & Trivy Scan') {
            steps {
                sh "docker build -t ${ECR_REPO_NAME}:${IMAGE_TAG} ./docker"
                sh "trivy image --severity HIGH,CRITICAL --exit-code 1 ${ECR_REPO_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Push to Amazon ECR') {
            steps {
                sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                sh "docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}"
                sh "docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest"
                sh "docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}"
                sh "docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest"
            }
        }

        stage('Deploy to EKS via Helm') {
            when {
                branch 'main'
            }
            steps {
                sh "aws eks update-kubeconfig --name nti-devops-eks --region ${AWS_REGION}"
                sh "helm upgrade --install ${APP_NAME} ./05-Helm-Package-Management/Lab14-Custom-Helm-Chart-App/myapp -n ${K8S_NAMESPACE} --set image.tag=${IMAGE_TAG}"
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline Succeeded! Application deployed to EKS."
        }
        failure {
            echo "❌ Pipeline Failed! Check build logs for details."
        }
    }
}
```

#### Line-by-Line Breakdown:
- **`withSonarQubeEnv('SonarServer')`**: Injects SonarQube authentication tokens into `sonar-scanner`.
- **`waitForQualityGate abortPipeline: true`**: Pauses pipeline until SonarQube webhook returns quality gate status. Fails pipeline if code fails thresholds.
- **`trivy image --severity HIGH,CRITICAL --exit-code 1`**: Scans container layers for vulnerabilities. Retuns exit code 1 to abort build if HIGH or CRITICAL CVEs are detected.

---

## 📋 Step-by-Step Execution Guide

### Step 1: Push Code to GitHub Repository
```bash
git add .
git commit -m "feat: trigger enterprise multibranch CI/CD pipeline"
git push origin main
```

### Step 2: Verify EKS Deployment Rollout Status
```bash
kubectl rollout status deployment/nti-django-app -n nti-devops
```

---

## 🧪 Verification & Expected Terminal Output

```bash
kubectl rollout status deployment/nti-django-app -n nti-devops
```

**Expected Output:**
```text
deployment "nti-django-app" successfully rolled out
```

---

## 🔗 Connection to Next Lab
In **Lab 17**, we deploy the production monitoring and observability stack (kube-prometheus-stack) to observe cluster workloads!
