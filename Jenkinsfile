pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = 'saradapradhan'
        BACKEND_IMAGE      = "${DOCKERHUB_USERNAME}/travelmemory-backend" // saradapradhan/travelmemory-backend
        FRONTEND_IMAGE     = "${DOCKERHUB_USERNAME}/travelmemory-frontend" // saradapradhan/travelmemory-frontend
        TAG                = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Code checkout done'
                sh 'ls -la'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                withCredentials([string(
                    credentialsId: 'nvd-api-key',
                    variable: 'NVD_API_KEY'
                )]) {

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
                // context ./backend - Dockerfile yahin hai
                sh "docker build -t ${BACKEND_IMAGE}:${TAG} -t ${BACKEND_IMAGE}:latest ./backend"
            } // saradapradhan/travelmemory-backend:9 + saradapradhan/travelmemory-backend:latest
        }

        stage('Build Frontend Image') {
            steps {
                sh "docker build -t ${FRONTEND_IMAGE}:${TAG} -t ${FRONTEND_IMAGE}:latest ./frontend"
            }
        }

        stage('Trivy Image Scan') {
            steps {
                // Push se PEHLE image scan
                sh "trivy image --severity HIGH,CRITICAL --format table -o trivy-backend-image.txt ${BACKEND_IMAGE}:${TAG}"
                sh "trivy image --severity HIGH,CRITICAL --format table -o trivy-frontend-image.txt ${FRONTEND_IMAGE}:${TAG}"
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''

                    sh "docker push ${BACKEND_IMAGE}:${TAG}"
                    sh "docker push ${BACKEND_IMAGE}:latest"
                    sh "docker push ${FRONTEND_IMAGE}:${TAG}"
                    sh "docker push ${FRONTEND_IMAGE}:latest"
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
            echo 'Images built, scanned, pushed!'
        }

        failure {
            echo 'Fail hua - logs/reports dekhein'
        }
    }
}