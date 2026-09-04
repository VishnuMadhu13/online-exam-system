pipeline {
    agent any

    environment {
        // ==========================================
        // APPLICATION
        // ==========================================
        APP_NAME = 'online-exam-system-backend'
        CONTAINER_NAME = 'online-exam-backend'
        APP_PORT = '5000'

        // ==========================================
        // DOCKER HUB
        // ==========================================
        IMAGE_NAME = 'vishnumadhu/online-exam-system-backend'
        IMAGE_TAG = "${BUILD_NUMBER}"

        // ==========================================
        // JENKINS CREDENTIALS
        // ==========================================
        DOCKER_CREDENTIALS = 'dockerhub-credentials'
        SSH_CREDENTIALS = 'deployment-server-ssh'
        MONGO_CREDENTIALS = 'mongodb-uri'
        JWT_CREDENTIALS = 'jwt-secret'

        // ==========================================
        // DEPLOYMENT SERVER
        // ==========================================
        DEPLOY_HOST = '13.206.69.212'
        DEPLOY_USER = 'ubuntu'
    }

    stages {

        // ==========================================
        // CI - CONTINUOUS INTEGRATION
        // ==========================================

        stage('Install Dependencies') {
            steps {
                echo 'Installing backend dependencies...'

                sh '''
                    docker run --rm \
                        -v "$WORKSPACE/backend:/app" \
                        -w /app \
                        node:20-alpine \
                        npm ci
                '''
            }
        }

        stage('Lint') {
            steps {
                echo 'Running ESLint...'

                sh '''
                    docker run --rm \
                        -v "$WORKSPACE/backend:/app" \
                        -w /app \
                        node:20-alpine \
                        sh -c '
                            if npm run | grep -q "lint"; then
                                npm run lint
                            else
                                echo "Lint script not configured. Skipping..."
                            fi
                        '
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                echo 'Running unit tests...'

                sh '''
                    docker run --rm \
                        -v "$WORKSPACE/backend:/app" \
                        -w /app \
                        node:20-alpine \
                        sh -c '
                            if npm run | grep -q "test"; then
                                npm test
                            else
                                echo "Test script not configured. Skipping..."
                            fi
                        '
                '''
            }
        }

        stage('Syntax Check') {
            steps {
                echo 'Checking Node.js syntax...'

                sh '''
                    docker run --rm \
                        -v "$WORKSPACE/backend:/app" \
                        -w /app \
                        node:20-alpine \
                        node --check server.js
                '''
            }
        }

        stage('Dependency Security Scan') {
            steps {
                echo 'Scanning npm dependencies for vulnerabilities...'

                sh '''
                    docker run --rm \
                        -v "$WORKSPACE/backend:/app" \
                        -w /app \
                        node:20-alpine \
                        sh -c 'npm audit --audit-level=high || true'
                '''
            }
        }

        // ==========================================
        // DOCKER
        // ==========================================

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
                echo 'Starting temporary Docker container...'

                withCredentials([
                    string(
                        credentialsId: "${MONGO_CREDENTIALS}",
                        variable: 'MONGO_URI'
                    ),
                    string(
                        credentialsId: "${JWT_CREDENTIALS}",
                        variable: 'JWT_SECRET'
                    )
                ]) {

                    sh '''
                        set -e

                        docker run -d \
                            --name ${APP_NAME}-test \
                            -p 5001:${APP_PORT} \
                            -e NODE_ENV=production \
                            -e PORT=${APP_PORT} \
                            -e MONGO_URI="${MONGO_URI}" \
                            -e JWT_SECRET="${JWT_SECRET}" \
                            ${IMAGE_NAME}:${IMAGE_TAG}

                        echo "Waiting for application to start..."
                        sleep 10

                        echo "Checking container status..."

                        if ! docker ps | grep -q ${APP_NAME}-test; then
                            echo "Container failed to start!"
                            docker logs ${APP_NAME}-test || true
                            exit 1
                        fi

                        echo "Container is running."

                        echo "Application logs:"
                        docker logs ${APP_NAME}-test

                        echo "Checking application endpoint..."

                        curl -f http://localhost:5001/ || {
                            echo "Application smoke test failed!"
                            exit 1
                        }

                        echo "Docker smoke test passed!"
                    '''
                }
            }

            post {
                always {
                    sh '''
                        docker logs ${APP_NAME}-test || true
                        docker stop ${APP_NAME}-test || true
                        docker rm ${APP_NAME}-test || true
                    '''
                }
            }
        }

        // ==========================================
        // CD - CONTINUOUS DELIVERY
        // ==========================================

        stage('Docker Hub Login & Push') {
            steps {
                echo 'Logging into Docker Hub and pushing image...'

                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "${DOCKER_PASSWORD}" | docker login \
                            -u "${DOCKER_USERNAME}" \
                            --password-stdin

                        echo "Pushing versioned image..."

                        docker push ${IMAGE_NAME}:${IMAGE_TAG}

                        echo "Pushing latest image..."

                        docker push ${IMAGE_NAME}:latest

                        echo "Docker images pushed successfully."
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo "Deploying ${IMAGE_NAME}:${IMAGE_TAG} to ${DEPLOY_HOST}..."

                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    ),
                    string(
                        credentialsId: "${MONGO_CREDENTIALS}",
                        variable: 'MONGO_URI'
                    ),
                    string(
                        credentialsId: "${JWT_CREDENTIALS}",
                        variable: 'JWT_SECRET'
                    )
                ]) {

                    sshagent(credentials: ["${SSH_CREDENTIALS}"]) {

                        sh '''
                            set -e

                            ssh -o StrictHostKeyChecking=no \
                                ${DEPLOY_USER}@${DEPLOY_HOST} << EOF

                                set -e

                                echo "=========================================="
                                echo "ONLINE EXAM SYSTEM DEPLOYMENT"
                                echo "=========================================="

                                echo "Logging into Docker Hub..."

                                echo "${DOCKER_PASSWORD}" | docker login \
                                    -u "${DOCKER_USERNAME}" \
                                    --password-stdin

                                echo "Pulling Docker image..."

                                docker pull ${IMAGE_NAME}:${IMAGE_TAG}

                                echo "Stopping existing container..."

                                docker stop ${CONTAINER_NAME} || true
                                docker rm ${CONTAINER_NAME} || true

                                echo "Starting new application container..."

                                docker run -d \
                                    --name ${CONTAINER_NAME} \
                                    --restart unless-stopped \
                                    -p ${APP_PORT}:${APP_PORT} \
                                    -e NODE_ENV=production \
                                    -e PORT=${APP_PORT} \
                                    -e MONGO_URI="${MONGO_URI}" \
                                    -e JWT_SECRET="${JWT_SECRET}" \
                                    ${IMAGE_NAME}:${IMAGE_TAG}

                                echo "Waiting for application to start..."

                                sleep 10

                                echo "Checking container..."

                                if ! docker ps | grep -q ${CONTAINER_NAME}; then
                                    echo "Deployment failed: container is not running."
                                    docker logs ${CONTAINER_NAME} || true
                                    exit 1
                                fi

                                echo "Container is running."

                                echo "Application logs:"
                                docker logs --tail 50 ${CONTAINER_NAME}

                                echo "=========================================="
                                echo "DEPLOYMENT SUCCESSFUL"
                                echo "=========================================="

                            EOF
                        '''
                    }
                }
            }
        }

        stage('Post Deployment Health Check') {
            steps {
                echo 'Checking deployed application...'

                sh '''
                    sleep 5

                    curl -f http://${DEPLOY_HOST}:${APP_PORT}/

                    echo "=========================================="
                    echo "APPLICATION HEALTH CHECK PASSED"
                    echo "=========================================="
                '''
            }
        }
    }

    // ==========================================
    // POST ACTIONS
    // ==========================================

    post {

        success {
            echo '''
            ==========================================
                 PIPELINE SUCCESSFUL
            ==========================================
            '''

            echo "Application : ${APP_NAME}"
            echo "Docker Image: ${IMAGE_NAME}:${IMAGE_TAG}"
            echo "Build Number: ${BUILD_NUMBER}"
            echo "Deployment  : SUCCESS"
        }

        failure {
            echo '''
            ==========================================
                 PIPELINE FAILED
            ==========================================
            '''

            echo "Build Number: ${BUILD_NUMBER}"
            echo "Deployment failed."
            echo "Check Jenkins console output for details."
        }

        always {
            echo 'Cleaning Jenkins Docker resources...'

            sh '''
                docker logout || true

                docker image rm ${IMAGE_NAME}:${IMAGE_TAG} || true
                docker image rm ${IMAGE_NAME}:latest || true
            '''

            cleanWs()
        }
    }
}