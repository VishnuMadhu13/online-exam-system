pipeline {

    agent any

    environment {

        // ==========================================
        // APPLICATION
        // ==========================================

        APP_NAME = 'online-exam-system'


        // ==========================================
        // DOCKER HUB
        // ==========================================

        BACKEND_IMAGE = 'vishnumadhu13/online-exam-system-backend'
        FRONTEND_IMAGE = 'vishnumadhu13/online-exam-system-frontend'

        IMAGE_TAG = "${BUILD_NUMBER}"


        // ==========================================
        // JENKINS CREDENTIALS
        // ==========================================

        DOCKER_CREDENTIALS = 'docker-cred'
        SSH_CREDENTIALS = 'deployment-server-ssh'
        JWT_CREDENTIALS = 'jwt-secret'


        // ==========================================
        // EC2
        // ==========================================

        DEPLOY_HOST = '3.111.196.132'
        DEPLOY_USER = 'ubuntu'
    }


    stages {


        // ==========================================
        // BACKEND DEPENDENCIES
        // ==========================================

        stage('Backend Dependencies') {

            steps {

                dir('backend') {

                    sh '''
                        set -e

                        echo "Installing backend dependencies..."

                        docker run --rm \
                            -v "$PWD:/app" \
                            -w /app \
                            node:20-alpine \
                            npm ci
                    '''
                }
            }
        }


        // ==========================================
        // FRONTEND DEPENDENCIES
        // ==========================================

        stage('Frontend Dependencies') {

            steps {

                dir('frontend') {

                    sh '''
                        set -e

                        echo "Installing frontend dependencies..."

                        docker run --rm \
                            -v "$PWD:/app" \
                            -w /app \
                            node:20-alpine \
                            npm ci
                    '''
                }
            }
        }


        // ==========================================
        // BACKEND SYNTAX CHECK
        // ==========================================

        stage('Backend Syntax Check') {

            steps {

                dir('backend') {

                    sh '''
                        set -e

                        echo "Checking backend syntax..."

                        docker run --rm \
                            -v "$PWD:/app" \
                            -w /app \
                            node:20-alpine \
                            node --check server.js
                    '''
                }
            }
        }


        // ==========================================
        // BACKEND LINT
        // ==========================================

        stage('Backend Lint') {

            steps {

                dir('backend') {

                    script {

                        def hasLint = sh(
                            script: '''
                                docker run --rm \
                                    -v "$PWD:/app" \
                                    -w /app \
                                    node:20-alpine \
                                    node -e "
                                        const p=require('./package.json');
                                        process.exit(
                                            p.scripts && p.scripts.lint ? 0 : 1
                                        )
                                    "
                            ''',
                            returnStatus: true
                        )

                        if (hasLint == 0) {

                            sh '''
                                docker run --rm \
                                    -v "$PWD:/app" \
                                    -w /app \
                                    node:20-alpine \
                                    npm run lint
                            '''

                        } else {

                            echo 'No backend lint script found. Skipping.'
                        }
                    }
                }
            }
        }


        // ==========================================
        // FRONTEND LINT
        // ==========================================

        stage('Frontend Lint') {

            steps {

                dir('frontend') {

                    script {

                        def hasLint = sh(
                            script: '''
                                docker run --rm \
                                    -v "$PWD:/app" \
                                    -w /app \
                                    node:20-alpine \
                                    node -e "
                                        const p=require('./package.json');
                                        process.exit(
                                            p.scripts && p.scripts.lint ? 0 : 1
                                        )
                                    "
                            ''',
                            returnStatus: true
                        )

                        if (hasLint == 0) {

                            sh '''
                                docker run --rm \
                                    -v "$PWD:/app" \
                                    -w /app \
                                    node:20-alpine \
                                    npm run lint
                            '''

                        } else {

                            echo 'No frontend lint script found. Skipping.'
                        }
                    }
                }
            }
        }


        // ==========================================
        // FRONTEND BUILD
        // ==========================================

        stage('Frontend Build') {

            steps {

                dir('frontend') {

                    sh '''
                        set -e

                        echo "Building React frontend..."

                        docker run --rm \
                            -v "$PWD:/app" \
                            -w /app \
                            node:20-alpine \
                            npm run build
                    '''
                }
            }
        }


        // ==========================================
        // BACKEND DEPENDENCY SECURITY
        // ==========================================

        stage('Backend Dependency Security Scan') {

            steps {

                dir('backend') {

                    sh '''
                        echo "Running backend npm audit..."

                        docker run --rm \
                            -v "$PWD:/app" \
                            -w /app \
                            node:20-alpine \
                            npm audit --audit-level=high || true
                    '''
                }
            }
        }


        // ==========================================
        // FRONTEND DEPENDENCY SECURITY
        // ==========================================

        stage('Frontend Dependency Security Scan') {

            steps {

                dir('frontend') {

                    sh '''
                        echo "Running frontend npm audit..."

                        docker run --rm \
                            -v "$PWD:/app" \
                            -w /app \
                            node:20-alpine \
                            npm audit --audit-level=high || true
                    '''
                }
            }
        }


        // ==========================================
        // BUILD BACKEND IMAGE
        // ==========================================

        stage('Build Backend Docker Image') {

            steps {

                sh '''
                    set -e

                    echo "Building backend Docker image..."

                    docker build \
                        -f Dockerfile.backend \
                        -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                        -t ${BACKEND_IMAGE}:latest \
                        .
                '''
            }
        }


        // ==========================================
        // BUILD FRONTEND IMAGE
        // ==========================================

        stage('Build Frontend Docker Image') {

            steps {

                sh '''
                    set -e

                    echo "Building frontend Docker image..."

                    docker build \
                        -f Dockerfile.frontend \
                        -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -t ${FRONTEND_IMAGE}:latest \
                        .
                '''
            }
        }


        // ==========================================
        // TRIVY SCAN
        // ==========================================

        stage('Trivy Security Scan') {

            steps {

                script {

                    def trivyInstalled = sh(
                        script: 'command -v trivy',
                        returnStatus: true
                    )

                    if (trivyInstalled == 0) {

                        sh '''
                            set -e

                            echo "Scanning backend image..."

                            trivy image \
                                --severity HIGH,CRITICAL \
                                --exit-code 0 \
                                ${BACKEND_IMAGE}:${IMAGE_TAG}

                            echo "Scanning frontend image..."

                            trivy image \
                                --severity HIGH,CRITICAL \
                                --exit-code 0 \
                                ${FRONTEND_IMAGE}:${IMAGE_TAG}
                        '''

                    } else {

                        echo 'Trivy is not installed on Jenkins. Skipping image scan.'
                    }
                }
            }
        }


        // ==========================================
        // COMPOSE SMOKE TEST
        // ==========================================

        stage('Docker Compose Smoke Test') {

            steps {

                sh '''
                    set -e

                    echo "Starting temporary Compose environment..."

                    export BACKEND_IMAGE="${BACKEND_IMAGE}:${IMAGE_TAG}"
                    export FRONTEND_IMAGE="${FRONTEND_IMAGE}:${IMAGE_TAG}"

                    docker compose \
                        -f docker-compose.test.yml \
                        -p online-exam-test \
                        up -d

                    echo "Waiting for containers..."
                    sleep 15

                    echo "=========================================="
                    echo "CONTAINER STATUS"
                    echo "=========================================="

                    docker compose \
                        -f docker-compose.test.yml \
                        -p online-exam-test \
                        ps

                    echo "=========================================="
                    echo "FRONTEND PORT CHECK"
                    echo "=========================================="

                    docker port online-exam-test-frontend-1 || true

                    echo "=========================================="
                    echo "FRONTEND LOGS"
                    echo "=========================================="

                    docker logs online-exam-test-frontend-1 || true

                    echo "=========================================="
                    echo "BACKEND TEST"
                    echo "=========================================="

                    curl --fail http://localhost:5001/health
                    echo

                    echo "=========================================="
                    echo "FRONTEND TEST"
                    echo "=========================================="

                    curl --fail http://localhost:8081/
                    echo

                    echo "Compose smoke test passed."
                '''
            }

            post {

                always {

                    sh '''
                        docker compose \
                            -f docker-compose.test.yml \
                            -p online-exam-test \
                            down -v || true
                    '''
                }
            }
        }


        // ==========================================
        // DOCKER HUB LOGIN
        // ==========================================

        stage('Docker Hub Login') {

            steps {

                withCredentials([

                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )

                ]) {

                    sh '''
                        set -e

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USER" \
                            --password-stdin
                    '''
                }
            }
        }


        // ==========================================
        // PUSH DOCKER IMAGES
        // ==========================================

        stage('Push Docker Images') {

            steps {

                sh '''
                    set -e

                    echo "Pushing backend image..."

                    docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                    docker push ${BACKEND_IMAGE}:latest

                    echo "Pushing frontend image..."

                    docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                    docker push ${FRONTEND_IMAGE}:latest
                '''
            }
        }


        // ==========================================
        // DEPLOY TO EC2
        // ==========================================

        stage('Deploy to EC2') {

            steps {

                withCredentials([

                    sshUserPrivateKey(
                        credentialsId: "${SSH_CREDENTIALS}",
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    ),

                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    ),

                    string(
                        credentialsId: "${JWT_CREDENTIALS}",
                        variable: 'JWT_SECRET'
                    )

                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "Deploying application to EC2"
                        echo "Host: ${DEPLOY_HOST}"
                        echo "Image Tag: ${IMAGE_TAG}"
                        echo "=========================================="

                        ssh \
                            -o StrictHostKeyChecking=no \
                            -o UserKnownHostsFile=/dev/null \
                            -i "$SSH_KEY" \
                            "$SSH_USER@$DEPLOY_HOST" \
                            "DOCKER_USER='$DOCKER_USER' \
                             DOCKER_PASSWORD='$DOCKER_PASSWORD' \
                             BACKEND_IMAGE='$BACKEND_IMAGE' \
                             FRONTEND_IMAGE='$FRONTEND_IMAGE' \
                             IMAGE_TAG='$IMAGE_TAG' \
                             JWT_SECRET='$JWT_SECRET' \
                             bash -s" <<'REMOTE_SCRIPT'

set -e

echo "=========================================="
echo "Connected to EC2"
echo "=========================================="

echo "Creating application directory..."

mkdir -p ~/online-exam-system
cd ~/online-exam-system

echo "Logging into Docker Hub..."

echo "$DOCKER_PASSWORD" | docker login \
    -u "$DOCKER_USER" \
    --password-stdin

echo "Creating production Compose file..."

cat > docker-compose.yml <<'EOF'
services:

  mongo:
    image: mongo:7
    container_name: online-exam-mongo
    restart: unless-stopped
    volumes:
      - online-exam-mongo-data:/data/db
    networks:
      - online-exam-network

  backend:
    image: ${BACKEND_IMAGE}:${IMAGE_TAG}
    container_name: online-exam-backend
    restart: unless-stopped
    environment:
      NODE_ENV: production
      PORT: 5000
      MONGO_URI: mongodb://mongo:27017/online-exam-system
      JWT_SECRET: ${JWT_SECRET}
    ports:
      - "5000:5000"
    depends_on:
      - mongo
    networks:
      - online-exam-network

  frontend:
    image: ${FRONTEND_IMAGE}:${IMAGE_TAG}
    container_name: online-exam-frontend
    restart: unless-stopped
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - online-exam-network

networks:
  online-exam-network:
    driver: bridge

volumes:
  online-exam-mongo-data:
EOF

echo "=========================================="
echo "Validating Docker Compose configuration..."
echo "=========================================="

docker compose config

echo "=========================================="
echo "Pulling Docker images..."
echo "=========================================="

docker compose pull

echo "=========================================="
echo "Stopping old application..."
echo "=========================================="

docker compose down || true

echo "=========================================="
echo "Starting application..."
echo "=========================================="

docker compose up -d

echo "=========================================="
echo "Waiting for services..."
echo "=========================================="

sleep 15

echo "=========================================="
echo "Compose status"
echo "=========================================="

docker compose ps

echo "=========================================="
echo "Backend health check"
echo "=========================================="

curl --fail http://localhost:5000/health

echo

echo "=========================================="
echo "Frontend health check"
echo "=========================================="

curl --fail http://localhost/

echo

echo "=========================================="
echo "Deployment successful"
echo "=========================================="

REMOTE_SCRIPT
                    '''
                }
            }
        }


        // ==========================================
        // POST DEPLOYMENT HEALTH CHECK
        // ==========================================

        stage('Post Deployment Health Check') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "Checking frontend..."
                    echo "=========================================="

                    curl --fail \
                        http://${DEPLOY_HOST}/

                    echo

                    echo "=========================================="
                    echo "Checking backend..."
                    echo "=========================================="

                    curl --fail \
                        http://${DEPLOY_HOST}:5000/health

                    echo

                    echo "Application is healthy."
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

            Frontend:
            http://13.207.150.113

            Backend:
            http://13.207.150.113:5000

            Backend Health:
            http://13.207.150.113:5000/health

            MongoDB:
            Internal Docker network only

            ==========================================
            '''
        }


        failure {

            echo '''
            ==========================================
            PIPELINE FAILED
            ==========================================
            '''
        }


        always {

            sh '''
                docker logout || true
            '''
        }
    }
}