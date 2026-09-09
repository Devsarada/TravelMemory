pipeline {
    agent any

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
                        additionalArguments: '''
                            --scan ./
                            --format XML
                            --format HTML
                            --nvdApiKey $NVD_API_KEY
                        ''',
                        odcInstallation: 'DP-Check'
                    )

                    // Report Jenkins UI par dikhane ke liye
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                // Report mode: HIGH,CRITICAL findings ek file mein
                sh '''
                    trivy fs . \
                        --severity HIGH,CRITICAL \
                        --format table \
                        -o trivy-fs-report.txt
                '''
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
    }

    post {
        always {
            // Saari reports download ke liye archive
            archiveArtifacts artifacts: 'trivy-*.txt,**/dependency-check-report.*',
            allowEmptyArchive: true
        }

        success {
            echo 'SonarQube + Quality checks passed'
        }

        failure {
            echo 'Kuch fail hua - reports check karein'
        }
    }
}