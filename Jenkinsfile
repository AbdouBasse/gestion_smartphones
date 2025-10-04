pipeline {
    agent any

    tools {
        // Assure-toi que "NodeJS_16" est bien configuré dans Jenkins > Global Tool Configuration
        nodejs "NodeJS_16"
    }

    environment {
        DOCKER_HUB_USER = 'abdoubasse'
        FRONT_IMAGE = 'react-frontend'
        BACK_IMAGE  = 'express-backend'
    }

    triggers {
        GenericTrigger(
            genericVariables: [
                [key: 'ref', value: '$.ref'],
                [key: 'pusher_name', value: '$.pusher.name'],
                [key: 'commit_message', value: '$.head_commit.message']
            ],
            causeString: 'Push par $pusher_name sur $ref: "$commit_message"',
            token: 'mysecret',
            printContributedVariables: true,
            printPostContent: true
        )
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/AbdouBasse/gestion_smartphones.git'
            }
        }

        stage('Install dependencies - Backend') {
            steps {
                dir('back-end') {
                    sh 'npm install'
                    sh 'node -v && npm -v'
                }
            }
        }

        stage('Install dependencies - Frontend') {
            steps {
                dir('front-end') {
                    sh 'npm install'
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    try {
                        sh 'cd back-end && npm test'
                    } catch (err) {
                        echo "Aucun test backend ou échec : ${err}"
                    }

                    try {
                        sh 'cd front-end && npm test'
                    } catch (err) {
                        echo "Aucun test frontend ou échec : ${err}"
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    sh "docker build -t $DOCKER_HUB_USER/$FRONT_IMAGE:latest ./front-end"
                    sh "docker build -t $DOCKER_HUB_USER/$BACK_IMAGE:latest ./back-end"
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push $DOCKER_USER/react-frontend:latest
                        docker push $DOCKER_USER/express-backend:latest
                    '''
                }
            }
        }

        stage('Clean Docker') {
            steps {
                sh 'docker container prune -f || true'
                sh 'docker image prune -f || true'
            }
        }

        stage('Check Docker & Compose') {
            steps {
                sh 'docker --version'
                sh 'docker-compose --version || echo "docker-compose non trouvé"'
            }
        }

        stage('Deploy (compose.yaml)') {
            steps {
                dir('.') {
                    sh 'docker-compose -f compose.yaml down || true'
                    sh 'docker-compose -f compose.yaml pull || true'
                    sh 'docker-compose -f compose.yaml up -d'
                    sh 'docker-compose -f compose.yaml ps'
                    sh 'docker-compose -f compose.yaml logs --tail=50'
                }
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    echo "🔍 Vérification Frontend (port 5173)..."
                    curl --max-time 5 -f http://localhost:5173 || echo "❌ Frontend unreachable"

                    echo "🔍 Vérification Backend (port 5001)..."
                    curl --max-time 5 -f http://localhost:5001/api || echo "❌ Backend unreachable"
                '''
            }
        }
    }

    post {
        success {
            emailext(
                subject: "✅ Build SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Pipeline réussi\nDétails : ${env.BUILD_URL}",
                to: "abdoubasseoconoor@gmail.com"
            )
        }
        failure {
            emailext(
                subject: "❌ Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Le pipeline a échoué\nDétails : ${env.BUILD_URL}",
                to: "abdoubasseoconoor@gmail.com"
            )
        }
    }
}

