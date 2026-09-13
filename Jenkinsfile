pipeline {
    // Top level par koi fixed agent nahi - branch decide karega
    agent none

    environment {
        DOCKERHUB_USERNAME = 'saradapradhan'   // BADLEIN
        BACKEND_IMAGE      = "${DOCKERHUB_USERNAME}/travelmemory-backend"
        FRONTEND_IMAGE     = "${DOCKERHUB_USERNAME}/travelmemory-frontend"
    }

    stages {
        stage('Branch Pipeline') {
            // YAHI JAADU HAI: main -> prod worker, warna -> staging worker
            agent { label "${env.BRANCH_NAME == 'main' ? 'prod' : 'staging'}" }

            environment {
                // Har environment ki apni values
                TAG        = "${env.BRANCH_NAME}-${BUILD_NUMBER}"
                MONGO_CRED = "${env.BRANCH_NAME == 'main' ? 'mongo-uri-prod' : 'mongo-uri-staging'}"
            }

            stages {
                stage('Checkout & Verify Worker') {
                    steps {
                        echo "Branch: ${env.BRANCH_NAME}"
                        // Ye hostname confirm karega ki sahi worker par chal raha hai
                        sh 'hostname'
                    }
                }

                stage('Build Images') {
                    steps {
                        sh 'docker build -t "$BACKEND_IMAGE:$TAG" ./backend'
                        sh 'docker build -t "$FRONTEND_IMAGE:$TAG" ./frontend'
                    }
                }

                stage('Push to DockerHub') {
                    steps {
                        withCredentials([usernamePassword( 
                            credentialsId: 'docker-hub-creds',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS')]) {
                            sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                            sh 'docker push "$BACKEND_IMAGE:$TAG"'
                            sh 'docker push "$FRONTEND_IMAGE:$TAG"'
                        }
                    }
                }

                stage('Deploy') {
                    steps {
                        withCredentials([string(credentialsId: env.MONGO_CRED,
                                                variable: 'MONGO_URI')]) {
                            sh '''
                                docker network create travel-net || true

                                docker rm -f mongo || true
                                docker run -d --name mongo --network travel-net \
                                  --restart unless-stopped \
                                  -v mongo-data:/data/db mongo:7

                                docker rm -f backend || true
                                docker run -d --name backend --network travel-net \
                                  --restart unless-stopped \
                                  -e PORT=3001 -e MONGO_URI="$MONGO_URI" \
                                  "$BACKEND_IMAGE:$TAG"

                                docker rm -f frontend || true
                                docker run -d --name frontend --network travel-net \
                                  --restart unless-stopped \
                                  -p 80:80 "$FRONTEND_IMAGE:$TAG"
                            '''
                        }
                    }
                }
            }

            post {
                always {
                    sh 'docker logout || true'
                }
            }
        }
    }
}