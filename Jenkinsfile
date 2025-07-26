// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml """
                apiVersion: v1
                kind: Pod
                spec:
                  serviceAccountName: jenkins-agent
                  containers:
                  - name: docker
                    image: docker:24-dind
                    securityContext:
                      privileged: true
                    volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock
                  - name: kubectl
                    image: bitnami/kubectl:latest
                    command:
                    - cat
                    tty: true
                  - name: helm
                    image: alpine/helm:latest
                    command:
                    - cat
                    tty: true
                  - name: sonar-scanner
                    image: sonarsource/sonar-scanner-cli:latest
                    command:
                    - cat
                    tty: true
                  - name: trivy
                    image: aquasec/trivy:latest
                    command:
                    - cat
                    tty: true
                  - name: aws-cli
                    image: amazon/aws-cli:latest
                    command:
                    - cat
                    tty: true
                  volumes:
                  - name: docker-sock
                    hostPath:
                      path: /var/run/docker.sock
            """
        }
    }
    
    environment {
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        FRONTEND_REPO = "${ECR_REGISTRY}/todo-app/frontend"
        BACKEND_REPO = "${ECR_REGISTRY}/todo-app/backend"
        CLUSTER_NAME = 'todo-app-cluster'
        NAMESPACE = 'todo-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
        SONAR_PROJECT_KEY = 'todo-app'
        SONAR_HOST_URL = 'http://sonarqube:9000'
    }
    
    options {
        buildDiscarder(logRotator(
            numToKeepStr: '10',
            daysToKeepStr: '30',
            artifactNumToKeepStr: '5',
            artifactDaysToKeepStr: '7'
        ))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    checkout scm
                    env.GIT_COMMIT = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                }
            }
        }
        
        stage('Code Quality Analysis') {
            parallel {
                stage('SonarQube Analysis') {
                    steps {
                        container('sonar-scanner') {
                            withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                                sh '''
                                    sonar-scanner \
                                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                        -Dsonar.sources=. \
                                        -Dsonar.exclusions=**/node_modules/**,**/coverage/**,**/*.test.js \
                                        -Dsonar.host.url=${SONAR_HOST_URL} \
                                        -Dsonar.login=${SONAR_TOKEN} \
                                        -Dsonar.projectVersion=${BUILD_NUMBER} \
                                        -Dsonar.scm.revision=${GIT_COMMIT}
                                '''
                            }
                        }
                    }
                }
                
                stage('Security Scan - Source Code') {
                    steps {
                        container('trivy') {
                            sh '''
                                trivy fs --security-checks vuln,config,secret \
                                    --severity HIGH,CRITICAL \
                                    --format sarif \
                                    --output trivy-source-results.sarif \
                                    .
                            '''
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: '.',
                                reportFiles: 'trivy-source-results.sarif',
                                reportName: 'Trivy Source Code Security Report'
                            ])
                        }
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        def qualityGate = waitForQualityGate()
                        if (qualityGate.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qualityGate.status}"
                        }
                    }
                }
            }
        }
        
        stage('Build Docker Images') {
            parallel {
                stage('Build Frontend') {
                    steps {
                        container('docker') {
                            dir('app/frontend') {
                                sh '''
                                    docker build \
                                        --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
                                        --build-arg VCS_REF=${GIT_COMMIT} \
                                        --build-arg VERSION=${IMAGE_TAG} \
                                        -t ${FRONTEND_REPO}:${IMAGE_TAG} \
                                        -t ${FRONTEND_REPO}:latest \
                                        .
                                '''
                            }
                        }
                    }
                }
                
                stage('Build Backend') {
                    steps {
                        container('docker') {
                            dir('app/backend') {
                                sh '''
                                    docker build \
                                        --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
                                        --build-arg VCS_REF=${GIT_COMMIT} \
                                        --build-arg VERSION=${IMAGE_TAG} \
                                        -t ${BACKEND_REPO}:${IMAGE_TAG} \
                                        -t ${BACKEND_REPO}:latest \
                                        .
                                '''
                            }
                        }
                    }
                }
            }
        }
        
        stage('Security Scan - Docker Images') {
            parallel {
                stage('Scan Frontend Image') {
                    steps {
                        container('trivy') {
                            sh '''
                                trivy image \
                                    --severity HIGH,CRITICAL \
                                    --format sarif \
                                    --output trivy-frontend-results.sarif \
                                    ${FRONTEND_REPO}:${IMAGE_TAG}
                                
                                # Also create a human-readable report
                                trivy image \
                                    --severity HIGH,CRITICAL \
                                    --format table \
                                    ${FRONTEND_REPO}:${IMAGE_TAG} | tee trivy-frontend-report.txt
                            '''
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: '.',
                                reportFiles: 'trivy-frontend-results.sarif',
                                reportName: 'Trivy Frontend Image Security Report'
                            ])
                        }
                    }
                }
                
                stage('Scan Backend Image') {
                    steps {
                        container('trivy') {
                            sh '''
                                trivy image \
                                    --severity HIGH,CRITICAL \
                                    --format sarif \
                                    --output trivy-backend-results.sarif \
                                    ${BACKEND_REPO}:${IMAGE_TAG}
                                
                                # Also create a human-readable report
                                trivy image \
                                    --severity HIGH,CRITICAL \
                                    --format table \
                                    ${BACKEND_REPO}:${IMAGE_TAG} | tee trivy-backend-report.txt
                            '''
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: '.',
                                reportFiles: 'trivy-backend-results.sarif',
                                reportName: 'Trivy Backend Image Security Report'
                            ])
                        }
                    }
                }
            }
        }
        
        stage('Push to ECR') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    branch 'staging'
                }
            }
            steps {
                container('aws-cli') {
                    withCredentials([
                        [
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: 'aws-credentials'
                        ]
                    ]) {
                        sh '''
                            # Login to ECR
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            
                            # Push frontend image
                            docker push ${FRONTEND_REPO}:${IMAGE_TAG}
                            docker push ${FRONTEND_REPO}:latest
                            
                            # Push backend image
                            docker push ${BACKEND_REPO}:${IMAGE_TAG}
                            docker push ${BACKEND_REPO}:latest
                            
                            echo "Images pushed successfully:"
                            echo "Frontend: ${FRONTEND_REPO}:${IMAGE_TAG}"
                            echo "Backend: ${BACKEND_REPO}:${IMAGE_TAG}"
                        '''
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    branch 'staging'
                }
            }
            steps {
                container('helm') {
                    withCredentials([
                        [
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: 'aws-credentials'
                        ]
                    ]) {
                        script {
                            def environment = 'dev'
                            if (env.BRANCH_NAME == 'main') {
                                environment = 'prod'
                            } else if (env.BRANCH_NAME == 'staging') {
                                environment = 'staging'
                            }
                            
                            sh '''
                                # Configure kubectl
                                aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}
                                
                                # Create namespace if it doesn't exist
                                kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                                
                                # Deploy using Helm
                                helm upgrade --install todo-app ./helm/todo-app \
                                    --namespace ${NAMESPACE} \
                                    --set frontend.image.repository=${FRONTEND_REPO} \
                                    --set frontend.image.tag=${IMAGE_TAG} \
                                    --set backend.image.repository=${BACKEND_REPO} \
                                    --set backend.image.tag=${IMAGE_TAG} \
                                    --set environment=''' + environment + ''' \
                                    --set aws.region=${AWS_REGION} \
                                    --wait \
                                    --timeout=300s
                                
                                # Verify deployment
                                kubectl rollout status deployment/frontend -n ${NAMESPACE} --timeout=300s
                                kubectl rollout status deployment/backend -n ${NAMESPACE} --timeout=300s
                                
                                # Get service information
                                echo "Deployment completed successfully!"
                                kubectl get services -n ${NAMESPACE}
                                kubectl get pods -n ${NAMESPACE}
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Integration Tests') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    branch 'staging'
                }
            }
            steps {
                container('kubectl') {
                    sh '''
                        # Wait for pods to be ready
                        kubectl wait --for=condition=ready pod -l app=frontend -n ${NAMESPACE} --timeout=300s
                        kubectl wait --for=condition=ready pod -l app=backend -n ${NAMESPACE} --timeout=300s
                        
                        # Run basic health checks
                        FRONTEND_POD=$(kubectl get pod -l app=frontend -n ${NAMESPACE} -o jsonpath="{.items[0].metadata.name}")
                        BACKEND_POD=$(kubectl get pod -l app=backend -n ${NAMESPACE} -o jsonpath="{.items[0].metadata.name}")
                        
                        echo "Testing frontend health endpoint..."
                        kubectl exec ${FRONTEND_POD} -n ${NAMESPACE} -- curl -f http://localhost:80/health
                        
                        echo "Testing backend health endpoint..."
                        kubectl exec ${BACKEND_POD} -n ${NAMESPACE} -- curl -f http://localhost:3000/health
                        
                        echo "Integration tests completed successfully!"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            script {
                // Archive artifacts
                archiveArtifacts artifacts: 'trivy-*-results.sarif, trivy-*-report.txt', 
                                fingerprint: true, 
                                allowEmptyArchive: true
                
                // Clean up workspace
                cleanWs()
            }
        }
        
        success {
            echo 'Pipeline completed successfully!'
            // Send success notification (Slack, email, etc.)
        }
        
        failure {
            echo 'Pipeline failed!'
            // Send failure notification
        }
        
        unstable {
            echo 'Pipeline completed with warnings!'
            // Send warning notification
        }
    }
}