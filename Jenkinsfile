pipeline {
    agent any

    environment {
        IMAGE_NAME = 'online-exam-system-backend'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Build Docker Image') {
            steps {
                sh 'docker build --pull -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }

        stage('Smoke Test') {
            steps {
                sh 'docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} node --check server.js'
            }
        }
    }

    post {
        always {
            sh 'docker image rm ${IMAGE_NAME}:${IMAGE_TAG} || true'
        }
    }
}
