```groovy
pipeline {
    agent any

    environment {
        // Application
        APP_NAME = 'online-exam-system-backend'

        // Docker
        IMAGE_NAME = 'vishnumadhu/online-exam-system-backend'
        IMAGE_TAG = "${BUILD_NUMBER}"

        // Jenkins credentials IDs
        DOCKER_CREDENTIALS = 'dockerhub-credentials'
        SSH_CREDENTIALS = 'deployment-server-ssh'

        // Deployment
        DEPLOY_HOST = 'YOUR_SERVER_IP'
        DEPLOY_USER = 'ubuntu'
        CONTAINER_NAME = 'online-exam-backend'
        APP_PORT = '5000'
    }

    stages {

        /*
         * ==========================================
         * CI - CONTINUOUS INTEGRATION
         * ==========================================
         */

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                echo 'Installing Node.js dependencies...'

                sh '''
                    cd backend
                    npm ci
                '''
            }
        }

        stage('Lint') {
            steps {
                echo 'Running ESLint...'

                sh '''
                    cd backend
                    if [ -f package.json ] && npm run | grep -q "lint"; then
                        npm run lint
                    else
                        echo "Lint script not configured. Skipping..."
                    fi
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                echo 'Running unit tests...'

                sh '''
                    cd backend
                    if [ -f package.json ] && npm run | grep -q "test"; then
                        npm test
                    else
                        echo "Test script not configured. Skipping..."
                    fi
                '''
            }
        }

        stage('Node.js Syntax Check') {
            steps {
                echo 'Checking Node.js syntax...'

                sh '''
                    node --check backend/server.js
                '''
            }
        }

        stage('Security Scan - Dependencies') {
            steps {
                echo 'Scanning npm dependencies for vulnerabilities...'

                sh '''
                    cd backend
                    npm audit --audit-level=high || true
                '''
            }
        }

        /*
         * ==========================================
         * DOCKER
         * ==========================================
         */

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"

                sh '''
                    docker build \
                        --pull \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} \
                        -t ${IMAGE_NAME}:latest \
                        .
                '''
            }
        }

        stage('Docker Image Scan') {
            steps {
                echo 'Scanning Docker image for vulnerabilities...'

                sh '''
                    if command -v trivy >/dev/null 2>&1; then
                        trivy image \
                            --severity HIGH,CRITICAL \
                            --exit-code 0 \
                            ${IMAGE_NAME}:${IMAGE_TAG}
                    else
                        echo "Trivy is not installed. Skipping image scan."
                    fi
                '''
            }
        }

        stage('Docker Smoke Test') {
            steps {
                echo 'Starting temporary container for smoke testing...'

                sh '''
                    docker run -d \
                        --name ${APP_NAME}-test \
                        -p 3001:${APP_PORT} \
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    sleep 10

                    docker ps | grep ${APP_NAME}-test

                    docker exec ${APP_NAME}-test node --check server.js
                '''
            }

            post {
                always {
                    sh '''
                        docker stop ${APP_NAME}-test || true
                        docker rm ${APP_NAME}-test || true
                    '''
                }
            }
        }

        /*
         * ==========================================
         * CD - CONTINUOUS DELIVERY
         * ==========================================
         */

        stage('Login to Docker Hub') {
            steps {
                echo 'Logging in to Docker Hub...'

                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "${DOCKER_PASSWORD}" | docker login \
                            -u "${DOCKER_USERNAME}" \
                            --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                echo "Pushing Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"

                sh '''
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Deploy to Server') {
            steps {
                echo "Deploying ${IMAGE_NAME}:${IMAGE_TAG} to ${DEPLOY_HOST}..."

                sshagent(credentials: ["${SSH_CREDENTIALS}"]) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                            ${DEPLOY_USER}@${DEPLOY_HOST} << EOF

                        echo "Logging into Docker Hub..."

                        docker login \
                            -u ${DOCKER_USERNAME} \
                            -p ${DOCKER_PASSWORD}

                        echo "Pulling latest Docker image..."

                        docker pull ${IMAGE_NAME}:${IMAGE_TAG}

                        echo "Stopping existing container..."

                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true

                        echo "Starting new container..."

                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            --restart unless-stopped \
                            -p ${APP_PORT}:${APP_PORT} \
                            ${IMAGE_NAME}:${IMAGE_TAG}

                        echo "Deployment completed."

                        docker ps

                        EOF
                    '''
                }
            }
        }

        stage('Post Deployment Smoke Test') {
            steps {
                echo 'Testing deployed application...'

                sh '''
                    sleep 10

                    curl -f http://${DEPLOY_HOST}:${APP_PORT}/
                '''
            }
        }
    }

    /*
     * ==========================================
     * POST ACTIONS
     * ==========================================
     */

    post {

        success {
            echo '''
            ==========================================
            PIPELINE SUCCESSFUL
            ==========================================
            '''
            echo "Application: ${APP_NAME}"
            echo "Docker Image: ${IMAGE_NAME}:${IMAGE_TAG}"
            echo "Build Number: ${BUILD_NUMBER}"
            echo "Deployment: SUCCESS"
        }

        failure {
            echo '''
            ==========================================
            PIPELINE FAILED
            ==========================================
            '''
            echo "Build Number: ${BUILD_NUMBER}"
            echo "Check Jenkins console output for details."
        }

        always {
            echo 'Cleaning Jenkins workspace and Docker resources...'

            sh '''
                docker logout || true

                docker image rm ${IMAGE_NAME}:${IMAGE_TAG} || true
                docker image rm ${IMAGE_NAME}:latest || true

                docker system prune -f || true
            '''

            cleanWs()
        }
    }
}
```
