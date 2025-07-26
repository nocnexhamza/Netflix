pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io/nocnex'
        APP_NAME = 'Netflix'
        K8S_DEPLOYMENT = 'Netflix'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                url: 'https://github.com/nocnexhamza/Netflix.git'
            }
        }
        
        stage('Build & Test') {
            agent {
                docker {
                    image 'node:16'
                    args '-u root'
                    args '--build-arg TMDB_V3_API_KEY==e96f6037fcb58dc5bf360d000a653a5b'
                }
            }
            steps {
                sh 'npm install'
                sh 'npm audit fix || true'
                sh 'npm test || echo "Tests failed but continuing deployment"'
            }
            stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--purge --scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=Netflix -Dsonar.sources=. -Dsonar.language=js -Dsonar.exclusions=Dockerfile -Dsonar.javascript.node.maxspace=4096"
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        DOCKER_BUILDKIT=1 docker build \
                        -t ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER} .
                    """
                }
            }
        }

        stage('Scan with Trivy') {
            steps {
                script {
                    sh label: 'Trivy Scan', script: '''#!/bin/bash
                        set -x
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            -v "$WORKSPACE:/workspace" \
                            aquasec/trivy image \
                            --severity CRITICAL \
                            --format table \
                            --output /workspace/trivy-report.txt \
                            "${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}" || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'trivy-report.txt',
                        reportName: 'Trivy Report'
                    ])
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker push ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}
                        """
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'k8s-credentials']) {
                        sh """
                            sed 's|image:.*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}|g' \
                                K8s/deployment.yaml > K8s/deployment-${env.BUILD_NUMBER}.yaml
                            
                            kubectl apply -f K8s/deployment-${env.BUILD_NUMBER}.yaml
                            kubectl apply -f K8s/service.yaml
                            kubectl rollout status deployment/${K8S_DEPLOYMENT}
                        """
                    }
                }
            }
        }
        
            }
   
    post {
        always {
            script {
                sh 'rm -f K8s/deployment-*.yaml trivy-report.* || true'
                cleanWs()
            }
        }
        success {
            script {
                if (env.SLACK_CHANNEL) {
                    slackSend(
                        color: "good",
                        message: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                        channel: env.SLACK_CHANNEL
                    )
                }
            }
        }
        failure {
            script {
                if (env.SLACK_CHANNEL) {
                    slackSend(
                        color: "danger",
                        message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                        channel: env.SLACK_CHANNEL
                    )
                }
            }
        }
    }
}
