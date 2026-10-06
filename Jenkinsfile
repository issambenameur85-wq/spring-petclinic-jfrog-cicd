pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Compile') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'jfrog-credentials',
                        usernameVariable: 'JFROG_USERNAME',
                        passwordVariable: 'JFROG_TOKEN'
                    )
                ]) {
                    sh './mvnw -s .mvn/jfrog-settings.xml compile'
                }
            }
        }

       stage('Test') {
           steps {
               withCredentials([
                   usernamePassword(
                       credentialsId: 'jfrog-credentials',
                       usernameVariable: 'JFROG_USERNAME',
                       passwordVariable: 'JFROG_TOKEN'
                   )
               ]) {
                   sh './mvnw -s .mvn/jfrog-settings.xml test'
               }
           }
       }

       stage('Package') {
           steps {
               withCredentials([
                   usernamePassword(
                       credentialsId: 'jfrog-credentials',
                       usernameVariable: 'JFROG_USERNAME',
                       passwordVariable: 'JFROG_TOKEN'
                   )
               ]) {
                   sh './mvnw -s .mvn/jfrog-settings.xml package -DskipTests'
               }
           }
       }

        stage('Docker Build') {
            steps {
                sh 'docker build -t spring-petclinic:assignment .'
            }
        }
    }
}
