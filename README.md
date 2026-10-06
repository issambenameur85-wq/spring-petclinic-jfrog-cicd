# Spring PetClinic – Jenkins, Docker & JFrog CI/CD

This repository contains a CI/CD implementation for the Spring PetClinic application using Jenkins, Docker, Maven, GitHub, and JFrog Artifactory.

The pipeline:

1. Checks out the source code from GitHub.
2. Compiles the application.
3. Runs the automated tests.
4. Packages the Spring Boot application.
5. Builds a runnable Docker image.

The project is based on the official Spring PetClinic application:

https://github.com/spring-projects/spring-petclinic

---

## CI/CD Architecture

```text
GitHub
   |
   v
Jenkins
   |
   +--> Checkout
   |
   +--> Compile
   |
   +--> Test
   |
   +--> Package
   |
   +--> Docker Build
   |
   v
Runnable Docker Image
```

JFrog Artifactory is used as the Maven repository manager so that Maven dependencies are resolved through Artifactory rather than directly from Maven Central.

The target dependency flow is:

```text
Jenkins
   |
   v
Maven
   |
   v
JFrog Cloud Artifactory
   |
   v
Maven Central
```

JFrog acts as the controlled repository layer between the build environment and the public Maven repository.

---

## Technologies

- Java 17
- Spring Boot
- Maven / Maven Wrapper
- Jenkins
- Docker
- JFrog Artifactory
- Git
- GitHub

---

## Repository Contents

The main files related to the CI/CD implementation are:

```text
.
├── Jenkinsfile
├── Dockerfile
├── jenkins/
│   └── Dockerfile
├── README.md
├── pom.xml
├── mvnw
├── mvnw.cmd
└── src/
```

The root `Dockerfile` builds the Spring PetClinic application image.

`jenkins/Dockerfile` defines the custom Jenkins environment used for this assignment.

---

# Jenkins Environment Setup

For this assignment, Jenkins runs locally inside a Docker container.

Running Jenkins in Docker keeps the CI environment isolated and avoids requiring a native Jenkins installation on the host machine.

A custom Jenkins image is used because the pipeline needs access to the Docker CLI to build the Spring PetClinic Docker image.

## Jenkins Docker Image

The Jenkins Dockerfile is located at:

```text
jenkins/Dockerfile
```

It contains:

```dockerfile
FROM jenkins/jenkins:lts-jdk21

USER root

RUN apt-get update \
    && apt-get install -y docker.io \
    && rm -rf /var/lib/apt/lists/*

USER jenkins
```

Build the custom Jenkins image:

```bash
docker build -t jenkins-petclinic:lts -f jenkins/Dockerfile .
```

## Create Persistent Jenkins Storage

A Docker volume is used to persist Jenkins configuration, plugins, jobs, and credentials between container restarts:

```bash
docker volume create jenkins_home
```

## Start Jenkins

Start Jenkins with:

```bash
docker run -d \
  --name jenkins \
  --restart unless-stopped \
  --user root \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins-petclinic:lts
```

Jenkins is then available at:

```text
http://localhost:8080
```

The `jenkins_home` volume keeps the Jenkins configuration persistent even if the Jenkins container is recreated.

---

## Docker Access from Jenkins

The Docker socket from the host is mounted inside the Jenkins container:

```text
/var/run/docker.sock
```

This allows the Docker CLI running inside Jenkins to communicate with the host Docker engine.

The architecture is:

```text
Jenkins Container
       |
       | Docker CLI
       v
/var/run/docker.sock
       |
       v
Host Docker Engine
       |
       v
Spring PetClinic Docker Image
```

This enables the Jenkins pipeline to execute commands such as:

```bash
docker build -t spring-petclinic:assignment .
```

---

# Configure Jenkins with GitHub

The Jenkins job uses **Pipeline script from SCM**.

This allows Jenkins to retrieve both the application source code and the pipeline definition directly from GitHub.

Create a new Jenkins **Pipeline** job and configure it with:

```text
Job name:    spring-petclinic-cicd
Definition:  Pipeline script from SCM
SCM:         Git
Repository:  https://github.com/issambenameur85-wq/spring-petclinic-jfrog-cicd.git
Credentials: None
Branch:      */main
Script Path: Jenkinsfile
```

The GitHub repository is public, so Jenkins does not require GitHub credentials for read-only checkout.

