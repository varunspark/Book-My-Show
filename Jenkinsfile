pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'varunspark/bookmyshow:latest'
        DOCKER_REGISTRY = 'docker.io'
        EKS_CLUSTER_NAME = 'bookMyShow-cluster'
        AWS_REGION = 'us-east-1'
        SONAR_HOST_URL = 'http://sonarqube:9000'
        SONAR_LOGIN_TOKEN = credentials('sonar-token')
    }

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
    }

    stages {
        stage('Clean Workspace') {
            steps {
                script {
                    echo "🧹 Cleaning workspace..."
                    cleanWs()
                }
            }
        }

        stage('Checkout from Git') {
            steps {
                script {
                    echo "📥 Checking out code from GitHub..."
                    git branch: 'main', url: 'https://github.com/varunspark/Book-My-Show.git'
                    sh 'ls -la'
                }
            }
        }

        stage('Display Repository Structure') {
            steps {
                script {
                    echo "📂 Repository Structure:"
                    sh '''
                    echo "=== Root Directory ==="
                    ls -la
                    echo ""
                    echo "=== bookmyshow-app Directory ==="
                    ls -la bookmyshow-app/
                    echo ""
                    echo "=== src Directory ==="
                    ls -la bookmyshow-app/src/
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    echo "🔍 Running SonarQube analysis..."
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
        }

        stage('Quality Gate') {
            steps {
                script {
                    echo "⏳ Waiting for Quality Gate result..."
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    echo "📦 Installing dependencies..."
                    sh '''
                    cd bookmyshow-app
                    ls -la
                    
                    if [ -f package.json ]; then
                        echo "✅ package.json found"
                        rm -rf node_modules package-lock.json
                        npm install --legacy-peer-deps
                        echo "✅ Dependencies installed successfully"
                    else
                        echo "❌ Error: package.json not found in bookmyshow-app!"
                        exit 1
                    fi
                    '''
                }
            }
        }

        stage('Build Application') {
            steps {
                script {
                    echo "🔨 Building React application..."
                    sh '''
                    cd bookmyshow-app
                    npm run build
                    echo "✅ Build completed successfully"
                    '''
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                script {
                    echo "🔐 Running Trivy filesystem security scan..."
                    sh '''
                    trivy fs . --format json -o trivyfs.json || true
                    trivy fs . --format sarif -o trivyfs-sarif.json || true
                    echo "✅ Trivy filesystem scan completed"
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    echo "🐳 Building Docker image..."
                    sh '''
                    docker build \
                        --no-cache \
                        -t ${DOCKER_IMAGE} \
                        -f bookmyshow-app/Dockerfile \
                        bookmyshow-app/
                    
                    echo "✅ Docker image built successfully: ${DOCKER_IMAGE}"
                    '''
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    echo "🔐 Running Trivy container image scan..."
                    sh '''
                    trivy image --format json -o trivyimage.json ${DOCKER_IMAGE} || true
                    trivy image --format sarif -o trivyimage-sarif.json ${DOCKER_IMAGE} || true
                    echo "✅ Trivy image scan completed"
                    '''
                }
            }
        }

        stage('Docker Push to Registry') {
            steps {
                script {
                    echo "📤 Pushing Docker image to registry..."
                    withDockerRegistry(credentialsId: 'docker-credentials', toolName: 'docker') {
                        sh '''
                        echo "🔑 Logged into Docker registry"
                        docker push ${DOCKER_IMAGE}
                        echo "✅ Image pushed successfully to ${DOCKER_REGISTRY}"
                        
                        # Cleanup old images
                        docker rmi ${DOCKER_IMAGE} || true
                        '''
                    }
                }
            }
        }

        stage('Deploy to Docker Container') {
            steps {
                script {
                    echo "🚀 Deploying to Docker container..."
                    sh '''
                    echo "🛑 Stopping and removing old container..."
                    docker stop bookmyshow || true
                    docker rm bookmyshow || true
                    
                    echo "🚀 Starting new container..."
                    docker run -d \
                        --restart=always \
                        --name bookmyshow \
                        -p 3000:3000 \
                        -e NODE_ENV=production \
                        ${DOCKER_IMAGE}
                    
                    echo "✅ Container deployment successful"
                    sleep 5
                    
                    echo "📋 Running container details:"
                    docker ps -a | grep bookmyshow
                    
                    echo "📊 Container logs (last 20 lines):"
                    docker logs --tail 20 bookmyshow || echo "No logs yet"
                    '''
                }
            }
        }

        stage('Smoke Test') {
            steps {
                script {
                    echo "🧪 Running smoke tests..."
                    sh '''
                    echo "⏳ Waiting for application to be ready..."
                    for i in {1..30}; do
                        if curl -f http://localhost:3000/ > /dev/null 2>&1; then
                            echo "✅ Application is up and running!"
                            break
                        fi
                        echo "⏳ Attempt $i/30 - Waiting for application..."
                        sleep 2
                    done
                    
                    if curl -f http://localhost:3000/ > /dev/null 2>&1; then
                        echo "✅ Smoke test PASSED"
                    else
                        echo "❌ Smoke test FAILED - Application not responding"
                        exit 1
                    fi
                    '''
                }
            }
        }

        // Uncomment below stages if you want to deploy to EKS
        /*
        stage('Deploy to EKS Cluster') {
            steps {
                script {
                    echo "☸️ Deploying to EKS cluster..."
                    sh '''
                    echo "🔐 Verifying AWS credentials..."
                    aws sts get-caller-identity
                    
                    echo "⚙️ Updating kubeconfig for EKS..."
                    aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                    
                    echo "📋 Verifying kubectl context..."
                    kubectl config current-context
                    
                    echo "🚀 Creating kubernetes namespace..."
                    kubectl create namespace bookmyshow || true
                    
                    echo "🚀 Deploying application to EKS..."
                    kubectl apply -f bookmyshow-app/k8s/deployment.yml
                    kubectl apply -f bookmyshow-app/k8s/service.yml
                    
                    echo "✅ Waiting for pods to be ready..."
                    kubectl wait --for=condition=ready pod \
                        -l app=bookmyshow \
                        -n bookmyshow \
                        --timeout=300s || true
                    
                    echo "📊 Deployment status:"
                    kubectl get pods -n bookmyshow
                    kubectl get svc -n bookmyshow
                    '''
                }
            }
        }
        */
    }

    post {
        always {
            script {
                echo "📊 Collecting artifacts and generating report..."
                
                // Archive artifacts
                archiveArtifacts artifacts: '**/*.json,**/*.txt,**/*.html',
                                  allowEmptyArchive: true
                
                // Publish Trivy results (if installed)
                sh '''
                if command -v trivy &> /dev/null; then
                    echo "✅ Trivy reports generated"
                fi
                '''
            }
        }

        success {
            script {
                echo "✅ Pipeline execution SUCCESSFUL!"
                emailext(
                    subject: "✅ Build #${BUILD_NUMBER} - ${JOB_NAME} SUCCESS",
                    body: """
                        <h3>✅ Build Successful!</h3>
                        <p>
                            <b>Project:</b> ${JOB_NAME}<br/>
                            <b>Build Number:</b> ${BUILD_NUMBER}<br/>
                            <b>Build Status:</b> SUCCESS<br/>
                            <b>Build URL:</b> <a href="${BUILD_URL}">${BUILD_URL}</a><br/>
                            <b>Docker Image:</b> ${DOCKER_IMAGE}<br/>
                            <b>Duration:</b> ${currentBuild.durationString}
                        </p>
                        <hr/>
                        <p>Application is running at: <b>http://localhost:3000</b></p>
                    """,
                    to: '${DEFAULT_RECIPIENTS}',
                    mimeType: 'text/html',
                    attachLog: true,
                    attachmentsPattern: 'trivyfs.json,trivyimage.json'
                )
            }
        }

        failure {
            script {
                echo "❌ Pipeline execution FAILED!"
                emailext(
                    subject: "❌ Build #${BUILD_NUMBER} - ${JOB_NAME} FAILED",
                    body: """
                        <h3>❌ Build Failed!</h3>
                        <p>
                            <b>Project:</b> ${JOB_NAME}<br/>
                            <b>Build Number:</b> ${BUILD_NUMBER}<br/>
                            <b>Build Status:</b> FAILED<br/>
                            <b>Build URL:</b> <a href="${BUILD_URL}">${BUILD_URL}</a><br/>
                            <b>Duration:</b> ${currentBuild.durationString}
                        </p>
                        <hr/>
                        <p>Please check the logs for more details.</p>
                    """,
                    to: '${DEFAULT_RECIPIENTS}',
                    mimeType: 'text/html',
                    attachLog: true
                )
            }
        }

        unstable {
            script {
                echo "⚠️ Pipeline execution is UNSTABLE (Quality Gate may have failed)"
            }
        }

        cleanup {
            script {
                echo "🧹 Cleaning up..."
                sh '''
                # Remove dangling Docker images
                docker image prune -f || true
                '''
            }
        }
    }
}
