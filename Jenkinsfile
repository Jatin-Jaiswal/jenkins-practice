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

        stage('GCP Authentication') {
            steps {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
                    # Authenticate with GCP service account
                    gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS

                    # Set the correct project
                    gcloud config set project jenkins-practice-gke

                    # Get credentials for your GKE cluster
                    gcloud container clusters get-credentials jenkins-cluster --zone us-central1-a
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
                        cp $FRONTEND_ENV_FILE .env
                        # Dynamically fetch backend LoadBalancer IP
                        BACKEND_IP=$(kubectl get svc server-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                        if [ -z "$BACKEND_IP" ]; then
                            echo "ERROR: Backend IP not found. Exiting..."
                            exit 1
                        fi
                        # Replace in frontend .env
                        sed -i "s|REACT_APP_API_URL=.*|REACT_APP_API_URL=http://$BACKEND_IP:5000|" .env
                        '''
                    }
                }
            }
        }

        stage('SonarQube Analysis - Server') {
            steps {
                echo 'Skipping SonarQube analysis for server.'
            }
        }

        stage('SonarQube Analysis - Client') {
            steps {
                echo 'Skipping SonarQube analysis for client.'
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                docker build -t $DOCKERHUB_USER/backend:$BUILD_NUMBER repo/server
                docker build -t $DOCKERHUB_USER/frontend:$BUILD_NUMBER repo/client
                '''
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    docker push $DOCKERHUB_USER/backend:$BUILD_NUMBER
                    docker push $DOCKERHUB_USER/frontend:$BUILD_NUMBER
                    '''
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                sh '''
                kubectl set image deployment/server-deployment server=$DOCKERHUB_USER/backend:$BUILD_NUMBER
                kubectl set image deployment/client-deployment client=$DOCKERHUB_USER/frontend:$BUILD_NUMBER
                kubectl rollout status deployment/server-deployment
                kubectl rollout status deployment/client-deployment
                '''
            }
        }
    }

    post {
        failure {
            echo "❌ Build or deployment failed. Check Jenkins logs."
        }
    }
}