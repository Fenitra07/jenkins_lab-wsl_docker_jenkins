pipeline {
    agent none

    environment {
        DOCKER_HUB_USER = 'fenitrar07'
        IMAGE_NAME      = 'jenkins-python-app'
        IMAGE_TAG       = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            agent { label 'review' }
            steps {
                echo "Code recupere depuis GitHub"
                checkout scm
            }
        }

        stage('Build') {
            agent { label 'review' }
            steps {
                echo "Build de l image Docker..."
                sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "docker tag ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
            }
        }

        stage('Test') {
            agent { label 'review' }
            steps {
                echo "Tests en cours..."
                sh """
                    docker run --rm \
                        ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} \
                        python3 -m pytest tests/ -v
                """
            }
        }

        stage('Push Docker Hub') {
            agent { label 'review' }
            steps {
                echo "Push sur Docker Hub..."
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Deploy Review') {
            agent { label 'review' }
            steps {
                echo "Deploy sur Review..."
                sh """
                    docker stop app-review || true
                    docker rm app-review || true
                    docker run -d --name app-review -p 5001:5000 \
                        ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy Staging') {
            agent { label 'stage' }
            steps {
                echo "Deploy sur Staging..."
                sh """
                    docker stop app-stage || true
                    docker rm app-stage || true
                    docker run -d --name app-stage -p 5002:5000 \
                        ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy Production') {
            agent { label 'prod' }
            when { branch 'main' }
            steps {
                echo "Deploy sur Production..."
                sh """
                    docker stop app-prod || true
                    docker rm app-prod || true
                    docker run -d --name app-prod -p 5003:5000 \
                        ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
    }

    post {
        success { echo "Pipeline reussi ! Image: ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}" }
        failure { echo "Pipeline echoue !" }
    }
}