When the Jenkins job starts:

```text
GitHub Repository
       |
       | HTTPS Git checkout
       v
Jenkins
       |
       | loads
       v
Jenkinsfile
       |
       +--> Checkout
       +--> Compile
       +--> Test
       +--> Package
       +--> Docker Build
```

---

## Jenkins Checkout Configuration

The pipeline contains an explicit source-code checkout stage:

```groovy
stage('Checkout') {
    steps {
        checkout scm
    }
}
```

Jenkins Declarative Pipeline normally performs an automatic SCM checkout.

Because this pipeline contains an explicit `Checkout` stage for visibility, the default checkout is disabled:

```groovy
options {
    skipDefaultCheckout(true)
}
```

This prevents Jenkins from cloning the repository twice during the same pipeline execution.

---

## GitHub Authentication

There are two different GitHub access paths in this environment.

### Development Machine

SSH authentication is used from the development machine when pushing changes to GitHub:

```text
Developer Machine
       |
       | SSH
       v
GitHub
```

The Git remote used for development is:

```text
git@github.com:issambenameur85-wq/spring-petclinic-jfrog-cicd.git
```

### Jenkins

Jenkins only requires read access to the public repository.

It therefore performs checkout over HTTPS:

```text
Jenkins
   |
   | HTTPS / read-only
   v
GitHub
```

Repository:

```text
https://github.com/issambenameur85-wq/spring-petclinic-jfrog-cicd.git
```

No GitHub credentials are required by Jenkins for this public repository.

This keeps developer authentication separate from Jenkins source-code checkout.

---

# Jenkins Pipeline

The `Jenkinsfile` defines the CI pipeline.

The pipeline contains the following stages:

```text
Checkout
   |
   v
Compile
   |
   v
Test
   |
   v
Package
   |
   v
Docker Build
```

## 1. Checkout

Retrieves the source code from GitHub:

```groovy
stage('Checkout') {
    steps {
        checkout scm
    }
}
```

## 2. Compile

Compiles the application using the Maven Wrapper:

```bash
./mvnw compile
```

Using the Maven Wrapper ensures the expected Maven version can be used without requiring Maven to be installed manually on the Jenkins environment.

## 3. Test

Runs the automated test suite:

```bash
./mvnw test
```

The test stage is kept separate so that test results and failures are clearly visible in the Jenkins pipeline.

## 4. Package

Packages the Spring Boot application as an executable JAR:

```bash
./mvnw package -DskipTests
```

Tests are skipped during this stage because they have already been executed in the dedicated Test stage.

The generated Spring Boot application artifact is:

```text
target/spring-petclinic-4.0.0-SNAPSHOT.jar
```

## 5. Docker Build

The final pipeline stage builds the runnable Docker image:

```bash
docker build -t spring-petclinic:assignment .
```

The resulting image is:

```text
spring-petclinic:assignment
```

---

# Application Docker Image

The application `Dockerfile` uses Eclipse Temurin Java 17 as the runtime environment.

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY target/spring-petclinic-4.0.0-SNAPSHOT.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

The Spring Boot application listens on port `8080` inside the container.

---

# Run the Docker Image

If the image has already been built locally:

```bash
docker run --rm -p 8081:8080 spring-petclinic:assignment
```

The application is available at:

```text
http://localhost:8081
```

Port `8081` is used on the host because Jenkins is already running on port `8080`.

The mapping is:

```text
Host                  Container
8081  --------------> 8080
                       Spring PetClinic
```

Open the application in a browser:

```text
http://localhost:8081
```

Or verify it from the command line:

```bash
curl -I http://localhost:8081
```

A successful application startup should return:

```text
HTTP/1.1 200
```

---

# Load and Run the Submitted Docker Image

A runnable Docker image is provided separately from the GitHub repository as:

```text
spring-petclinic-assignment.tar.gz
```

The image archive is intentionally not stored in Git because it is a generated binary artifact.

## Load the Image

Load the submitted image into Docker:

```bash
gunzip -c spring-petclinic-assignment.tar.gz | docker load
```

Verify that the image was loaded:

```bash
docker images spring-petclinic
```

The image should appear as:

```text
spring-petclinic   assignment
```

## Run the Image

Run the submitted image:

```bash
docker run --rm -p 8081:8080 spring-petclinic:assignment
```

