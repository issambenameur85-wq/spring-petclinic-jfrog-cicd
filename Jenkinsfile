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

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'jfrog-credentials',
                        usernameVariable: 'JFROG_USERNAME',
                        passwordVariable: 'JFROG_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$JFROG_TOKEN" | docker login petclinictestcicd.jfrog.io \
                            -u "$JFROG_USERNAME" \
                            --password-stdin

                        docker tag spring-petclinic:assignment \
                            petclinictestcicd.jfrog.io/docker-local/spring-petclinic:assignment

                        docker push \
                            petclinictestcicd.jfrog.io/docker-local/spring-petclinic:assignment
                    '''
                }
            }
        }
    }
}
