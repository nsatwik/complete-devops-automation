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
                        git clone https://$GIT_USERNAME:$GIT_PASSWORD@github.com/nsatwik/complete-devops-automation.git gitops-repo

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
