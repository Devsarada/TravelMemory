pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = 'saradapradhan'   // BADLEIN
        BACKEND_IMAGE      = "${DOCKERHUB_USERNAME}/travelmemory-backend"
        FRONTEND_IMAGE     = "${DOCKERHUB_USERNAME}/travelmemory-frontend"
        TAG                = "${BUILD_NUMBER}"
        EMAIL_RECIPIENTS   = 'saradapradhan2432gmail.com'           // BADLEIN
        // SLACK_CHANNEL      = '#devops-alerts'             // BADLEIN
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Code SCM se checkout ho gaya'
                sh 'ls -la'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key',
                                        variable: 'NVD_API_KEY')]) {
                    dependencyCheck(
                        additionalArguments: '--scan ./ --format XML --format HTML --nvdApiKey $NVD_API_KEY',
                        odcInstallation: 'DP-Check'
                    )
                }
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh 'trivy fs . --severity HIGH,CRITICAL --format table -o trivy-fs-report.txt'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    script {
                        def scannerHome = tool 'sonar-scanner'
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Backend Image') {
            steps {
                sh "docker build -t ${BACKEND_IMAGE}:${TAG} -t ${BACKEND_IMAGE}:latest ./backend"
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh "docker build -t ${FRONTEND_IMAGE}:${TAG} -t ${FRONTEND_IMAGE}:latest ./frontend"
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --severity HIGH,CRITICAL --format table -o trivy-backend-image.txt ${BACKEND_IMAGE}:${TAG}"
                sh "trivy image --severity HIGH,CRITICAL --format table -o trivy-frontend-image.txt ${FRONTEND_IMAGE}:${TAG}"
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                    sh "docker push ${BACKEND_IMAGE}:${TAG}"
                    sh "docker push ${BACKEND_IMAGE}:latest"
                    sh "docker push ${FRONTEND_IMAGE}:${TAG}"
                    sh "docker push ${FRONTEND_IMAGE}:latest"
                }
            }
        }

        stage('Deploy 3-Tier App') {
            steps {
                withCredentials([string(credentialsId: 'mongo-uri',
                                        variable: 'MONGO_URI')]) {
                    sh '''
                        docker network create travel-net || true

                        docker rm -f mongo || true
                        docker run -d --name mongo \
                          --network travel-net \
                          --restart unless-stopped \
                          -v mongo-data:/data/db \
                          mongo:7

                        docker rm -f backend || true
                        docker run -d --name backend \
                          --network travel-net \
                          --restart unless-stopped \
                          -e PORT=3001 \
                          -e MONGO_URI="$MONGO_URI" \
                          "$BACKEND_IMAGE:$TAG"

                        docker rm -f frontend || true
                        docker run -d --name frontend \
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
            archiveArtifacts artifacts: 'trivy-*.txt,**/dependency-check-report.*',
                             allowEmptyArchive: true
            sh 'docker logout || true'
        }
        success {
           // slackSend(
               // channel: env.SLACK_CHANNEL,
               // color: 'good',
               // message: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} deployed " +
                        // "(<${env.BUILD_URL}|open build>)"
           // )
            emailext(
                to: env.EMAIL_RECIPIENTS,
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-*.txt',
                body: "<h3>Build Successful</h3>" +
                      "<p>Job: ${env.JOB_NAME} #${env.BUILD_NUMBER}<br/>" +
                      "URL: <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>" +
                      "<p>App live: http://&lt;EC2-IP&gt; | Trivy reports attached.</p>"
            )
        }
        failure {
            //slackSend(
              //  channel: env.SLACK_CHANNEL,
               // color: 'danger',
                //message: "FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER} failed " +
                       //  "(<${env.BUILD_URL}console|check logs>)"
           // )
            emailext(
                to: env.EMAIL_RECIPIENTS,
                subject: "FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                mimeType: 'text/html',
                body: "<h3>Build Failed</h3>" +
                      "<p>Job: ${env.JOB_NAME} #${env.BUILD_NUMBER}<br/>" +
                      "Console: <a href='${env.BUILD_URL}console'>view logs</a></p>"
            )
        }
    }
}