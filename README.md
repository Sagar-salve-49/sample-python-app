# DevOps CI/CD Pipeline Project

# End-to-End CI/CD Pipeline Using Jenkins, Docker, Kubernetes (EKS), Prometheus & Grafana

---

# Project Overview

This project demonstrates a complete DevOps CI/CD pipeline that automates:

- Source code integration from GitHub
- Docker image build process
- Automated testing
- Docker image push to DockerHub
- Deployment to AWS EKS Kubernetes Cluster
- Monitoring using Prometheus and Grafana

The entire workflow is automated using Jenkins Pipeline.

---

# Technologies Used

| Tool | Purpose |
|---|---|
| GitHub | Source Code Management |
| Jenkins | CI/CD Automation |
| Docker | Containerization |
| DockerHub | Container Registry |
| AWS EC2 | Jenkins Server |
| AWS EKS | Kubernetes Cluster |
| Kubernetes | Container Orchestration |
| Prometheus | Monitoring |
| Grafana | Visualization Dashboard |
| Python Flask | Sample Application |
| Java 21 | Jenkins Runtime |

---

# Architecture Diagram

```text
Developer Push Code
        ↓
     GitHub
        ↓
 Jenkins Pipeline
        ↓
 Build Docker Image
        ↓
 Push Image to DockerHub
        ↓
 Deploy to AWS EKS
        ↓
 Kubernetes Deployment
        ↓
 Prometheus + Grafana Monitoring
```

---

# AWS Infrastructure Used

| Resource | Count | Type |
|---|---|---|
| Jenkins EC2 | 1 | t2.medium |
| EKS Worker Nodes | 2 | t3.small |
| EKS Cluster | 1 | Managed |

---

# Project Structure

```text
sample-python-app/
│
├── app.py
├── requirements.txt
├── Dockerfile
├── Jenkinsfile
├── .env.example
├── README.md
│
└── k8s/
    ├── deployment.yaml
    └── service.yaml
```

---

# Step 1: Launch Jenkins EC2 Server

## EC2 Configuration

- AMI: Ubuntu 22.04
- Instance Type: t2.medium
- Storage: 20 GB

## Security Group Ports

| Port | Purpose |
|---|---|
| 22 | SSH |
| 8080 | Jenkins |
| 3000 | Grafana |
| 80 | Application |
| 443 | HTTPS |

---

# Step 2: Install Java 21

```bash
sudo apt update -y
sudo apt install openjdk-21-jdk -y
```

Verify:

```bash
java -version
```

---

# Step 3: Install Jenkins

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
/usr/share/keyrings/jenkins-keyring.asc > /dev/null
```

```bash
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
```

```bash
sudo apt update
sudo apt install jenkins -y
```

Start Jenkins:

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Access Jenkins:

```text
http://<JENKINS_PUBLIC_IP>:8080
```

---

# Step 4: Install Docker

```bash
sudo apt install docker.io -y
```

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Add Docker permissions:

```bash
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins
```

Restart Jenkins:

```bash
sudo systemctl restart jenkins
```

---

# Step 5: Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

```bash
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

---

# Step 6: Install eksctl

```bash
curl --silent --location \
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
| tar xz -C /tmp
```

```bash
sudo mv /tmp/eksctl /usr/local/bin
```

Verify:

```bash
eksctl version
```

---

# Step 7: Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

```bash
sudo apt install unzip -y
```

```bash
unzip awscliv2.zip
```

```bash
sudo ./aws/install
```

Configure AWS CLI:

```bash
aws configure
```

---

# Step 8: Create EKS Cluster

```bash
eksctl create cluster \
--name devops-cluster \
--region eu-north-1 \
--nodegroup-name workers \
--node-type t3.small \
--nodes 2
```

Verify Cluster:

```bash
kubectl get nodes
```

---

# Step 9: Create Flask Application

## app.py

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return "CI/CD Pipeline Running Successfully"

@app.route('/health')
def health():
    return "healthy"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)
```

---

# Step 10: Create requirements.txt

```text
flask
```

---

# Step 11: Create .env.example

```text
APP_PORT=3000
```

---

# Step 12: Create Dockerfile

```dockerfile
FROM python:3.10

WORKDIR /app

COPY requirements.txt .

RUN pip install -r requirements.txt

COPY . .

EXPOSE 3000

CMD ["python", "app.py"]
```

---

# Step 13: Create Kubernetes Deployment

## k8s/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: sample-app

spec:
  replicas: 2

  selector:
    matchLabels:
      app: sample-app

  template:
    metadata:
      labels:
        app: sample-app

    spec:
      containers:
      - name: sample-app
        image: sagarsalve49/sample-python-app:latest

        ports:
        - containerPort: 3000
```

---

# Step 14: Create Kubernetes Service

## k8s/service.yaml

```yaml
apiVersion: v1
kind: Service

metadata:
  name: sample-service

spec:
  type: LoadBalancer

  selector:
    app: sample-app

  ports:
    - port: 80
      targetPort: 3000
```

