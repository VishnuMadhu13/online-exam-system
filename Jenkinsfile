pipeline {
    agent any

    environment {
        // Application
        APP_NAME = 'online-exam-system-backend'
        CONTAINER_NAME = 'online-exam-backend'
        APP_PORT = '5000'

        // Docker Hub
        IMAGE_NAME = 'vishnumadhu/online-exam-system-backend'
        IMAGE_TAG = "${BUILD_NUMBER}"

        // Jenkins Credentials IDs
        DOCKER_CREDENTIALS = 'dockerhub-credentials'
        SSH_CREDENTIALS = 'deployment-server-ssh'
        MONGO_CREDENTIALS = 'mongodb-uri'
        JWT_CREDENTIALS = 'jwt-secret'

        // Deployment Server
        DEPLOY_HOST = 'YOUR_EC2_PUBLIC_IP'
        DEPLOY_USER = 'ubuntu'
    }

    stages {

        // ==========================================
        // CI - CONTINUOUS INTEGRATION
        // ==========================================

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                echo 'Installing backend dependencies...'

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

                    if npm run | grep -q "lint"; then
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

                    if npm run | grep -q "test"; then
                        npm test
                    else
                        echo "Test script not configured. Skipping..."
                    fi
                '''
            }
        }

        stage('Syntax Check') {
            steps {
                echo 'Checking Node.js syntax...'

                sh '''
                    node --check backend/server.js
                '''
            }
        }

        stage('Dependency Security Scan') {
            steps {
                echo 'Scanning npm dependencies...'

                sh '''
                    cd backend
                    npm audit --audit-level=high || true
                '''
            }
        }

        // ==========================================
        // DOCKER
        // ==========================================

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image ${IMAGE_NAME}:${IMAGE_TAG}"

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
                echo 'Starting temporary container...'

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
                        docker run -d \
                            --name ${APP_NAME}-test \
                            -p 5001:${APP_PORT} \
                            -e MONGO_URI="${MONGO_URI}" \
                            -e JWT_SECRET="${JWT_SECRET}" \
                            -e PORT=${APP_PORT} \
                            ${IMAGE_NAME}:${IMAGE_TAG}

                        echo "Waiting for application..."
                        sleep 10

                        echo "Container status:"
                        docker ps -a | grep ${APP_NAME}-test

                        echo "Container logs:"
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

                        echo "Pushing versioned image..."

                        docker push ${IMAGE_NAME}:${IMAGE_TAG}

                        echo "Pushing latest image..."

                        docker push ${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {

                echo "Deploying ${IMAGE_NAME}:${IMAGE_TAG} to ${DEPLOY_HOST}"

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
                            ssh -o StrictHostKeyChecking=no \
                                ${DEPLOY_USER}@${DEPLOY_HOST} << EOF

                                set -e

                                echo "=========================================="
                                echo "Deploying Online Exam System Backend"
                                echo "=========================================="

                                echo "Logging into Docker Hub..."

                                echo "${DOCKER_PASSWORD}" | docker login \
                                    -u "${DOCKER_USERNAME}" \
                                    --password-stdin

                                echo "Pulling image..."

                                docker pull ${IMAGE_NAME}:${IMAGE_TAG}

                                echo "Stopping existing container..."

                                docker stop ${CONTAINER_NAME} || true
                                docker rm ${CONTAINER_NAME} || true

                                echo "Starting new container..."

                                docker run -d \
                                    --name ${CONTAINER_NAME} \
                                    --restart unless-stopped \
                                    -p ${APP_PORT}:${APP_PORT} \
                                    -e NODE_ENV=production \
                                    -e PORT=${APP_PORT} \
                                    -e MONGO_URI="${MONGO_URI}" \
                                    -e JWT_SECRET="${JWT_SECRET}" \
                                    ${IMAGE_NAME}:${IMAGE_TAG}

                                echo "Waiting for application..."

                                sleep 10

                                echo "Container status:"

                                docker ps | grep ${CONTAINER_NAME}

                                echo "Application logs:"

                                docker logs --tail 50 ${CONTAINER_NAME}

                                echo "Deployment completed successfully."

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
            echo "Image       : ${IMAGE_NAME}:${IMAGE_TAG}"
            echo "Build       : ${BUILD_NUMBER}"
            echo "Deployment  : SUCCESS"
        }

        failure {
            echo '''
            ==========================================
                 PIPELINE FAILED
            ==========================================
            '''

            echo "Build ${BUILD_NUMBER} failed."
            echo "Check Jenkins console output."
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