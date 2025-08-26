pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "jatinjaiswal"   // your Docker Hub username
    }

    stages {
        stage('Checkout Code') {
            steps {
                // Using Secret Text for GitHub token
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                    # Clone repo using token
                    rm -rf repo || true
                    git clone https://$GITHUB_TOKEN@github.com/Jatin-Jaiswal/your-repo.git repo
                    cd repo
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    sh '''
                    echo "🚀 Building Docker images..."
                    docker build -t $DOCKERHUB_USER/backend:$BUILD_NUMBER ./repo/server
                    docker build -t $DOCKERHUB_USER/frontend:$BUILD_NUMBER ./repo/client
                    '''
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                script {
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
        }

        stage('Deploy Locally') {
            steps {
                script {
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
