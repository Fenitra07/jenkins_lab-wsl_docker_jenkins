cat > Jenkinsfile << 'EOF'
pipeline {
    agent none

    stages {
        stage('Checkout') {
            agent { label 'review' }
            steps {
                echo "✅ Code récupéré"
            }
        }

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

        stage('Deploy Staging') {
            agent { label 'stage' }
            when { branch 'develop' }
            steps {
                echo "📦 Deploy sur Staging..."
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
EOF
