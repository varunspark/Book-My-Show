pipeline {
    agent any

    tools {
        nodejs 'node23'
    }

    environment {
        ECR_REGISTRY = '154810814607.dkr.ecr.ap-south-1.amazonaws.com'
        ECR_REPO = 'bookmyshow'
        IMAGE_TAG = "build-${BUILD_NUMBER}"
        AWS_REGION = 'ap-south-1'
        EKS_CLUSTER_NAME = 'bookmyshow-cluster'
        SCANNER_HOME = tool 'sonar-scanner'
    }

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
    }

    stages {

        stage('Clean Workspace') {
            steps {
                echo "Cleaning workspace..."
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                echo "Checking out code from GitHub..."
                git branch: 'main', url: 'https://github.com/varunspark/Book-My-Show.git'
                sh 'ls -la'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "Running SonarQube analysis..."
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectName=BookMyShow \
                            -Dsonar.projectKey=BookMyShow \
                            -Dsonar.sources=bookmyshow-app/src \
                            -Dsonar.exclusions=node_modules/**,**/*.test.js
                    '''
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                echo "Running Trivy filesystem security scan..."
                sh '''
                    trivy fs bookmyshow-app --format table -o trivyfs.txt || true
                '''
            }
        }

        stage('Docker Build') {
            steps {
                echo "Building Docker image..."
                sh '''
                    docker build \
                        -t ${ECR_REPO}:${IMAGE_TAG} \
                        -f bookmyshow-app/Dockerfile \
                        bookmyshow-app/
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                echo "Running Trivy container image scan..."
                sh '''
                    trivy image --format table -o trivyimage.txt ${ECR_REPO}:${IMAGE_TAG} || true
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                echo "Pushing Docker image to AWS ECR..."
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                    docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:latest

                    docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${ECR_REPO}:latest
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                echo "Deploying to EKS cluster..."
                sh '''
                    aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}

                    kubectl set image deployment/bookmyshow bookmyshow=${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} -n bookmyshow

                    kubectl rollout status deployment/bookmyshow -n bookmyshow --timeout=180s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                echo "Verifying deployment..."
                sh '''
                    kubectl get pods -n bookmyshow
                    kubectl get svc -n bookmyshow
                '''
            }
        }
    }

    post {
        always {
            echo "Archiving scan reports..."
            archiveArtifacts artifacts: 'trivyfs.txt,trivyimage.txt', allowEmptyArchive: true
        }
        success {
            echo "Pipeline SUCCESSFUL! Image ${IMAGE_TAG} deployed to EKS."
        }
        failure {
            echo "Pipeline FAILED. Check logs above for details."
        }
        cleanup {
            sh 'docker image prune -f || true'
        }
    }
}
