# Micro Services Project

## OpenTelemetry Astronomy Shop Demo

This project contains the Open Telemetry Astronomy Shop, a microservice-based distributed system intended to illustrate the implementation of Open Telemetry in a near real-world environment.

### Project Goals
- Provide a realistic example of a distributed system that can be used to demonstrate OpenTelemetry instrumentation and observability.
- Build a base for vendors, tooling authors, and others to extend and demonstrate their OpenTelemetry integrations.
- Create a living example for OpenTelemetry contributors to use for testing new versions of the API, SDK, and other components or enhancements.

### Supported Languages
Open Telemetry supports many popular programming languages including:
- Python
- Java
- JavaScript / Node.js
- Go
- .NET
- Ruby
- PHP

## Micro Services Architecture

<img width="1166" height="1044" alt="image" src="https://github.com/user-attachments/assets/9ce5f0d0-c669-4a5a-a941-07c3937bed42" />

This architecture illustrates a microservices-based e-commerce architecture using various communication protocols (mainly gRPC, HTTP, and TCP), and integrates tools like Kafka, Envoy, Valkey, and Flagd for key functionalities.

### Top Layer – Entry Points & Proxy
- Internet, Load Generator, React Native App
  - These represent external users or automated testing tools accessing the application via HTTP.
- Frontend Proxy (Envoy)
  - Acts as an API Gateway or Ingress Controller.
  - Routes HTTP traffic to internal services like Frontend, Image Provider, and flagd-ui.
  - Offers load balancing, routing, and security features.

### Frontend Layer
- Frontend
  - Serves the web UI and connects users with backend services.
  - Communicates via gRPC to:
    - Ad, Cart, Checkout, Currency, Recommendation, Product Catalog.
- flagd-ui
  - UI dashboard for feature flag management (likely connects to Flagd).
- Image Provider (nginx)
  - Serves static assets (e.g., product images).

### Core Services
- Checkout
  - Central service in the buying workflow.
  - Communicates with:
    - Shipping, Quote, Email, Currency, Payment, Fraud Detection, Flagd.
- Cart
  - Maintains user carts, backed by Cache (Valkey) and integrated with:
    - Ad, Flagd.
- Ad
  - Provides ad content, possibly for recommendations or marketing.
- Recommendation
  - Suggests products to users based on various data points.
  - Talks to Product Catalog.
- Product Catalog
  - Stores product details, connects to both Recommendation and Frontend.

### Supporting Services
- Cache (Valkey)
  - Used for caching (probably Redis-compatible like Valkey).
  - Supports Cart and Ad.
- queue (Kafka)
  - Event streaming platform.
  - Accepts messages from Checkout, passes them to:
    - Accounting
    - Fraud Detection

### Backend Services
- Accounting
  - Processes financial data from Kafka.
- Fraud Detection
  - Analyzes data from Kafka to detect fraudulent activity.
  - Also talks to Flagd.
- Shipping
  - Handles shipping details; communicates over HTTP and gRPC.
- Quote
  - Generates shipping or price quotes.
- Email
  - Sends confirmation and update emails.
- Currency
  - Converts prices to different currencies.
- Payment
  - Processes transactions, talks to Flagd.

### Feature Flagging
- Flagd
  - Central service managing feature flags.
  - Communicated with by:
    - Cart, Checkout, Fraud Detection, Payment.

### Communication Types
- gRPC – Used extensively for internal service-to-service calls (fast & efficient).
- HTTP – Used for user-facing and REST-based services.
- TCP – Used where event-streaming or lower-level connections are needed (Kafka, service queues).

### Architecture Summary
- Microservices based – each component handles a specific responsibility.
- Event-driven – Kafka is used for decoupling & async processing.
- Service Mesh / Proxy – Envoy acts as the API gateway.
- Caching – via Valkey (Redis-like).
- Observability & Feature Flags – via Flagd and possibly frontend telemetry.
- Platform-Agnostic Access – UI via browser + mobile app (React Native).

### Benefits
- Unified approach to collecting metrics, logs, and traces.
- Vendor-neutral and widely adopted.
- Integrates easily with tools like Prometheus, Grafana, and Jaeger.
- Supports both manual and automatic instrumentation.

## 1. Provisioning EC2 Instance

### Step 1: Launch EC2
- Choose Ubuntu 22.04 LTS
- Instance type: t2.xlarge is a chargeable instance type. Be cautious — make sure to stop or resize it to t2.micro after your practice session to avoid incurring unnecessary charges.
- Open ports in security group:
  - SSH: 22
  - HTTP: 80
  - Jenkins: 8080
  - All port

### Step 2: Connect to Jenkins EC2 server
```bash
ssh -i your-key.pem ubuntu@your-ec2-ip
```

