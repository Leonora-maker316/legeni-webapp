pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timeout(time: 1, unit: 'HOURS')
    }

    environment {
        IMAGE           = "leonoranams/legeni-webapp"
        VERSION         = "1.0.${env.BUILD_NUMBER}"
        COMPOSE_STAGING = "docker compose -f deploy/docker-compose.deploy.yml -p legeni-staging"
        COMPOSE_PROD    = "docker compose -f deploy/docker-compose.deploy.yml -p legeni-prod"
    }

    stages {
        stage('Checkout Code') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                echo "Construction de l'image Docker ${env.IMAGE}:${env.VERSION}..."
                sh "docker build -t ${env.IMAGE}:${env.VERSION} -t ${env.IMAGE}:latest ."
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USER')]) {
                    sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USER --password-stdin"
                    sh "docker push ${env.IMAGE}:${env.VERSION}"
                    sh "docker push ${env.IMAGE}:latest"
                }
            }
        }

        stage('Deploy Staging') {
            steps {
                echo "Déploiement sur l'environnement de Staging (Port 8081)..."
                script {
                    env.APP_ENV = "staging"
                    env.PORT = "8081"
                    env.APP_COLOR = "#bf8700"
                    env.IMAGE_REF = "${env.IMAGE}:${env.VERSION}"
                }
                withCredentials([string(credentialsId: 'redis-password-staging', variable: 'REDIS_PASSWORD')]) {
                    sh "${env.COMPOSE_STAGING} up -d --build"
                }
            }
        }

        stage('Validation Production') {
            steps {
                input message: "Valider le passage en Production (Port 8082) ?", ok: "Déployer"
            }
        }

        stage('Deploy Production & Rollback') {
            steps {
                script {
                    def previousImage = sh(script: "docker inspect --format='{{.Config.Image}}' legeni-prod-web-1 2>/dev/null || echo '${env.IMAGE}:latest'", returnStdout: true).trim()
                    
                    try {
                        echo "Déploiement sur l'environnement de Production (Port 8082)..."
                        env.APP_ENV = "production"
                        env.PORT = "8082"
                        env.APP_COLOR = "#28a745"
                        env.IMAGE_REF = "${env.IMAGE}:${env.VERSION}"
                        
                        withCredentials([string(credentialsId: 'redis-password-prod', variable: 'REDIS_PASSWORD')]) {
                            sh "${env.COMPOSE_PROD} up -d --build"
                        }
                        
                        // Laisse le temps au conteneur de démarrer et de se stabiliser
                        sh "sleep 10"
                        
                        // Test de santé robuste testant l'IP hôte/passerelle et localhost
                        sh "curl -f http://172.17.0.1:8082/ || curl -f http://localhost:8082/ || exit 1"
                        echo "✅ Déploiement en production réussi !"
                    } catch (Exception e) {
                        echo "❌ Échec du déploiement en production ! Lancement du rollback vers ${previousImage}..."
                        env.IMAGE_REF = previousImage
                        withCredentials([string(credentialsId: 'redis-password-prod', variable: 'REDIS_PASSWORD')]) {
                            sh "${env.COMPOSE_PROD} up -d"
                        }
                        error("Le déploiement a échoué, le rollback a été appliqué avec succès.")
                    }
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline exécuté avec succès du début à la fin !"
        }
        failure {
            echo "💥 Le pipeline a échoué à l'une des étapes."
        }
    }
}