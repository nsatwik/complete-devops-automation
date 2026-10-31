# application-code
# CI/CD Pipeline for Microservices with Jenkins, Docker & Argo CD on EKS 
“Designed and implemented a complete CI/CD pipeline for microservices deployment to Amazon EKS using Jenkins, Docker, and Argo CD, following a GitOps workflow with separate source and deployment repositories.”


“This repository demonstrates an end-to-end CI/CD pipeline for building and containerizing microservices using Jenkins and Docker. It uses a separate deployment repository with Argo CD to automatically deploy applications to an Amazon EKS cluster using a GitOps workflow.”

Set up a Jenkins pipeline to build Docker images on every code commit and push them to a container registry such as AWS ECR or Docker Hub.

Organize microservices and Kubernetes manifests in a GitOps-friendly structure using separate repositories.

Configure Argo CD to automatically sync manifests from Git and deploy applications to an Amazon EKS cluster. 🚀

**#Pipeline workflow**

1.Developer pushes code to the Source Repository.
2.Jenkins CI pipeline triggers, runs SonarQube code analysis, builds the Docker image, and pushes it to AWS ECR.
3.Jenkins updates the Kubernetes deployment.yaml with the new image tag and commits the change to the Deployment Repository.
4.Argo CD detects the update, syncs the manifests, and deploys the application to Amazon EKS.

# Final Architecture

```
Developer
   ↓
GitHub (Source Repo)
   ↓
Jenkins Pipeline
   ↓
Docker Build
   ↓
AWS ECR
   ↓
Update Deployment Repo
   ↓
ArgoCD (GitOps)
   ↓
Kubernetes (EKS)
   ↓
LoadBalancer
   ↓
Browser → HTML Website
```

5.The application becomes live on Kubernetes.

## Jenkins Installation

 **Install Jenkins from official jenkins page**

## Docker Installation

Jenkins needs Docker access to build Docker images during the CI pipeline.

### Commands:

```bash
sudo apt-get update
sudo apt install -y docker.io
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

---

## SonarQube Installation

Run SonarQube using Docker (recommended for lab/demo environments).

### Commands:

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:latest
```

**Note:** You must manually generate a SonarQube token after logging into the dashboard

## Sonar Scanner Installation

### Let's install SonarScanner CLI on your Ubuntu EC2 instance and configure it so Jenkins can run SonarQube analysis
**1. Install prerequisites**
Connect to the EC2 instance where you want to run SonarScanner, then execute:
```bash
sudo apt update
sudo apt install -y curl unzip
```

Check Java availability:
```bash
java -version
```

**If Java is missing, install it**
```bash
sudo apt install -y openjdk-21-jre
```

**2. Download and install SonarScanner CLI**
This installs the scanner under /opt:
```bash
cd /tmp
curl -fSL \
  https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-8.2.0.6438-linux-x64.zip \
  -o sonar-scanner.zip
sudo unzip -o sonar-scanner.zip -d /opt
sudo ln -sfn /opt/sonar-scanner-8.2.0.6438-linux-x64 /opt/sonar-scanner
```

**Add it to the system PATH**
```bash
echo 'export PATH="/opt/sonar-scanner/bin:$PATH"' \
  | sudo tee /etc/profile.d/sonar-scanner.sh
source /etc/profile.d/sonar-scanner.sh
```

**3. Verify installation**
Run:
```bash
sonar-scanner --version
which sonar-scanner
```

**You should see the scanner version and a path similar to:**
/opt/sonar-scanner/bin/sonar-scanner


Official installation reference: SonarScanner CLI documentation. 
Sonar Documentation

**4. Make it available to Jenkins**
If Jenkins runs on the same EC2 instance, test the scanner as the Jenkins user:
```bash
sudo -u jenkins /opt/sonar-scanner/bin/sonar-scanner --version
```

---

## Trivy Installation

Trivy is used to scan Docker images for security vulnerabilities.

### **Commands:**