## 2. Install Java (Required by Jenkins)
```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
java -version
```

## 3. Install Jenkins

### Step 1: Add Jenkins Repository
```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
```

### Step 2: Install Jenkins
```bash
sudo apt update
sudo apt install -y jenkins
```

### Step 3: Start and Enable Jenkins
```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

### Step 4: Access Jenkins UI
Go to: http://<your-ec2-ip>:8080

Get the admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## 4. Install Jenkins Plugins
Go to Manage Jenkins → Manage Plugins → Install the following plugins:
- Docker Pipeline
- Kubernetes
- Pipeline
- Blue Ocean (optional)
- Docker
- Kubernetes cli
- Kubernetes credentials

➢ In the tool session add docker installation from dockerhub

## 5. Configure Docker
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo chmod 666 /var/run/docker.sock
```

### Step 2: Add Jenkins User to Docker Group
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

## 6. Configure Docker Credentials in Jenkins

### Step1: Store Docker Credentials in Jenkins
1. Go to Jenkins Dashboard → Manage Jenkins → Credentials
2. Select (global) domain
3. Click Add Credentials
   - Kind: Username with password
   - Username: DockerHub username
   - Password: password
   - ID: docker-cred
   - Description: Docker Hub Credentials

## 8. Create Multibranch Pipeline

### Step 1: Create a New Multibranch Pipeline Job
- Go to Jenkins Dashboard → New Item
- Name: my-multibranch-job
- Type: Multibranch Pipeline

### Step 2: Configure Branch Source
- Choose GitHub
- Add repository URL: https://github.com/jadalaramani/open_telemetry_microservices_CI.git
- Add GitHub credentials (if private)
- Jenkins will scan all branches and create jobs for each branch with a Jenkinsfile

## 9. Example Jenkinsfile (for Docker builds)
Create a Jenkinsfile in your GitHub repo root:
```groovy
pipeline {
    agent any

    stages {
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t ramanijadala/frontend:latest ."
                    }
                }
            }
        }      
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push ramanijadala/frontend:latest "
                    }
                }
            }
        }    
    }
}
```

## 10. Verify Setup
- Push a new branch with a Jenkinsfile. Jenkins automatically creates and runs a pipeline for it
- Docker image should be built and pushed to Docker Hub

## Eks setup in Ubuntu server using Terraform

Terraform code Repository URL: 
https://github.com/adarsh0331/Project_10_Eks_Cluster_with_terraform.git

Complete terraform files to create EKS in AWS VPC is available in the eks-install folder of this repo. This includes remote backend and statelocking implementation as well.

- eks-install: Folder that holds the complete terraform hcl files.
- backend: Folder that holds hcl files for s3 bucket and dynamodb creation.
- modules: Terraform Modules for VPC and EKS.
- main.tf: Main file that invokes the modules to create EKS in VPC.
- variables.tf: Variables for main.tf
- Jenkinfile: Jenkins file to trigger pipeline
- outputs.tf: Output values you wish to see post terraform execution, For example - VPC ID.

### Connect to Jenkins EC2 Server
```bash
ssh -i your-key.pem ubuntu@your-ec2-ip
```

### aws cli:
```bash
sudo apt update
sudo apt install unzip curl -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### kubectl installation:
```bash
curl -LO "https://dl.k8s.io/release/$(curl -sSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version –client
```

### eksctl installation:
```bash
curl -sSL "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" -o eksctl.tar.gz
tar -xzf eksctl.tar.gz
sudo mv eksctl /usr/local/bin/
eksctl version
```

### AWS Configure
Provide your AWS Access Key, Secret Key, region, and output format.

## Storing AWS Access & Secret Keys in Jenkins Using Secret Text

### Step 1: Add AWS Credentials as Secret Text
1. Go to your Jenkins Dashboard
2. Navigate to: Manage Jenkins ➝ Credentials ➝ (global) ➝ Add Credentials

### Step 2: Add 1st Credential (AWS Access Key)
- Kind: Secret text
- Secret: <your AWS access key>
- ID: aws-access-key
- Description: AWS Access Key

Click OK

### Add 2nd Credential (AWS Secret Key)
- Kind: Secret text
- Secret: <your AWS secret key>
- ID: aws-secret-key
- Description: AWS Secret Access Key

Click OK

## Setup Jenkins Pipeline
A. Go to Jenkins ➝ New Item ➝ Pipeline

B. Choose “Pipeline” and name it

C. In the pipeline config, choose:
- Pipeline script from SCM
- SCM: Git
- Repository URL: https://github.com/adarsh0331/Project_10_Eks_Cluster_with_terraform.git
- Script Path: Jenkinsfile

## Trigger the Pipeline
Click “Build Now” in Jenkins to provision the EKS cluster.

### Check:
- EKS cluster in AWS Console
- Nodes are in Ready state
- Jenkins console output shows public IPs and cluster status

### Connect to Jenkins EC2 server
```bash
ssh -i your-key.pem ubuntu@your-ec2-ip
```

## Argocd Installation:
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argocd/stable/manifests/install.yaml
kubectl get pods -n argocd
kubectl get svc -n argocd
kubectl edit svc argocd-server -n argocd            # change type: LoadBalancer
kubectl get svc -n argocd
```

