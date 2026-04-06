pipeline {
    agent none

    stages {
        stage('Build') {
            agent { label 'review' }
            steps {
                echo "🔨 Build en cours..."
                sh 'echo "Build OK" > build.txt'
            }
        }

        stage('Test') {
            agent { label 'stage' }
            steps {
                echo "🧪 Tests en cours..."
            }
        }

        stage('Deploy Production') {
            agent { label 'prod' }
            when { branch 'main' }
            steps {
                echo "🚀 Deploy en Production !"
            }
        }
    }

    post {
        success { echo "✅ Pipeline réussi !" }
        failure { echo "❌ Pipeline échoué !" }
    }
}
