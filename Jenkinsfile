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
        IMAGE_NAME = 'vishnumadhu13/online-exam-system-backend'
        IMAGE_TAG = "${BUILD_NUMBER}"

        // ==========================================
        // JENKINS CREDENTIALS
        // ==========================================
        DOCKER_CREDENTIALS = 'docker-cred'
        SSH_CREDENTIALS = 'deployment-server-ssh'
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

        // ==========================================
        // DOCKER SMOKE TEST
        // ==========================================

        stage('Docker Smoke Test') {
            steps {
                echo 'Starting Docker smoke test with MongoDB...'

                withCredentials([
                    string(
                        credentialsId: "${JWT_CREDENTIALS}",
                        variable: 'JWT_SECRET'
                    )
                ]) {
                    sh '''
                        set -e

                        NETWORK_NAME="online-exam-network"
                        MONGO_CONTAINER="online-exam-mongo-test"
                        BACKEND_CONTAINER="${APP_NAME}-test"

                        echo "=========================================="
                        echo "DOCKER SMOKE TEST"
                        echo "=========================================="

                        echo "Creating Docker network..."

                        docker network create ${NETWORK_NAME} 2>/dev/null || true

                        echo "Cleaning up old test containers..."

                        docker rm -f ${BACKEND_CONTAINER} 2>/dev/null || true
                        docker rm -f ${MONGO_CONTAINER} 2>/dev/null || true

                        echo ""
                        echo "Starting MongoDB container..."

                        docker run -d \
                            --name ${MONGO_CONTAINER} \
                            --network ${NETWORK_NAME} \
                            mongo:7

                        echo "MongoDB container started."

                        echo ""
                        echo "Waiting for MongoDB to become ready..."

                        for i in $(seq 1 30); do

                            if docker exec ${MONGO_CONTAINER} \
                                mongosh --quiet \
                                --eval "db.adminCommand('ping').ok" \
                                2>/dev/null | grep -q "1"; then

                                echo "MongoDB is ready!"
                                break
                            fi

                            echo "Waiting for MongoDB... attempt ${i}/30"

                            sleep 2

                            if [ "$i" -eq 30 ]; then
                                echo "MongoDB failed to become ready!"

                                docker logs ${MONGO_CONTAINER} || true

                                exit 1
                            fi

                        done

                        echo ""
                        echo "Starting backend container..."

                        docker run -d \
                            --name ${BACKEND_CONTAINER} \
                            --network ${NETWORK_NAME} \
                            -p 5001:${APP_PORT} \
                            -e NODE_ENV=production \
                            -e PORT=${APP_PORT} \
                            -e MONGO_URI="mongodb://${MONGO_CONTAINER}:27017/online-exam-system" \
                            -e JWT_SECRET="${JWT_SECRET}" \
                            ${IMAGE_NAME}:${IMAGE_TAG}

                        echo "Backend container started."

                        echo ""
                        echo "Waiting for backend to become healthy..."

                        for i in $(seq 1 30); do

                            if curl -fsS http://localhost:5001/health > /dev/null 2>&1; then

                                echo "Backend is healthy!"

                                break
                            fi

                            if ! docker ps --format '{{.Names}}' | grep -q "^${BACKEND_CONTAINER}$"; then

                                echo "Backend container stopped unexpectedly!"

                                docker logs ${BACKEND_CONTAINER} || true
                                docker logs ${MONGO_CONTAINER} || true

                                exit 1
                            fi

                            echo "Waiting for backend... attempt ${i}/30"

                            sleep 2

                            if [ "$i" -eq 30 ]; then

                                echo "Backend health check failed!"

                                docker logs ${BACKEND_CONTAINER} || true
                                docker logs ${MONGO_CONTAINER} || true

                                exit 1
                            fi

                        done

                        echo ""
                        echo "Testing root endpoint..."

                        curl -f http://localhost:5001/

                        echo ""
                        echo "Testing health endpoint..."

                        curl -f http://localhost:5001/health

                        echo ""
                        echo "=========================================="
                        echo "DOCKER SMOKE TEST PASSED"
                        echo "=========================================="
                    '''
                }
            }

            post {
                always {
                    sh '''
                        echo ""
                        echo "=========================================="
                        echo "BACKEND LOGS"
                        echo "=========================================="

                        docker logs ${APP_NAME}-test 2>/dev/null || true

                        echo ""
                        echo "=========================================="
                        echo "MONGODB LOGS"
                        echo "=========================================="

                        docker logs online-exam-mongo-test 2>/dev/null || true

                        echo ""
                        echo "Cleaning smoke-test containers..."

                        docker rm -f ${APP_NAME}-test 2>/dev/null || true
                        docker rm -f online-exam-mongo-test 2>/dev/null || true

                        echo "Removing smoke-test network..."

                        docker network rm online-exam-network 2>/dev/null || true
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

        // ==========================================
        // DEPLOY TO EC2
        // ==========================================

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

                                NETWORK_NAME="online-exam-network"
                                MONGO_CONTAINER="online-exam-mongo"
                                BACKEND_CONTAINER="${CONTAINER_NAME}"
                                MONGO_VOLUME="online-exam-mongo-data"

                                echo "=========================================="
                                echo "ONLINE EXAM SYSTEM DEPLOYMENT"
                                echo "=========================================="

                                echo ""
                                echo "Logging into Docker Hub..."

                                echo "${DOCKER_PASSWORD}" | docker login \
                                    -u "${DOCKER_USERNAME}" \
                                    --password-stdin

                                echo ""
                                echo "Pulling Docker image..."

                                docker pull ${IMAGE_NAME}:${IMAGE_TAG}

                                echo ""
                                echo "Creating Docker network..."

                                docker network create ${NETWORK_NAME} 2>/dev/null || true

                                echo ""
                                echo "Creating MongoDB volume..."

                                docker volume create ${MONGO_VOLUME} 2>/dev/null || true

                                echo ""
                                echo "Checking MongoDB container..."

                                if docker ps -a --format '{{.Names}}' | grep -q "^${MONGO_CONTAINER}$"; then

                                    echo "MongoDB container already exists."

                                    if ! docker ps --format '{{.Names}}' | grep -q "^${MONGO_CONTAINER}$"; then

                                        echo "Starting existing MongoDB container..."

                                        docker start ${MONGO_CONTAINER}

                                    else

                                        echo "MongoDB container is already running."

                                    fi

                                else

                                    echo "Creating MongoDB container..."

                                    docker run -d \
                                        --name ${MONGO_CONTAINER} \
                                        --restart unless-stopped \
                                        --network ${NETWORK_NAME} \
                                        -v ${MONGO_VOLUME}:/data/db \
                                        mongo:7

                                fi

                                echo ""
                                echo "Waiting for MongoDB to become ready..."

                                for i in $(seq 1 30); do

                                    if docker exec ${MONGO_CONTAINER} \
                                        mongosh --quiet \
                                        --eval "db.adminCommand('ping').ok" \
                                        2>/dev/null | grep -q "1"; then

                                        echo "MongoDB is ready!"

                                        break
                                    fi

                                    echo "Waiting for MongoDB... attempt ${i}/30"

                                    sleep 2

                                    if [ "$i" -eq 30 ]; then

                                        echo "MongoDB failed to become ready!"

                                        docker logs ${MONGO_CONTAINER} || true

                                        exit 1
                                    fi

                                done

                                echo ""
                                echo "Stopping existing backend container..."

                                docker stop ${BACKEND_CONTAINER} 2>/dev/null || true
                                docker rm ${BACKEND_CONTAINER} 2>/dev/null || true

                                echo ""
                                echo "Starting new backend container..."

                                docker run -d \
                                    --name ${BACKEND_CONTAINER} \
                                    --restart unless-stopped \
                                    --network ${NETWORK_NAME} \
                                    -p ${APP_PORT}:${APP_PORT} \
                                    -e NODE_ENV=production \
                                    -e PORT=${APP_PORT} \
                                    -e MONGO_URI="mongodb://${MONGO_CONTAINER}:27017/online-exam-system" \
                                    -e JWT_SECRET="${JWT_SECRET}" \
                                    ${IMAGE_NAME}:${IMAGE_TAG}

                                echo ""
                                echo "Backend container started."

                                echo ""
                                echo "Waiting for application to become healthy..."

                                for i in $(seq 1 30); do

                                    if curl -fsS \
                                        http://localhost:${APP_PORT}/health \
                                        > /dev/null 2>&1; then

                                        echo "Application is healthy!"

                                        break
                                    fi

                                    if ! docker ps --format '{{.Names}}' | grep -q "^${BACKEND_CONTAINER}$"; then

                                        echo "Deployment failed: backend container is not running."

                                        docker logs ${BACKEND_CONTAINER} || true
                                        docker logs ${MONGO_CONTAINER} || true

                                        exit 1
                                    fi

                                    echo "Waiting for application... attempt ${i}/30"

                                    sleep 2

                                    if [ "$i" -eq 30 ]; then

                                        echo "Application failed to become healthy."

                                        docker logs ${BACKEND_CONTAINER} || true
                                        docker logs ${MONGO_CONTAINER} || true

                                        exit 1
                                    fi

                                done

                                echo ""
                                echo "Backend container status:"

                                docker ps --filter "name=${BACKEND_CONTAINER}"

                                echo ""
                                echo "MongoDB container status:"

                                docker ps --filter "name=${MONGO_CONTAINER}"

                                echo ""
                                echo "Backend application logs:"

                                docker logs --tail 50 ${BACKEND_CONTAINER}

                                echo ""
                                echo "=========================================="
                                echo "DEPLOYMENT SUCCESSFUL"
                                echo "=========================================="

EOF
                        '''
                    }
                }
            }
        }

        // ==========================================
        // POST DEPLOYMENT HEALTH CHECK
        // ==========================================

        stage('Post Deployment Health Check') {
            steps {
                echo 'Checking deployed application...'

                sh '''
                    echo "Waiting for deployed application..."

                    for i in $(seq 1 30); do

                        if curl -fsS \
                            http://${DEPLOY_HOST}:${APP_PORT}/health \
                            > /dev/null 2>&1; then

                            echo "Application is responding!"

                            break
                        fi

                        echo "Waiting for application... attempt ${i}/30"

                        sleep 2

                        if [ "$i" -eq 30 ]; then

                            echo "Application health check failed!"

                            exit 1
                        fi

                    done

                    echo ""
                    echo "=========================================="
                    echo "APPLICATION HEALTH CHECK PASSED"
                    echo "=========================================="

                    curl -f http://${DEPLOY_HOST}:${APP_PORT}/health

                    echo ""
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