---

# Step 15: Jenkins Pipeline Configuration

## Jenkinsfile

```groovy
pipeline {

    agent any

    environment {
        IMAGE_NAME = "sagarsalve49/sample-python-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/Sagar-salve-49/sample-python-app.git'
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Test') {
            steps {

                sh '''
                docker run -d --name test-container \
                -p 3000:3000 $IMAGE_NAME:$IMAGE_TAG

                sleep 10

                curl -f http://localhost:3000/health

                docker stop test-container
                docker rm test-container
                '''
            }
        }

        stage('Push Image') {

            steps {

                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'

                    sh 'docker push $IMAGE_NAME:$IMAGE_TAG'

                    sh 'docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest'

                    sh 'docker push $IMAGE_NAME:latest'
                }
            }
        }

        stage('Deploy') {

            steps {

                sh '''
                sed -i "s|image: .*|image: $IMAGE_NAME:$IMAGE_TAG|g" \
                k8s/deployment.yaml

                kubectl apply -f k8s/deployment.yaml

                kubectl apply -f k8s/service.yaml
                '''
            }
        }
    }
}
```

---

# Step 16: Install Jenkins Plugins

Installed Plugins:

- Docker Pipeline
- GitHub Integration
- Kubernetes
- Pipeline
- Credentials Binding

---

# Step 17: Configure DockerHub Credentials

Jenkins:

```text
Manage Jenkins
→ Credentials
→ Global Credentials
→ Add Credentials
```

Credential ID:

```text
dockerhub-creds
```

---

# Step 18: Create Jenkins Pipeline Job

```text
New Item
→ Pipeline
→ Pipeline Script from SCM
→ Git
```

Add GitHub repository URL.

---

# Step 19: Configure GitHub Webhook

GitHub:

```text
Repository
→ Settings
→ Webhooks
```

Payload URL:

```text
http://<JENKINS_PUBLIC_IP>:8080/github-webhook/
```

---

# Step 20: Monitoring Setup

## Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

## Add Prometheus Repository

```bash
helm repo add prometheus-community \
https://prometheus-community.github.io/helm-charts
```

```bash
helm repo update
```

---

## Install Prometheus & Grafana

```bash
helm install monitoring prometheus-community/kube-prometheus-stack
```

---

# Monitoring Features

The following metrics are monitored:

- CPU Usage
- Memory Usage
- Pod Health
- Node Monitoring
- Container Monitoring
- Kubernetes Cluster Health

---

# Important Commands

## Check Pods

```bash
kubectl get pods
```

---

## Check Services

```bash
kubectl get svc
```

---

## Check Nodes

```bash
kubectl get nodes
```

---

## Check Node Metrics

```bash
kubectl top nodes
```

---

## Check Pod Metrics

```bash
kubectl top pods
```

---

# CI/CD Pipeline Flow

```text
Developer Push Code
        ↓
GitHub Repository
        ↓
Jenkins Pipeline Triggered
        ↓
Docker Image Build
        ↓
Application Testing
        ↓
Docker Image Push to DockerHub
        ↓
Deploy to Kubernetes (EKS)
        ↓
Monitoring with Prometheus & Grafana
```

---

# Monitoring Dashboards

## Grafana Dashboards Used

- Kubernetes / Compute Resources / Cluster
- Kubernetes / Compute Resources / Pod
- Kubernetes / Compute Resources / Node (Pods)

These dashboards monitor:

- CPU Utilization
- Memory Utilization
- Pod Health
- Container Status
- Cluster Metrics

---

# Challenges Faced

- AWS IAM authentication issue
- EKS node provisioning issue
- Jenkins Kubernetes authentication issue
- Grafana accessibility issue

---

# Solutions Implemented

- Reconfigured AWS credentials
- Changed node type to t3.small
- Copied kubeconfig to Jenkins user
- Used Grafana port-forwarding

---

# Final Output

## Application URL

```text
http://<LOADBALANCER-URL>
```

---

# Screenshots

## 1. GitHub Repository

(Add Screenshot Here)

---

## 2. Jenkins Pipeline Success

(Add Screenshot Here)

---

## 3. DockerHub Repository

(Add Screenshot Here)

---

## 4. Kubernetes Pods

(Add Screenshot Here)

---

## 5. Kubernetes Services

(Add Screenshot Here)

---

## 6. Running Application

(Add Screenshot Here)

---

## 7. Grafana CPU & Memory Monitoring

(Add Screenshot Here)

---

## 8. Grafana Pod Monitoring

(Add Screenshot Here)

---

# Conclusion

Successfully implemented a complete end-to-end CI/CD pipeline using:

- Jenkins
- Docker
- Kubernetes (EKS)
- Prometheus
- Grafana

The project automates:

- Code Integration
- Containerization
- Automated Testing
- Deployment
- Monitoring

This project demonstrates real-world DevOps automation, CI/CD practices, container orchestration, and monitoring implementation on AWS Cloud.
