pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Compile') {
            steps {
                sh './mvnw compile'
            }
        }

        stage('Test') {
            steps {
                sh './mvnw test'
            }
        }
        stage('Package') {
            steps {
                sh './mvnw package -DskipTests'
            }
        }
    }
}