```bash
sudo apt-get update
sudo apt-get install -y wget gnupg lsb-release

# Create keyrings directory
sudo mkdir -p /etc/apt/keyrings

# Download and add Trivy GPG key
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
gpg --dearmor | sudo tee /etc/apt/keyrings/trivy.gpg > /dev/null

# Add Trivy repository
echo "deb [signed-by=/etc/apt/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | \
sudo tee /etc/apt/sources.list.d/trivy.list

# Update packages
sudo apt-get update

# Install Trivy
sudo apt-get install -y trivy
```

---

## Install and Configure kubectl
## 1. Add the Kubernetes package repository
**Run these commands on your Ubuntu EC2 instance**

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.37/deb/Release.key \
  | sudo gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.37/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

**2. Install kubectl**

```bash
sudo apt-get update
sudo apt-get install -y kubectl
```

**If you need to prevent automatic upgrades to a newer minor version**

```bash
sudo apt-mark hold kubectl
```

**3. Verify the version**
```bash
kubectl version --client
which kubectl
apt-cache policy kubectl
```

---

## Connect to Your EKS Cluster

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws configure
aws eks update-kubeconfig --name <cluster-name> --region <region>
kubectl get nodes
```

---

## Install Argo CD in EKS
```bash
kubectl create namespace argocd
```

**Then install the official Argo CD manifests**
```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**This installs the Argo CD server, repository server, application controller and supporting components**
**Step 3 — Verify installation**
```bash
kubectl get pods -n argocd
```

**Wait until the pods are Running and Ready. The application controller is a StatefulSet, so it may take a little longer**

**Also check**
```bash
kubectl get svc -n argocd
```
**Step 4 — Install Argo CD CLI**
```bash
ARCH=amd64
VERSION=3.5.6
curl -fL --retry 3 -o /tmp/argocd "https://github.com/argoproj/argo-cd/releases/download/v${VERSION}/argocd-linux-${ARCH}"
```

**Verify the download**
```bash
file /tmp/argocd
ls -lh /tmp/argocd
```

**If the file exists and is identified as an ELF 64-bit executable, install it**
```bash
install -m 555 /tmp/argocd /usr/local/bin/argocd
argocd version --client
```
**porting**
```bash
kubectl port-forward -n argocd svc/argocd-server 8081:443
```

## Jenkinsfile

### code checkout

```
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/nsatwik/complete-devops-automation.git'
            }
        }
    }
}
```
### Setting the image for build

```
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'mydockerimage'
    }

    stages {

        stage("Checkout") {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/nsatwik/complete-devops-automation.git'
            }
        }

        stage('Set Image Tag') {
            steps {
                script {
                    def shortHash = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${BUILD_NUMBER}-${shortHash}"
                    echo "Using image tag: ${env.IMAGE_TAG}"
                }
            }
        }

    }
}
```
### added the sonarqube stage for securing the application

```
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'mydockerimage'
        SONARQUBE_SERVER = 'SonarQubeServer'
        PATH = "/opt/sonar-scanner/bin:${PATH}"
    }

    stages {

        stage("Checkout") {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/nsatwik/complete-devops-automation.git'
            }
        }

        stage('Set Image Tag') {
            steps {
                script {
                    def shortHash = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${BUILD_NUMBER}-${shortHash}"
                    echo "Using image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh 'sonar-scanner'
                }
            }
        }

    }
}

```
### Build the docker image and Trivy image 

```
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'mydockerimage'
        SONARQUBE_SERVER = 'SonarQubeServer'
        PATH = "/opt/sonar-scanner/bin:${PATH}"
    }

    stages {

        stage("Checkout") {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/nsatwik/complete-devops-automation.git'
            }
        }

        stage('Set Image Tag') {
            steps {
                script {
                    def shortHash = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${BUILD_NUMBER}-${shortHash}"
                    echo "Using image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh "sonar-scanner"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Using updated Image tag ${IMAGE_NAME}:${IMAGE_TAG}"

                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest"
            }
        }

        stage('Image Scan') {
            steps {
                sh "trivy image ${IMAGE_NAME}:${IMAGE_TAG} > report.txt"
            }
        }

    }
}

```

