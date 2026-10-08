
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    environment {
        JFROG_REGISTRY = 'petclinictestcicd.jfrog.io'
        JFROG_DOCKER_REPO = 'docker-local'
        IMAGE_NAME = 'spring-petclinic'
        MAVEN_SETTINGS = '.mvn/jfrog-settings.xml'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    env.LOCAL_IMAGE = "${IMAGE_NAME}:${IMAGE_TAG}"
                    env.JFROG_IMAGE = "${JFROG_REGISTRY}/${JFROG_DOCKER_REPO}/${IMAGE_NAME}:${IMAGE_TAG}"
                }

                echo "Building commit: ${env.IMAGE_TAG}"
            }
        }

        stage('Maven Build & Test') {
            environment {
                JFROG_USERNAME = credentials('jfrog-credentials_USR')
            }

            stages {
                stage('Compile') {
                    steps {
                        withCredentials([
                            usernamePassword(
                                credentialsId: 'jfrog-credentials',
                                usernameVariable: 'JFROG_USERNAME',
                                passwordVariable: 'JFROG_TOKEN'
                            )
                        ]) {
                            sh './mvnw -B -ntp -s "$MAVEN_SETTINGS" compile'
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
                            sh './mvnw -B -ntp -s "$MAVEN_SETTINGS" test'
                        }
                    }

                    post {
                        always {
                            junit(
                                testResults: 'target/surefire-reports/*.xml',
                                allowEmptyResults: true
                            )
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
                            sh './mvnw -B -ntp -s "$MAVEN_SETTINGS" package -DskipTests'
                        }
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -eu

                    docker build \
                        --tag "$LOCAL_IMAGE" \
                        --tag "$IMAGE_NAME:assignment" \
                        .
                '''
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
                        set -eu

                        export DOCKER_CONFIG="$(mktemp -d)"
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT

                        printf '%s' "$JFROG_TOKEN" |
                            docker login "$JFROG_REGISTRY" \
                                --username "$JFROG_USERNAME" \
                                --password-stdin

                        docker tag "$LOCAL_IMAGE" "$JFROG_IMAGE"
                        docker push "$JFROG_IMAGE"

                        echo "Published image: $JFROG_IMAGE"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
            echo "Published image: ${env.JFROG_IMAGE}"
            echo "Assignment image: ${env.IMAGE_NAME}:assignment"
        }

        failure {
            echo 'Pipeline failed. Review the stage logs.'
        }

        cleanup {
            deleteDir()
        }
    }
}
