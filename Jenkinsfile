pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Code SCM se automatically checkout ho gaya'
                sh 'ls -la'
            }
        }

        stage('Verify Structure') {
            steps {
                sh 'ls -la backend frontend'
            }
        }
    }

    post {
        success { echo 'Starter pipeline successful!' }
        failure { echo 'Kuch galat hua - logs check karein' }
    }
}