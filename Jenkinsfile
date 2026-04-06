cat > Jenkinsfile << 'EOF'
pipeline {
    agent none

    stages {
        stage('Checkout') {
            agent { label 'review' }
            steps {
                echo "✅ Code récupéré sur branche ${env.BRANCH_NAME}"
            }
        }

        stage('Install') {
            agent { label 'review' }
            steps {
                echo "📦 Installation des dépendances..."
                sh '''
                    apt-get update -q
                    apt-get install -y python3 python3-pip -q
                    pip3 install -r app/requirements.txt
                '''
            }
        }

        stage('Test') {
            agent { label 'stage' }
            steps {
                echo "🧪 Lancement des tests..."
                sh '''
                    apt-get update -q
                    apt-get install -y python3 python3-pip -q
                    pip3 install -r app/requirements.txt
                    cd app && python3 -m pytest tests/ -v
                '''
            }
        }

        stage('Deploy Staging') {
            agent { label 'stage' }
            when { branch 'develop' }
            steps {
                echo "📦 Deploy sur Staging..."
                sh '''
                    pkill -f "python3 app.py" || true
                    cd app && nohup python3 app.py &
                    echo "✅ App lancée sur staging"
                '''
            }
        }

        stage('Deploy Production') {
            agent { label 'prod' }
            when { branch 'main' }
            steps {
                echo "🚀 Deploy en Production !"
                sh '''
                    pkill -f "python3 app.py" || true
                    cd app && nohup python3 app.py &
                    echo "✅ App lancée en prod"
                '''
            }
        }
    }

    post {
        success { echo "✅ Pipeline réussi !" }
        failure { echo "❌ Pipeline échoué !" }
    }
}
EOF