Take the LoadBalancer ip and access it in web

```bash
kubectl get secrets -n argocd
kubectl edit secret argocd-initial-admin-secret -n argocd
echo RDg3ZjNUaDg3S3ZmTDhpbw== | base64 –decode         # -------Replace it with ur secret
```

Username :  admin

Password : <Your passaword>

## Create Application via Web UI
1. Open Argo CD Web UI (e.g., http://<ARGOCD_SERVER>:80)
2. Login with username/password
3. Click “New App”
4. Fill the form:
   - Application Name: opentelemetry-demo
   - Project: default
   - Repository URL: https://github.com/open-telemetry/opentelemetry-demo.git
   - Revision: HEAD
   - Path: Kubernetes/complete
   - Cluster URL: https://kubernetes.default.svc
   - Namespace: argocd
5. Enable Auto-Sync if you want Argo CD to automatically apply changes
6. Click Create

To access the application in the web change frontend-proxy cluster ip to LoadBalancer

## Prometheus & Grafana & Jaeger :

### Helm installation:
```bash
wget https://get.helm.sh/helm-v3.17.2-linux-amd64.tar.gz
tar -xvf helm-v3.17.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

Add OpenTelemetry Helm repository:
```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

To install the chart with the release name my-otel-demo:
```bash
helm install my-otel-demo open-telemetry/opentelemetry-demo
```

TAKE THE FRONTEND-PROXY LOAD-BALANCER IP AND ACCESS PROMETHEUS & GRAFANA IN WEB

## Extra don’t care

### Eks setup in Ubuntu server

#### #AWS CLI Installation
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure
```

#### #Kubectl Installations
```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

#### #Eksctl installation
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

#### # Create cluster
```bash
eksctl create cluster mycluster
```

## 11. Kubernetes Deployment with RBAC (CD Setup)

### Step 1: Create RBAC for Jenkins in Kubernetes
Apply the following YAML to create a ServiceAccount, ClusterRole, and ClusterRoleBinding for Jenkins:

```yaml
#kubectl create namespace ms

#serviceaccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: ms

#Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: ms
rules:
  - apiGroups:
      - ""
      - apps
      - autoscaling
      - batch
      - extensions
      - policy
      - rbac.authorization.k8s.io
    resources:
      - pods
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

#Role binding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: ms
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role
subjects:
- namespace: ms
  kind: ServiceAccount
  name: jenkins

#secret
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: myserviceaccount
```

Apply the YAML:
```bash
kubectl apply -f rbac.yaml
```

```bash
kubectl -n ms describe secret mysecretname
```

### Step 2: Get the Token for Jenkins Authentication
Note down the token — you will use this in Jenkins.

### Step 3: Add Kubeconfig or Token to Jenkins Credentials
In Jenkins:
- Go to Manage Jenkins → Credentials → Global → Add Credentials
  - Kind: Secret text
  - Secret: (Paste the token from above)
  - ID: k8s-cred
  - Description: Kubernetes Cluster Token for CD

### Step 5: Update Jenkinsfile for CD Deployment
```groovy
pipeline {
    agent any

    stages {
        stage('Deploy To Kubernetes') {
            steps {
                   withKubeConfig(caCertificate: '', clusterName: 'eks', contextName: '', credentialsId: 'k8s-cred', namespace: 'ms', restrictKubeConfigAccess: false, serverUrl: 'https://B775572DFEC747C548A81ADFA31189DC.gr7.us-east-1.eks.amazonaws.com') {
                    sh "kubectl apply -f deployment-service.yml"
                     
                }
            }
        }
             
        stage('verify Deployment') {
            steps {
                  withKubeConfig(caCertificate: '', clusterName: 'eks', contextName: '', credentialsId: 'k8s-cred', namespace: 'ms', restrictKubeConfigAccess: false, serverUrl: 'https://B775572DFEC747C548A81ADFA31189DC.gr7.us-east-1.eks.amazonaws.com') {   
                  sh "kubectl get svc -n ms"
                }
            }
        }
    }
}
```
### Step 6: Kubernetes Deployment YAML Example
Link of deployment and service yaml file  
https://github.com/jadalaramani/open_telemetry_microservices/blob/main/deployment-service.yml
