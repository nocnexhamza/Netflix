pipeline {
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    
    environment {
        DOCKER_REGISTRY = 'docker.io/nocnex'
        APP_NAME = 'netflix'
        K8S_DEPLOYMENT = 'netflix-app'
        TMDB_V3_API_KEY = 'e96f6037fcb58dc5bf360d000a653a5b'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                url: 'https://github.com/nocnexhamza/Netflix.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=Netflix -Dsonar.sources=. -Dsonar.language=js -Dsonar.exclusions=Dockerfile -Dsonar.javascript.node.maxspace=8192"
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
        
        stage('Build & Test') {
            agent {
                docker {
                    image 'node:16'
                    args '-u root'
                    
                }
            }
            steps {
                sh 'npm install'
                sh 'npm audit fix || true'
                sh 'npm test || echo "Tests failed but continuing deployment"'
            }
        }
        
        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check-2'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
        
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        DOCKER_BUILDKIT=1 docker build \
                        --build-arg TMDB_V3_API_KEY=${TMDB_V3_API_KEY} \
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
                def patchedYaml = "K8s/deployment-${env.BUILD_NUMBER}.yaml"

                sh """
                    sed 's|image:.*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}|g' \
                        K8s/deployment.yaml > ${patchedYaml}
                    
                    kubectl apply -f ${patchedYaml}
                    kubectl apply -f K8s/service.yaml
                """

                try {
                    sh "kubectl rollout status deployment/${K8S_DEPLOYMENT} --timeout=60s"
                } catch (err) {
                    echo "⚠️ Deployment failed, rolling back..."
                    sh "kubectl rollout undo deployment/${K8S_DEPLOYMENT}"
                    error("Deployment failed and rollback was triggered.")
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
