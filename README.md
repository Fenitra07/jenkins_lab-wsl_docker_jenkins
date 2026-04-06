# Jenkins CI/CD Pipeline — Flask Python App

## Environnement utilisé

- Windows 11 + WSL2 (Ubuntu 24.04)
- Docker Desktop (driver pour les conteneurs)
- Jenkins LTS (conteneur Docker, port 9090)
- 3 agents Jenkins (conteneurs Docker : review, stage, prod)
- Docker Hub (registry d'images)
- GitHub (source code)

---

## Architecture globale

```
Développeur
     ↓ git push
GitHub (source code)
     ↓ webhook / manuel
Jenkins (port 9090)
     ↓
agent-review
  ├── Checkout (clone GitHub)
  ├── Build (docker build)
  ├── Test (pytest)
  └── Push Docker Hub
     ↓
agent-review  → app-review  (port 5001)
agent-stage   → app-stage   (port 5002)
agent-prod    → app-prod    (port 5003, branch main uniquement)
```

---

## Structure du projet

```
jenkins_lab/
├── Dockerfile            ← build de l'image Docker
├── Jenkinsfile           ← définition du pipeline CI/CD
├── README.md
└── app/
    ├── app.py            ← application Flask
    ├── requirements.txt  ← dépendances Python
    └── tests/
        └── test_app.py   ← tests unitaires pytest
```

---

## Mise en place de l'environnement

### 1. Lancer Jenkins

```bash
docker run -d -p 9090:8080 -p 50000:50000 \
  --name jenkins \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

Récupère le mot de passe initial :
```bash
docker logs jenkins
```

Accède à Jenkins sur `http://localhost:9090` et installe les plugins suggérés.

### 2. Créer le réseau Docker

```bash
docker network create jenkins-network
docker network connect jenkins-network jenkins
```

### 3. Créer les 3 agents (conteneurs)

```bash
for container in prod stage review; do
  docker run -d --name $container \
    --network jenkins-network \
    -v /var/run/docker.sock:/var/run/docker.sock \
    ubuntu:22.04 sleep infinity
  echo "$container créé ✅"
done
```

### 4. Installer les dépendances dans chaque agent

```bash
for container in prod stage review; do
  docker exec $container apt-get update -q
  docker exec $container apt-get install -y git docker.io openjdk-17-jdk openssh-server
  docker exec $container mkdir -p /var/run/sshd
  docker exec $container service ssh start
  echo "$container configuré ✅"
done
```

### 5. Configurer SSH entre Jenkins et les agents

Générer une clé SSH dans Jenkins :
```bash
docker exec -it jenkins ssh-keygen -t rsa -b 4096 \
  -f /var/jenkins_home/.ssh/id_rsa -N ""
```

Copier la clé publique dans chaque agent :
```bash
PUB_KEY=$(docker exec jenkins cat /var/jenkins_home/.ssh/id_rsa.pub)

for container in prod stage review; do
  docker exec $container mkdir -p /root/.ssh
  docker exec $container bash -c "echo '$PUB_KEY' >> /root/.ssh/authorized_keys"
  docker exec $container chmod 600 /root/.ssh/authorized_keys
  echo "$container clé SSH ✅"
done
```

Récupérer les IPs des agents pour la config Jenkins :
```bash
for container in prod stage review; do
  echo -n "$container : "
  docker inspect $container \
    --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
done
```

### 6. Configurer les agents dans Jenkins

Dans **Manage Jenkins → Nodes → New Node** :

| Champ | Valeur |
|-------|--------|
| Nom | agent-review / agent-stage / agent-prod |
| Type | Permanent Agent |
| Remote root directory | /var/jenkins |
| Label | review / stage / prod |
| Méthode de lancement | Launch agents via SSH |
| Host | IP du conteneur correspondant |
| Credentials | SSH Username with private key (root + clé privée Jenkins) |
| Host Key Verification | Non verifying |

Récupérer la clé privée Jenkins pour les credentials :
```bash
docker exec jenkins cat /var/jenkins_home/.ssh/id_rsa
```

### 7. Ajouter les credentials Docker Hub

Dans **Manage Jenkins → Credentials → Global → Add Credentials** :

| Champ | Valeur |
|-------|--------|
| Kind | Username with password |
| ID | dockerhub-credentials |
| Username | ton_username_dockerhub |
| Password | token Docker Hub (pas le mot de passe) |

Générer un token Docker Hub :
```
hub.docker.com → Account Settings
→ Personal access tokens → Generate new token
```

---

## Contenu des fichiers

### app/app.py

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def home():
    return "Hello DevOps Lab ! 🚀"

@app.route('/health')
def health():
    return {"status": "ok"}, 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### app/requirements.txt

```
flask==3.0.0
pytest==7.4.0
```

### app/tests/test_app.py

```python
import sys
sys.path.insert(0, '..')
from app import app

def test_home():
    client = app.test_client()
    response = client.get('/')
    assert response.status_code == 200

def test_health():
    client = app.test_client()
    response = client.get('/health')
    assert response.status_code == 200
```

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY app/requirements.txt .
RUN pip install -r requirements.txt

COPY app/ .

EXPOSE 5000

CMD ["python3", "app.py"]
```

**Explication ligne par ligne :**

| Instruction | Rôle |
|-------------|------|
| `FROM python:3.11-slim` | Image de base Python légère |
| `WORKDIR /app` | Dossier de travail dans le conteneur |
| `COPY app/requirements.txt .` | Copie les dépendances |
| `RUN pip install -r requirements.txt` | Installe les dépendances |
| `COPY app/ .` | Copie le code source |
| `EXPOSE 5000` | Documente le port de l'app |
| `CMD ["python3", "app.py"]` | Commande de démarrage |

### Jenkinsfile

```groovy
pipeline {
    agent none

    environment {
        DOCKER_HUB_USER = 'ton_username_dockerhub'
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
                sh "docker tag ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} \
                    ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
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
        success {
            echo "Pipeline reussi ! Image: ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline echoue !"
        }
    }
}
```

**Explication des blocs Jenkinsfile :**

| Bloc | Rôle |
|------|------|
| `agent none` | Pas d'agent global, chaque stage définit le sien |
| `environment` | Variables globales du pipeline |
| `BUILD_NUMBER` | Numéro de build auto-incrémenté par Jenkins |
| `checkout scm` | Clone le repo GitHub configuré dans le pipeline |
| `withCredentials` | Injecte les secrets sans les afficher dans les logs |
| `docker stop \|\| true` | Stoppe le conteneur sans échouer s'il n'existe pas |
| `when { branch 'main' }` | Execute ce stage uniquement sur la branche main |
| `post { success/failure }` | Actions après le pipeline selon le résultat |

---

## Créer le pipeline dans Jenkins

1. **New Item** → nom : `jenkins-python-app` → type : **Pipeline**
2. **Pipeline** section :
   - Definition : `Pipeline script from SCM`
   - SCM : `Git`
   - Repository URL : `https://github.com/ton_user/jenkins_lab`
   - Branch : `*/main`
   - Script Path : `Jenkinsfile`
3. **Build Triggers** → cocher `GitHub hook trigger for GITScm polling`
4. **Save** → **Build Now**

---

## Vérifications

```bash
# Vérifier que les agents sont connectés
# Manage Jenkins → Nodes → tous en vert ✅

# Vérifier les conteneurs qui tournent
docker ps

# Vérifier l'app sur chaque environnement
curl http://localhost:5001  # Review
curl http://localhost:5002  # Staging
curl http://localhost:5003  # Production

# Logs d'un conteneur
docker logs app-review
docker logs app-stage
docker logs app-prod
```

---

## Commandes utiles

```bash
# Voir tous les conteneurs
docker ps

# Redémarrer un agent après reboot
docker start prod stage review jenkins

# Redémarrer SSH dans les agents
for container in prod stage review; do
  docker exec $container service ssh start
done

# Voir les logs Jenkins
docker logs jenkins

# Vérifier les IPs des agents (changent après redémarrage)
for container in prod stage review; do
  echo -n "$container : "
  docker inspect $container \
    --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
done

# Supprimer tout et recommencer
docker stop prod stage review jenkins
docker rm prod stage review jenkins
docker volume rm jenkins_home
```

---

## Concepts clés

| Concept | Explication |
|---------|-------------|
| CI (Continuous Integration) | Build + Test automatiques à chaque push |
| CD (Continuous Deployment) | Deploy automatique sur les environnements |
| Agent Jenkins | Machine worker qui exécute les stages |
| Jenkinsfile | Pipeline as Code — définit toutes les étapes |
| Docker Hub | Registry pour stocker et partager les images |
| BUILD_NUMBER | Numéro unique de build, utilisé comme tag d'image |
| `when { branch }` | Condition pour exécuter un stage selon la branche |

---

## Résultat final

```
✅ Checkout        — code récupéré depuis GitHub
✅ Build           — image Docker construite
✅ Test            — tests pytest passés
✅ Push Docker Hub — image poussée sur fenitrar07/jenkins-python-app:5
✅ Deploy Review   — app tournant sur port 5001
✅ Deploy Staging  — app tournant sur port 5002
✅ Deploy Prod     — app tournant sur port 5003 (branch main)
```
