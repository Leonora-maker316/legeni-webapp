pipeline {
    agent any

    options {
        skipDefaultCheckout(true) // Évite les conflits et l'erreur "not in a git directory"
        timeout(time: 1, unit: 'HOURS')
        ansiColor('xterm')
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
                cleanWs() // Nettoie complètement le workspace pour repartir sur du propre
                checkout scm // Récupère le code depuis GitHub
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
                // Assure-toi d'avoir configuré les credentials Docker Hub dans Jenkins si nécessaire, 
                // ou commente cette étape si tu fais les builds entièrement en local sur la machine.
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USER')]) {
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
                sh "${env.COMPOSE_STAGING} up -d --build"
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
                    // Récupération de l'ancienne version active pour un éventuel rollback
                    def previousImage = sh(script: "docker inspect --format='{{.Config.Image}}' legeni-prod-web-1 2>/dev/null || echo '${env.IMAGE}:latest'", returnStdout: true).trim()
                    
                    try {
                        echo "Déploiement sur l'environnement de Production (Port 8082)..."
                        env.APP_ENV = "production"
                        env.PORT = "8082"
                        env.APP_COLOR = "#28a745"
                        env.IMAGE_REF = "${env.IMAGE}:${env.VERSION}"
                        
                        sh "${env.COMPOSE_PROD} up -d --build"
                        
                        // Test de santé (Smoke Test)
                        sh "sleep 5"
                        sh "curl -f http://localhost:8082/ || exit 1"
                        echo "✅ Déploiement en production réussi !"
                    } catch (Exception e) {
                        echo "❌ Échec du déploiement en production ! Lancement du rollback vers ${previousImage}..."
                        env.IMAGE_REF = previousImage
                        sh "${env.COMPOSE_PROD} up -d"
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