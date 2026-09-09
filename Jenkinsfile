pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                echo 'Code SCM se checkout ho gaya'
                sh 'ls -la'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // 'sonar-server' = System config waala naam
                withSonarQubeEnv('sonar-server') {
                    script {
                        // 'sonar-scanner' = Tools waala naam
                        def scannerHome = tool 'sonar-scanner'
                        // properties file root mein hai, isliye sirf command chalegi
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                // 2 min tak wait; agar gate FAIL to pipeline abort
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline passed - code quality OK'
        }
        failure {
            echo 'Pipeline failed - Sonar ya Quality Gate check kare'
        }
    }
}