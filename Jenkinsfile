pipeline {
    agent none

    environment {
        DOCKERHUB_USERNAME = 'saradapradhan'
        BACKEND_IMAGE      = "${DOCKERHUB_USERNAME}/travelmemory-backend"
        FRONTEND_IMAGE     = "${DOCKERHUB_USERNAME}/travelmemory-frontend"
    }

    stages {
        stage('Branch Pipeline') {

            agent {
                label "${env.BRANCH_NAME == 'main' ? 'prod' : 'staging'}"
            }

            environment {
                TAG        = "${env.BRANCH_NAME}-${BUILD_NUMBER}"
                MONGO_CRED = "${env.BRANCH_NAME == 'main' ? 'mongo-uri-prod' : 'mongo-uri-staging'}"
            }

            stages {

                stage('Checkout & Verify Worker') {
                    steps {
                        echo "Branch: ${env.BRANCH_NAME}"
                        echo "Build Number: ${env.BUILD_NUMBER}"

                        sh 'hostname'

                        // GitHub token available only inside this block
                        withCredentials([
                            string(
                                credentialsId: 'github-token',
                                variable: 'GITHUB_TOKEN'
                            )
                        ]) {
                            sh '''
                                echo "Checking GitHub authentication..."

                                curl -s \
                                  -H "Authorization: Bearer $GITHUB_TOKEN" \
                                  -H "Accept: application/vnd.github+json" \
                                  https://api.github.com/rate_limit \
                                  | head -c 1000
                            '''
                        }
                    }
                }

                stage('Build Images') {
                    steps {
                        sh '''
                            docker build \
                              -t "$BACKEND_IMAGE:$TAG" \
                              ./backend

                            docker build \
                              -t "$FRONTEND_IMAGE:$TAG" \
                              ./frontend
                        '''
                    }
                }

                stage('Push to DockerHub') {
                    steps {
                        withCredentials([
                            usernamePassword(
                                credentialsId: 'docker-hub-creds',
                                usernameVariable: 'DOCKER_USER',
                                passwordVariable: 'DOCKER_PASS'
                            )
                        ]) {
                            sh '''
                                echo "$DOCKER_PASS" | docker login \
                                  -u "$DOCKER_USER" \
                                  --password-stdin

                                docker push "$BACKEND_IMAGE:$TAG"
                                docker push "$FRONTEND_IMAGE:$TAG"
                            '''
                        }
                    }
                }

                stage('Deploy') {
                    steps {
                        withCredentials([
                            string(
                                credentialsId: env.MONGO_CRED,
                                variable: 'MONGO_URI'
                            )
                        ]) {
                            sh '''
                                docker network create travel-net || true

                                docker rm -f mongo || true

                                docker run -d \
                                  --name mongo \
                                  --network travel-net \
                                  --restart unless-stopped \
                                  -v mongo-data:/data/db \
                                  mongo:7

                                docker rm -f backend || true

                                docker run -d \
                                  --name backend \
                                  --network travel-net \
                                  --restart unless-stopped \
                                  -e PORT=3001 \
                                  -e MONGO_URI="$MONGO_URI" \
                                  "$BACKEND_IMAGE:$TAG"

                                docker rm -f frontend || true

                                docker run -d \
                                  --name frontend \
                                  --network travel-net \
                                  --restart unless-stopped \
                                  -p 80:80 \
                                  "$FRONTEND_IMAGE:$TAG"
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