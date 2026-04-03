pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = "lydiahlaw"
        IMAGE_NAME = "tooling"
        IMAGE_TAG = "${env.BRANCH_NAME.replace('/', '-')}-0.0.1"
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

        stage('Test') {
            steps {
                sh """
                    docker run --name test-container \
                        --network tooling_app_network \
                        -e MYSQL_IP=mysqlserverhost \
                        -e MYSQL_USER=webaccess \
                        -e MYSQL_PASS=Devopslearn# \
                        -e MYSQL_DBNAME=toolingdb \
                        -d -p 8086:80 \
                        ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                    sleep 10
                    curl -s -o /dev/null -w "%{http_code}" http://localhost:8086 | grep 200
                """
            }
            post {
                always {
                    sh 'docker stop test-container && docker rm test-container || true'
                }
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
