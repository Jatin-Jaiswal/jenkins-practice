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
                    }
                }
            }   
        }

        stage('Prepare Frontend Env') {
    steps {
        dir('repo/client') {
            withCredentials([file(credentialsId: 'frontend-env', variable: 'FRONTEND_ENV_FILE')]) {
                sh '''
                # Copy the secret .env
                cp $FRONTEND_ENV_FILE .env

                # Fetch backend LoadBalancer IP dynamically from GKE
                BACKEND_IP=$(kubectl get svc server-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

                # Replace backend URL in .env
                sed -i "s|REACT_APP_API_URL=.*|REACT_APP_API_URL=http://$BACKEND_IP:5000|" .env

                # Show final .env for verification
                cat .env
                '''
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

        stage('GCP Authentication') {
            steps {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
                    sh '''
                    echo "🔑 Authenticating with GCP..."
                    gcloud auth activate-service-account --key-file=$GCP_KEY
                    gcloud config set project jenkins-practice-gke
                    gcloud container clusters get-credentials mern-cluster --zone us-central1-a --project jenkins-practice-gke
                    '''
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                sh '''
                echo "📦 Updating Kubernetes deployments..."

                # Update backend deployment
                kubectl set image deployment/server-deployment server=$DOCKERHUB_USER/backend:$BUILD_NUMBER --record

                # Update frontend deployment
                kubectl set image deployment/client-deployment client=$DOCKERHUB_USER/frontend:$BUILD_NUMBER --record

                # Wait for rollout to complete
                kubectl rollout status deployment/server-deployment
                kubectl rollout status deployment/client-deployment
                '''
            }
        }

    }

    post {
        success {
            echo "✅ Deployment successful! Backend & Frontend updated on GKE."
        }
        failure {
            echo "❌ Build or deployment failed. Check Jenkins logs."
        }
    }
}
