pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = "lydiahlaw"
        IMAGE_NAME = "tooling"
        IMAGE_TAG = "${env.BRANCH_NAME}-0.0.1"
    }

    stages {

        stage('Initial Cleanup') {
            steps {
                dir("${WORKSPACE}") {
                    deleteDir()
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ''' + "${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}" + '''
                    '''
                }
            }
        }

        stage('Cleanup Images') {
            steps {
                sh """
                    docker rmi ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} || true
                """
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
    }
}
