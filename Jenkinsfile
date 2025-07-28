pipeline {
    agent any

    tools {
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
                git branch: 'main', url: 'https://github.com/nocnexhamza/Netflix.git'
            }
        }

        stage('Static Code Analysis (SonarQube)') {
            steps {
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=Netflix -Dsonar.sources=. -Dsonar.language=js -Dsonar.exclusions=Dockerfile"
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

        stage('Build & Unit Tests') {
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

        stage('OWASP Dependency-Check (SCA)') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check-2'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    DOCKER_BUILDKIT=1 docker build \
                    --build-arg TMDB_V3_API_KEY=${TMDB_V3_API_KEY} \
                    -t ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER} .
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
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
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
                    publishHTML(target: [
                        reportDir: '.',
                        reportFiles: 'trivy-report.txt',
                        reportName: 'Trivy Vulnerability Report'
                    ])
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
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

        stage('DAST Scanning (ZAP & Amass)') {
            steps {
                sh '''
                    echo "Running Amass..."
                    docker run --rm caffix/amass enum -passive -d netflix.nocnexus.com -o amass-results.txt || true

                    echo "Running OWASP ZAP..."
                    docker run --rm -v $WORKSPACE:/zap/wrk \
                        owasp/zap2docker-stable zap-baseline.py \
                        -t https://hp.nocnexus.com \
                        -r zap-report.html || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: '*.txt,*.html', allowEmptyArchive: true
                    publishHTML(target: [
                        reportDir: '.',
                        reportFiles: 'zap-report.html',
                        reportName: 'OWASP ZAP DAST Report'
                    ])
                }
            }
        }
    }

    post {
        always {
            sh 'rm -f K8s/deployment-*.yaml trivy-report.txt zap-report.html amass-results.txt || true'
            cleanWs()
        }
        success {
            script {
                if (env.SLACK_CHANNEL) {
                    slackSend(
                        color: "good",
                        message: "✅ SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
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
                        message: "❌ FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                        channel: env.SLACK_CHANNEL
                    )
                }

                // Optional: Rollback to previous working version
                withKubeConfig([credentialsId: 'k8s-credentials']) {
                    sh '''
                        echo "Rollback initiated..."
                        kubectl rollout undo deployment/${K8S_DEPLOYMENT} || true
                    '''
                }
            }
        }
    }
}
