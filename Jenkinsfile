pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "jatinjaiswal"
    }

    stages {
        stage('Checkout Code') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                    rm -rf repo || true
                    git clone https://$GITHUB_TOKEN@github.com/Jatin-Jaiswal/jenkins-practice.git repo
                    '''
                }
            }
        }

        stage('Prepare Backend Env') {
            steps {
                dir('repo/server') {
                    withCredentials([file(credentialsId: 'backend-env', variable: 'BACKEND_ENV_FILE')]) {
                        sh 'cp $BACKEND_ENV_FILE .env'
                        sh 'ls -la .'
                        sh 'cat .env'
                    }
                }
            }
        }

        stage('Prepare Frontend Env') {
            steps {
                dir('repo/client') {
                    withCredentials([file(credentialsId: 'frontend-env', variable: 'FRONTEND_ENV_FILE')]) {
                        sh 'cp $FRONTEND_ENV_FILE .env'
                        sh 'ls -la .'
                        sh 'cat .env'
                    }
                }
            }
        }

        stage('SonarQube Analysis - Server') {
            steps {
                dir('repo/server') {
                    withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            echo "🔍 Running SonarQube Analysis for Server..."
                            /usr/bin/sonar-scanner \
                              -Dsonar.projectKey=jenkins-practice-server \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=http://localhost:9000 \
                              -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('SonarQube Analysis - Client') {
            steps {
                dir('repo/client') {
                    withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            echo "🔍 Running SonarQube Analysis for Client..."
                            /usr/bin/sonar-scanner \
                              -Dsonar.projectKey=jenkins-practice-client \
                              -Dsonar.sources=src \
                              -Dsonar.host.url=http://localhost:9000 \
                              -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                echo "🚀 Building Docker images..."
                docker build --no-cache -t $DOCKERHUB_USER/backend:$BUILD_NUMBER ./repo/server
                docker build --no-cache -t $DOCKERHUB_USER/frontend:$BUILD_NUMBER ./repo/client
                '''
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                    echo "🔑 Logging into Docker Hub..."
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                    echo "📤 Pushing images..."
                    docker push $DOCKERHUB_USER/backend:$BUILD_NUMBER
                    docker push $DOCKERHUB_USER/frontend:$BUILD_NUMBER
                    '''
                }
            }
        }

        stage('Deploy Locally') {
            steps {
                sh '''
                echo "🛑 Stopping old containers..."
                docker rm -f backend-container || true
                docker rm -f frontend-container || true

                echo "📥 Pulling latest images..."
                docker pull $DOCKERHUB_USER/backend:$BUILD_NUMBER
                docker pull $DOCKERHUB_USER/frontend:$BUILD_NUMBER

                echo "▶️ Starting new containers..."
                docker run -d -p 5000:5000 --name backend-container $DOCKERHUB_USER/backend:$BUILD_NUMBER
                docker run -d -p 3000:3000 --name frontend-container $DOCKERHUB_USER/frontend:$BUILD_NUMBER
                '''
            }
        }

    }

    post {
        success {
            echo "✅ Deployment successful! Frontend: http://localhost:3000 | Backend: http://localhost:5000"
        }
        failure {
            echo "❌ Build or deployment failed. Check Jenkins logs."
        }
    }
}