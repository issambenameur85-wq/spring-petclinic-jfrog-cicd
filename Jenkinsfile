
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

                    env.LOCAL_IMAGE = "${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                    env.JFROG_IMAGE = "${env.JFROG_REGISTRY}/${env.JFROG_DOCKER_REPO}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                    env.JFROG_ASSIGNMENT_IMAGE = "${env.JFROG_REGISTRY}/${env.JFROG_DOCKER_REPO}/${env.IMAGE_NAME}:assignment"
                }

                echo "Building commit: ${env.IMAGE_TAG}"
            }
        }

        stage('Maven Build & Test') {
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
                        docker tag "$LOCAL_IMAGE" "$JFROG_ASSIGNMENT_IMAGE"

                        docker push "$JFROG_IMAGE"
                        docker push "$JFROG_ASSIGNMENT_IMAGE"

                        echo "Published: $JFROG_IMAGE"
                        echo "Published: $JFROG_ASSIGNMENT_IMAGE"
                    '''
                }
            }
        }
        
        stage('Export Docker Image') {
            steps {
                sh '''
                    set -eu

                    mkdir -p dist

                    docker save "$IMAGE_NAME:assignment" | \
                        gzip > dist/spring-petclinic-assignment.tar.gz

                    sha256sum dist/spring-petclinic-assignment.tar.gz \
                        > dist/spring-petclinic-assignment.tar.gz.sha256

                    cat > dist/RUN-INSTRUCTIONS.txt <<'EOF'
        Spring PetClinic - Docker Image
        ===============================

        Prerequisite:
        Docker must be installed and running.

        1. Extract and load the Docker image:

           gunzip -c spring-petclinic-assignment.tar.gz | docker load

        2. Run the application:

           docker run --rm -p 8083:8080 spring-petclinic:assignment

        3. Open the application:

           http://localhost:8083

        4. Stop the application:

           Press Ctrl+C in the terminal.

        Note:
        Port 8083 on the host maps to port 8080 inside the container.
        EOF
                '''

                archiveArtifacts(
                    artifacts: 'dist/*',
                    fingerprint: true,
                    onlyIfSuccessful: true
                )

                echo '''
                ===========================================
                SPRING PETCLINIC - ASSIGNMENT DELIVERABLE
                ===========================================

                Download from Jenkins Build > Artifacts:

                spring-petclinic-assignment.tar.gz
                spring-petclinic-assignment.tar.gz.sha256
                RUN-INSTRUCTIONS.txt

                LOAD:
                gunzip -c spring-petclinic-assignment.tar.gz | docker load

                RUN:
                docker run --rm -p 8083:8080 spring-petclinic:assignment

                URL:
                http://localhost:8083
                ===========================================
                '''
            }
        }

    }

    post {
        success {
            echo 'Pipeline completed successfully.'
            echo "Versioned image: ${env.JFROG_IMAGE}"
            echo "Assignment image: ${env.JFROG_ASSIGNMENT_IMAGE}"
        }

        failure {
            echo 'Pipeline failed. Review the Jenkins console output.'
        }

        cleanup {
            deleteDir()
        }
    }
}
