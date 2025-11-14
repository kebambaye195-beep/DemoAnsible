pipeline {
    agent any

    environment {
    DOCKER_USER = 'kebambaye195-beep'
    FRONT_IMAGE = 'express-frontend'
    BACK_IMAGE  = 'express-backend'
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo '📦 Récupération du code...'
                checkout scm
            }
        }

        stage('Build Images') {
            parallel {
                stage('Build Backend') {
                    steps {
                        script {
                            echo '🔨 Construction Backend...'
                            sh """
                                docker build -t ${DOCKERHUB_USERNAME}/${IMAGE_BACKEND}:${BUILD_NUMBER} \
                                             -t ${DOCKERHUB_USERNAME}/${IMAGE_BACKEND}:latest \
                                             -f backend/Dockerfile ./backend
                            """
                        }
                    }
                }

                stage('Build Frontend') {
                    steps {
                        script {
                            echo '🔨 Construction Frontend...'
                            sh """
                                docker build -t ${DOCKERHUB_USERNAME}/${IMAGE_FRONTEND}:${BUILD_NUMBER} \
                                             -t ${DOCKERHUB_USERNAME}/${IMAGE_FRONTEND}:latest \
                                             -f Dockerfile . 
                            """
                        }
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    sh '''
                        echo "🔍 Vérification de Trivy..."
                        if ! command -v trivy &> /dev/null; then
                            echo "📦 Installation de Trivy..."
                            curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
                        fi

                        mkdir -p trivy-reports

                        echo "🧪 Scan des images Docker..."
                        # Scan image frontend
                        trivy image --no-progress --severity UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL \
                            --ignore-unfixed=false \
                            -f table -o trivy-reports/frontend-scan.txt \
                            -f json -o trivy-reports/frontend-scan.json \
                            ${DOCKERHUB_USERNAME}/${IMAGE_FRONTEND}:latest

                        # Scan image backend
                        trivy image --no-progress --severity UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL \
                            --ignore-unfixed=false \
                            -f table -o trivy-reports/backend-scan.txt \
                            -f json -o trivy-reports/backend-scan.json \
                            ${DOCKERHUB_USERNAME}/${IMAGE_BACKEND}:latest

                        echo "🧪 Scan du filesystem (Node + projet)..."
                        trivy fs --no-progress --severity UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL \
                            --ignore-unfixed=false \
                            -f table -o trivy-reports/fs-backend.txt ./backend
                        trivy fs --no-progress --severity UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL \
                            --ignore-unfixed=false \
                            -f table -o trivy-reports/fs-frontend.txt ./frontend

                        echo "✅ Scan Trivy terminé."
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-reports/*.*', fingerprint: true
                }
            }
        }

        stage('Create .env') {
            steps {
                script {
                    echo '🔐 Création du fichier .env...'
                    sh '''
                        mkdir -p backend
                        cat > backend/.env << 'ENVFILE'
PORT=5000
MONGO_URI=mongodb://mongo:27017/smartphoneDB
DELETE_CODE=123
ENVFILE
                        echo "✅ Fichier .env créé"
                        cat backend/.env
                    '''
                }
            }
        }

        stage('Deploy Local (Docker Compose)') {
            steps {
                script {
                    echo '🚀 Déploiement local avec Docker Compose...'
                    sh '''
                        docker compose down --remove-orphans || true
                        docker compose pull
                        docker compose up -d
                        sleep 5
                        docker compose ps
                        docker logs backend --tail 20
                    '''
                }
            }
        }

        stage('Terraform Deploy') {
            steps {
                script {
                    echo '📦 Déploiement Kubernetes avec Terraform...'
                }
                dir('terraform') {
                    sh 'terraform init'
                    sh 'terraform apply -auto-approve'
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline terminé avec succès !'
        }
        failure {
            echo '❌ Pipeline échoué. Vérifie les logs Jenkins.'
        }
    }
}