Open:

```text
http://localhost:8081
```

or verify it with:

```bash
curl -I http://localhost:8081
```

---

# Build and Run Manually

Clone the repository:

```bash
git clone https://github.com/issambenameur85-wq/spring-petclinic-jfrog-cicd.git
cd spring-petclinic-jfrog-cicd
```

## Compile

```bash
./mvnw compile
```

## Run Tests

```bash
./mvnw test
```

## Package

```bash
./mvnw package -DskipTests
```

The executable JAR will be generated under:

```text
target/spring-petclinic-4.0.0-SNAPSHOT.jar
```

## Build Docker Image

```bash
docker build -t spring-petclinic:assignment .
```

## Run

```bash
docker run --rm -p 8081:8080 spring-petclinic:assignment
```

Verify:

```bash
curl -I http://localhost:8081
```

---

# JFrog Artifactory

JFrog Artifactory is used as the repository manager for Maven dependencies.

The required dependency flow is:

```text
Jenkins
   |
   v
Maven
   |
   v
Artifactory Virtual Maven Repository
   |
   v
Artifactory Remote Maven Repository
   |
   v
Maven Central
```

## Repository Architecture

The **remote Maven repository** acts as a proxy for Maven Central.

When a dependency is requested for the first time:

```text
Maven
   |
   v
JFrog Artifactory
   |
   | Cache Miss
   v
Maven Central
   |
   v
JFrog Cache
   |
   v
Maven
```

Artifactory downloads the dependency from Maven Central and stores it in its cache.

For subsequent requests:

```text
Maven
   |
   v
JFrog Artifactory Cache
   |
   v
Dependency
```

This reduces direct dependency on external repositories and provides a centralized location for dependency management.

The **virtual Maven repository** provides Maven with a single Artifactory endpoint.

Maven is configured to use this virtual repository as its repository mirror so that dependency resolution is routed through JFrog rather than directly to Maven Central.

> **Status:** JFrog Cloud configuration and dependency-resolution verification are still being completed and will be finalized before submission.

---

# JFrog Credentials

JFrog credentials and access tokens must not be committed to GitHub.

Credentials required by the pipeline are stored using Jenkins Credentials and injected into the build environment only when required.

The repository must never contain:

```text
JFrog passwords
JFrog access tokens
Jenkins secrets
GitHub private keys
```

---

# Security Considerations

This assignment uses a local Jenkins environment designed to keep the setup simple and reproducible.

For the local environment, Jenkins runs as `root` and has access to:

```text
/var/run/docker.sock
```

Mounting the Docker socket provides Jenkins with significant access to the host Docker daemon.

This approach was intentionally used for the local assignment environment so Jenkins can build Docker images without introducing additional infrastructure.

For a production environment, I would instead use dedicated Jenkins build agents with controlled permissions and an appropriately secured container build strategy rather than exposing the host Docker socket directly to the Jenkins controller.

Secrets such as JFrog access tokens are also kept outside source control and managed using Jenkins Credentials.

---

# Verification

The complete application build and Docker execution flow has been validated:

```text
Source Code
    |
    v
Jenkins Checkout
    |
    v
Compile
    |
    v
Automated Tests
    |
    v
Spring Boot JAR
    |
    v
Docker Build
    |
    v
Docker Image
    |
    v
Docker Container
    |
    v
HTTP 200
```

The generated Docker image was started locally using:

```bash
docker run --rm -p 8081:8080 spring-petclinic:assignment
```

The running Spring PetClinic application successfully responded with:

```text
HTTP/1.1 200
```

This verifies that the Docker image produced by the CI process is runnable.

---

# Bonus – Self-Hosted JFrog Artifactory

As an additional part of the assignment, a self-hosted JFrog Artifactory environment will be used to demonstrate dependency resolution through a locally deployed Artifactory instance.

The intended bonus architecture is:

```text
GitHub
   |
   v
Jenkins
   |
   v
Maven
   |
   v
Self-Hosted JFrog Artifactory
   |
   v
Maven Central
```

The self-hosted configuration is kept separate from the primary JFrog Cloud implementation so that the bonus environment does not affect the required Cloud-based pipeline.

> **Status:** Bonus self-hosted Artifactory setup will be completed after the required JFrog Cloud integration.

---

# Author

Issam Ben Ameur