### Push to ECR in Aws

```
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'mydockerimage'
        SONARQUBE_SERVER = 'SonarQubeServer'
        PATH = "/opt/sonar-scanner/bin:${PATH}"
        ECR_REPO = '815018697163.dkr.ecr.ap-south-1.amazonaws.com/hello'
        AWS_REGION = 'ap-south-1'
    }

    stages {

        stage("Checkout") {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/nsatwik/complete-devops-automation.git'
            }
        }

        stage('Set Image Tag') {
            steps {
                script {
                    def shortHash = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${BUILD_NUMBER}-${shortHash}"
                    echo "Using image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh "sonar-scanner"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Using updated Image tag ${IMAGE_NAME}:${IMAGE_TAG}"

                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest"
            }
        }

        stage('Image Scan') {
            steps {
                sh "trivy image ${IMAGE_NAME}:${IMAGE_TAG} > report.txt"
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']]) {
                    sh '''
                        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
                        docker tag $IMAGE_NAME:$IMAGE_TAG $ECR_REPO:$IMAGE_TAG
                        docker push $ECR_REPO:$IMAGE_TAG

                        docker tag $IMAGE_NAME:$IMAGE_TAG $ECR_REPO:latest
                        docker push $ECR_REPO:latest
                    '''
                }
            }
        }

    }
}
```

### Final Jenkinsfile
```
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'mydockerimage'
        SONARQUBE_SERVER = 'SonarQubeServer'
        PATH = "/opt/sonar-scanner/bin:${PATH}"
        ECR_REPO = '815018697163.dkr.ecr.ap-south-1.amazonaws.com/hello'
        AWS_REGION = 'ap-south-1'
        DEPLOYMENT_REPO = 'https://github.com/nsatwik/complete-devops-automation.git'
    }

    stages {

        stage("Checkout") {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/nsatwik/complete-devops-automation.git'
            }
        }

        stage('Set Image Tag') {
            steps {
                script {
                    def shortHash = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${BUILD_NUMBER}-${shortHash}"
                    echo "Using image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh "sonar-scanner"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Using updated Image tag ${IMAGE_NAME}:${IMAGE_TAG}"

                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest"
            }
        }

        stage('Image Scan') {
            steps {
                sh "trivy image ${IMAGE_NAME}:${IMAGE_TAG} > report.txt"
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']]) {
                    sh '''
                        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
                        docker tag $IMAGE_NAME:$IMAGE_TAG $ECR_REPO:$IMAGE_TAG
                        docker push $ECR_REPO:$IMAGE_TAG

                        docker tag $IMAGE_NAME:$IMAGE_TAG $ECR_REPO:latest
                        docker push $ECR_REPO:latest
                    '''
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'git-cred',
                    usernameVariable: 'GIT_USERNAME',
                    passwordVariable: 'GIT_PASSWORD'
                )]) {
                    sh '''
                        rm -rf gitops-repo
                        git clone https://$GIT_USERNAME:$GIT_PASSWORD@github.com/nsatwik/complete-devops-automation.git.git gitops-repo

                        cd gitops-repo/k8s || exit 1

                        echo "Before update:"
                        grep image deployment.yaml

                        sed -i "s#image: ${ECR_REPO}:.*#image: ${ECR_REPO}:${IMAGE_TAG}#g" deployment.yaml

                        echo "After update:"
                        grep image deployment.yaml

                        git config user.email "jenkins@local"
                        git config user.name "Jenkins"

                        git add deployment.yaml
                        git commit -m "Update image tag to ${IMAGE_TAG} (Build ${BUILD_NUMBER})" || echo "No changes to commit"
                        git push origin main
                    '''
                }
            }
        }

    }
}

```
