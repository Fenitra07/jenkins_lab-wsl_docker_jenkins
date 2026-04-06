pipeline {
    agent none

    stages {
        stage('Checkout') {
            agent { label 'review' }
            steps {
                echo "Code recupere"
            }
        }

        stage('Build') {
            agent { label 'review' }
            steps {
                echo "Build en cours..."
                sh 'echo "Build OK" > build.txt'
            }
        }

        stage('Test') {
            agent { label 'stage' }
            steps {
                echo "Tests en cours..."
                sh 'apt-get install -y python3 python3-pip -q && cd app && python3 -m pytest tests/ -v'
            }
        }

        stage('Deploy Production') {
            agent { label 'prod' }
            when { branch 'main' }
            steps {
                echo "Deploy en Production !"
            }
        }
    }

    post {
        success { echo "Pipeline reussi !" }
        failure { echo "Pipeline echoue !" }
    }
}
