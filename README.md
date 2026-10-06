# Spring PetClinic - Jenkins, Docker & JFrog CI/CD

This repository contains a CI/CD implementation for the Spring PetClinic application using Jenkins, Maven, Docker, GitHub, and JFrog Artifactory.

The primary pipeline:

1. Checks out the source code from GitHub.
2. Compiles the application.
3. Runs the automated tests.
4. Packages the Spring Boot application.
5. Builds a runnable Docker image.
6. Publishes the Docker image to JFrog Cloud Artifactory.

All Maven dependency resolution performed by the CI pipeline is routed through JFrog Cloud Artifactory rather than directly to Maven Central.

An additional self-hosted Artifactory implementation is included as a bonus. It demonstrates the same dependency resolution and Docker image publishing workflow using a locally deployed JFrog Artifactory OSS instance.

The project is based on the official Spring PetClinic application:

https://github.com/spring-projects/spring-petclinic

---

# CI/CD Architecture

The primary CI pipeline uses JFrog Cloud for both Maven dependency resolution and Docker image storage.

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
   +--> Docker Push
   |
   v
JFrog Cloud Artifactory
   |
   v
docker-local
   |
   v
spring-petclinic:assignment
```

Maven dependency resolution follows a separate inbound path:

```text
Jenkins
   |
   v
Maven Wrapper
   |
   v
JFrog Cloud - maven-virtual
   |
   v
JFrog - maven-central-remote
   |
   v
Maven Central
```

JFrog Artifactory therefore performs two roles:

```text
                    JFrog Artifactory
                    /              \
                   /                \
                  v                  ^
          Maven Dependencies     Docker Image
                  |                  |
                  v                  |
               Jenkins --------------+
```

JFrog acts as the controlled repository layer between the Maven build and public Maven repositories while also storing Docker images produced by the CI pipeline.

---

# Technologies

- Java 17
- Spring Boot
- Maven / Maven Wrapper
- Jenkins
- Docker
- JFrog Artifactory
- PostgreSQL
- Git
- GitHub

---

# Repository Contents

The main files related to the CI/CD implementation are:

```text
.
├── .mvn/
│   ├── jfrog-settings.xml
│   └── jfrog-selfhosted-settings.xml
├── Jenkinsfile
├── Jenkinsfile.selfhosted
├── Dockerfile
├── jenkins/
│   └── Dockerfile
├── README.md
├── pom.xml
├── mvnw
├── mvnw.cmd
└── src/
```

`Jenkinsfile` defines the required JFrog Cloud CI pipeline.

`Jenkinsfile.selfhosted` defines the optional self-hosted Artifactory bonus pipeline.

The root `Dockerfile` defines the Spring PetClinic runtime image.

`.mvn/jfrog-settings.xml` configures Maven dependency resolution through JFrog Cloud Artifactory.

`.mvn/jfrog-selfhosted-settings.xml` configures Maven dependency resolution through the separate self-hosted Artifactory environment.

`jenkins/Dockerfile` defines the custom Jenkins environment used for this assignment.

---

# Prerequisites

The following tools are required to reproduce the complete local CI environment:

- Git
- Docker
- Docker Compose
- Java 17 for optional local Maven execution
- A JFrog Cloud account
- A JFrog access token

Maven does not need to be installed separately because the project uses the Maven Wrapper.

---

# Jenkins Environment Setup

For this assignment, Jenkins runs locally inside a Docker container.

Running Jenkins in Docker keeps the CI environment isolated and avoids requiring a native Jenkins installation on the host machine.

A custom Jenkins image is used because the pipeline needs access to the Docker CLI to build and publish the Spring PetClinic Docker image.

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

The volume is mounted at Jenkins' data directory:

```text
Docker Volume                  Jenkins Container

jenkins_home  ---------------> /var/jenkins_home
```

This allows Jenkins configuration to survive container recreation.

## Start Jenkins

For this local assignment environment, Jenkins is started with access to the host Docker socket:

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

---

# Docker Access from Jenkins

The Docker socket from the host is mounted inside the Jenkins container:

```text
/var/run/docker.sock
```

This allows the Docker CLI running inside Jenkins to communicate with the host Docker engine.

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

This enables the pipeline to execute Docker commands such as:

```bash
docker build -t spring-petclinic:assignment .
```

and:

```bash
docker push <registry>/docker-local/spring-petclinic:assignment
```

This configuration is intentionally limited to the local assignment environment. See the Security Considerations section for production considerations.

---

# Configure Jenkins with GitHub

The Jenkins job uses **Pipeline script from SCM**.

This allows Jenkins to retrieve both the application source code and the pipeline definition directly from GitHub.

Create a Jenkins Pipeline job with:

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

```text
GitHub Repository
       |
       | HTTPS checkout
       v
Jenkins
       |
       | loads
       v
Jenkinsfile
```

---

# Jenkins Checkout Configuration

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

# GitHub Authentication

There are two different GitHub access paths in this environment.

## Development Machine

SSH authentication is used from the development machine when pushing changes:

```text
Developer Machine
       |
       | SSH
       v
GitHub
```

Development remote:

```text
git@github.com:issambenameur85-wq/spring-petclinic-jfrog-cicd.git
```

## Jenkins

Jenkins only requires read access to the public repository.

It performs checkout over HTTPS:

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

The `Jenkinsfile` defines the required JFrog Cloud CI pipeline.

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
   |
   v
Docker Push
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

Compiles the application while resolving Maven dependencies through JFrog Cloud Artifactory:

```bash
./mvnw -s .mvn/jfrog-settings.xml compile
```

Using the Maven Wrapper ensures the expected Maven version can be used without requiring Maven to be installed manually in the Jenkins environment.

## 3. Test

Runs the automated test suite using the same JFrog Maven configuration:

```bash
./mvnw -s .mvn/jfrog-settings.xml test
```

The test stage is kept separate so failures are clearly visible in Jenkins.

A validated pipeline execution produced:

```text
Tests run: 79
Failures: 0
Errors: 0
Skipped: 2

BUILD SUCCESS
```

## 4. Package

Packages the Spring Boot application as an executable JAR:

```bash
./mvnw -s .mvn/jfrog-settings.xml package -DskipTests
```

Tests are skipped during this invocation because they have already been executed in the dedicated Test stage.

The generated artifact is:

```text
target/spring-petclinic-4.0.0-SNAPSHOT.jar
```

## 5. Docker Build

The Docker Build stage creates the runnable application image:

```bash
docker build -t spring-petclinic:assignment .
```

The resulting local image is:

```text
spring-petclinic:assignment
```

At this point, the image exists on the Docker engine used by Jenkins but has not yet been published to Artifactory.

## 6. Docker Push

After successfully building the image, Jenkins publishes it to the `docker-local` repository in JFrog Cloud Artifactory.

The publishing flow is:

```text
Jenkins
   |
   v
Docker Build
   |
   v
spring-petclinic:assignment
   |
   v
Docker Login
   |
   v
Docker Tag
   |
   v
Docker Push
   |
   v
JFrog Cloud Artifactory
   |
   v
docker-local
   |
   v
spring-petclinic:assignment
```

### Docker Authentication

The existing Jenkins credential:

```text
jfrog-credentials
```

provides:

```text
JFROG_USERNAME
JFROG_TOKEN
```

The pipeline authenticates the Docker client without hardcoding the JFrog access token in source control.

Conceptually:

```bash
echo "$JFROG_TOKEN" | docker login <JFROG_REGISTRY> \
    -u "$JFROG_USERNAME" \
    --password-stdin
```

### Docker Image Tagging

The locally built image:

```text
spring-petclinic:assignment
```

is tagged with its JFrog registry destination:

```text
petclinictestcicd.jfrog.io/docker-local/spring-petclinic:assignment
```

Conceptually:

```text
spring-petclinic:assignment
           |
           | docker tag
           v
petclinictestcicd.jfrog.io
           |
           v
docker-local
           |
           v
spring-petclinic:assignment
```

Tagging does not upload or rebuild the image. It creates a registry-qualified reference for the existing Docker image.

### Docker Image Push

The tagged image is published with:

```bash
docker push \
  petclinictestcicd.jfrog.io/docker-local/spring-petclinic:assignment
```

After the push completes, the image is stored in the JFrog Cloud `docker-local` repository.

---

# JFrog Cloud Artifactory

JFrog Cloud Artifactory serves two roles in this implementation:

1. Maven repository manager for application dependencies.
2. Docker registry for application images produced by Jenkins.

```text
                     JFrog Cloud Artifactory
                    /                      \
                   /                        \
                  v                          v
           Maven Repositories          Docker Repository
                  |                          |
          maven-virtual                 docker-local
                  |
                  v
       maven-central-remote
                  |
                  v
            Maven Central
```

---

# Maven Repository Architecture

Two Maven repositories are used in JFrog Cloud.

## Remote Repository

```text
maven-central-remote
```

This remote Maven repository proxies Maven Central:

```text
https://repo1.maven.org/maven2/
```

When an artifact is not already available in Artifactory's remote cache, Artifactory retrieves the artifact from Maven Central.

Conceptually:

```text
Maven
   |
   v
JFrog Artifactory
   |
   | cache miss
   v
Maven Central
   |
   v
JFrog Remote Cache
   |
   v
Maven
```

Future requests for cached artifacts can be served by Artifactory without downloading the same artifact again.

## Virtual Repository

```text
maven-virtual
```

The virtual repository contains:

```text
maven-central-remote
```

Maven communicates with the virtual repository rather than directly with the remote repository:

```text
Maven
   |
   v
maven-virtual
   |
   v
maven-central-remote
```

This provides a single repository endpoint to the build and allows additional repositories to be included later without changing Maven's repository endpoint.

---

# Docker Repository Architecture

A local Docker repository is used in JFrog Cloud:

```text
docker-local
```

Unlike `maven-central-remote`, which proxies an external Maven repository, `docker-local` stores Docker images produced internally by the CI pipeline.

The two repository flows therefore operate in opposite directions.

## Maven

```text
Maven Central
      |
      v
maven-central-remote
      |
      v
maven-virtual
      |
      v
Jenkins
```

## Docker

```text
Jenkins
      |
      v
Docker Build
      |
      v
Docker Push
      |
      v
docker-local
      |
      v
JFrog Artifactory
```

---

# Maven JFrog Configuration

The Maven configuration used by the CI pipeline is stored in:

```text
.mvn/jfrog-settings.xml
```

The JFrog Cloud virtual repository endpoint is:

```text
https://petclinictestcicd.jfrog.io/artifactory/maven-virtual/
```

Maven uses a mirror similar to:

```xml
<mirror>
    <id>jfrog</id>
    <name>JFrog Cloud Maven Virtual Repository</name>
    <url>https://petclinictestcicd.jfrog.io/artifactory/maven-virtual/</url>
    <mirrorOf>*</mirrorOf>
</mirror>
```

The important configuration is:

```xml
<mirrorOf>*</mirrorOf>
```

This instructs Maven to use the JFrog mirror for matching Maven repository requests instead of accessing repositories such as Maven Central directly.

The corresponding Maven server configuration uses environment variables:

```xml
<server>
    <id>jfrog</id>
    <username>${env.JFROG_USERNAME}</username>
    <password>${env.JFROG_TOKEN}</password>
</server>
```

No JFrog username, password, or access token is committed to the repository.

---

# Jenkins JFrog Credentials

JFrog authentication is managed through Jenkins Credentials.

The Jenkins credential ID used by the Cloud pipeline is:

```text
jfrog-credentials
```

The credential contains:

```text
Username: JFrog username
Password: JFrog access token
```

The pipeline injects the credentials only when Artifactory authentication is required.

For Maven:

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'jfrog-credentials',
        usernameVariable: 'JFROG_USERNAME',
        passwordVariable: 'JFROG_TOKEN'
    )
]) {
    sh './mvnw -s .mvn/jfrog-settings.xml test'
}
```

The same Jenkins-managed credentials can be used for authenticated Docker operations when the JFrog identity has the required repository permissions.

The repository must never contain:

```text
JFrog passwords
JFrog access tokens
Jenkins secrets
GitHub private keys
```

---

# Verifying JFrog Maven Dependency Resolution

A successful Maven build alone does not prove that dependencies are being resolved through Artifactory.

The Jenkins console output was therefore inspected to verify the repository being used.

The pipeline produced entries such as:

```text
Downloading from jfrog:
https://petclinictestcicd.jfrog.io/artifactory/maven-virtual/...

Downloaded from jfrog:
https://petclinictestcicd.jfrog.io/artifactory/maven-virtual/...
```

This verifies that Maven uses the configured JFrog virtual repository.

The effective dependency flow is:

```text
Jenkins
   |
   v
Maven
   |
   v
JFrog Cloud
   |
   v
maven-virtual
   |
   v
maven-central-remote
   |
   v
Maven Central
```

There are two relevant cache layers:

```text
Jenkins
   |
   v
Maven Local Repository (~/.m2)
   |
   | local cache miss
   v
JFrog Artifactory
   |
   | Artifactory cache miss
   v
Maven Central
```

If an artifact is already present in Maven's local repository, Maven may not need to contact Artifactory for that artifact.

If Maven needs an artifact that is not locally available, the configured JFrog mirror is used.

Artifactory can then either serve the artifact from its remote cache or retrieve it from Maven Central.

---

# Verifying JFrog Docker Publishing

Docker publishing was validated manually before being automated through Jenkins.

The image was first built locally:

```text
spring-petclinic:assignment
```

It was then tagged for JFrog:

```text
petclinictestcicd.jfrog.io/docker-local/spring-petclinic:assignment
```

The image was successfully pushed to JFrog Cloud.

The successful push returned the following image digest:

```text
sha256:339cfda88c079c845c542d9c713348fd664d1ac5a2918b04a3dc684eafb38e4c
```

A subsequent Docker pull returned the same digest:

```text
sha256:339cfda88c079c845c542d9c713348fd664d1ac5a2918b04a3dc684eafb38e4c
```

The matching digest verifies that the pushed and subsequently referenced image manifest is the same.

The tested flow was:

```text
Docker Build
     |
     v
Docker Tag
     |
     v
Docker Push
     |
     v
JFrog Cloud docker-local
     |
     v
Docker Pull
     |
     v
spring-petclinic:assignment
```

---

# Build Manually Using JFrog

Clone the repository:

```bash
git clone https://github.com/issambenameur85-wq/spring-petclinic-jfrog-cicd.git
cd spring-petclinic-jfrog-cicd
```

Provide the required JFrog credentials through environment variables:

```bash
export JFROG_USERNAME="<your-jfrog-username>"
export JFROG_TOKEN="<your-jfrog-access-token>"
```

Do not store these values in source control.

## Compile

```bash
./mvnw -s .mvn/jfrog-settings.xml compile
```

## Run Tests

```bash
./mvnw -s .mvn/jfrog-settings.xml test
```

## Package

```bash
./mvnw -s .mvn/jfrog-settings.xml package -DskipTests
```

The executable JAR is generated under:

```text
target/spring-petclinic-4.0.0-SNAPSHOT.jar
```

## Build Docker Image

```bash
docker build -t spring-petclinic:assignment .
```

---

# Application Docker Image

The application `Dockerfile` uses Eclipse Temurin Java 17 as the runtime environment:

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY target/spring-petclinic-4.0.0-SNAPSHOT.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

The image contains the executable Spring Boot JAR produced by the Maven Package stage.

The application listens on port `8080` inside the container.

---

# Run the Docker Image

If the image has already been built:

```bash
docker run --rm -p 8083:8080 spring-petclinic:assignment
```

The application is available at:

```text
http://localhost:8083
```

Port `8083` is used on the host because Jenkins uses port `8080`, while the local self-hosted Artifactory environment uses ports `8081` and `8082`.

```text
Host                  Container

8083  --------------> 8080
                       Spring PetClinic
```

Verify it:

```bash
curl -I http://localhost:8083
```

A successful application startup should return:

```text
HTTP/1.1 200
```

---

# Runnable Docker Image Deliverable

A runnable Docker image is provided separately from the GitHub repository as:

```text
spring-petclinic-assignment.tar.gz
```

The image archive is intentionally not stored in Git because it is a generated binary artifact.

The submitted archive has the following SHA-256 checksum:

```text
401f2c40fc07f48c647c9f07c375de178b0bdee85c12c4339b57dd66a2c18731
```

## Export the Image

The image can be exported with:

```bash
docker save spring-petclinic:assignment \
  | gzip > spring-petclinic-assignment.tar.gz
```

## Load the Submitted Image

Load the submitted image into Docker:

```bash
gunzip -c spring-petclinic-assignment.tar.gz | docker load
```

Verify:

```bash
docker images spring-petclinic
```

The image should appear with:

```text
REPOSITORY          TAG
spring-petclinic    assignment
```

## Run the Submitted Image

```bash
docker run --rm -p 8083:8080 spring-petclinic:assignment
```

Open:

```text
http://localhost:8083
```

or verify from the command line:

```bash
curl -I http://localhost:8083
```

---

# Cloud Pipeline Verification

The required Cloud CI/CD flow has been validated across the build stages and JFrog integrations:

```text
GitHub
   |
   v
Jenkins Checkout
   |
   v
Compile
   |
   | dependencies through JFrog
   v
Test
   |
   | 79 tests / 0 failures
   v
Package
   |
   v
Spring Boot JAR
   |
   v
Docker Build
   |
   v
spring-petclinic:assignment
   |
   v
Docker Push
   |
   v
JFrog Cloud docker-local
```

Maven logs verified dependency resolution through:

```text
https://petclinictestcicd.jfrog.io/artifactory/maven-virtual/
```

The Docker registry integration was independently validated with a successful push and pull of:

```text
petclinictestcicd.jfrog.io/docker-local/spring-petclinic:assignment
```

The application Docker image was successfully started and returned:

```text
HTTP/1.1 200
```

---

# Security Considerations

This assignment uses a local Jenkins environment designed to keep the setup simple and reproducible.

## Secrets

Secrets are not committed to Git.

JFrog credentials are stored using Jenkins Credentials and injected into Maven and Docker operations only when required.

The access token is masked by Jenkins in console output.

## Docker Socket

For this local environment, Jenkins runs as `root` and has access to:

```text
/var/run/docker.sock
```

Mounting the Docker socket gives the Jenkins container significant control over the host Docker daemon.

This approach was intentionally chosen for the local assignment environment so Jenkins can build and publish Docker images without introducing additional infrastructure.

For a production environment, dedicated Jenkins build agents with controlled permissions and an appropriately secured container build strategy should be used instead of exposing the host Docker socket directly to the Jenkins controller.

## GitHub

Jenkins uses anonymous HTTPS read access because the GitHub repository is public.

Developer write access is handled separately using SSH.

---

# Bonus - Self-Hosted JFrog Artifactory

The optional self-hosted Artifactory bonus demonstrates the CI workflow using a locally deployed Artifactory instance instead of the JFrog Cloud service.

JFrog Artifactory OSS 7.161.15 runs locally with PostgreSQL.

The self-hosted implementation is intentionally isolated from the required JFrog Cloud pipeline so the primary solution remains independently functional.

## Self-Hosted Environment

The local environment contains:

```text
Docker Host
│
├── Jenkins
│   ├── 8080
│   └── 50000
│
├── JFrog Artifactory OSS
│   ├── 8081
│   └── 8082
│
└── PostgreSQL
    └── 5432
```

Artifactory runtime data is persisted outside the container.

In the local development environment the Artifactory data directory is bind-mounted into:

```text
/var/opt/jfrog/artifactory
```

inside the Artifactory container.

## Bonus Architecture

The self-hosted environment uses Artifactory for both Maven dependencies and Docker images:

```text
                         GitHub
                            |
                            v
                         Jenkins
                            |
                  +---------+---------+
                  |                   |
                  v                   v
                Maven            Docker Build
                  |                   |
                  v                   v
           maven-virtual     spring-petclinic:selfhosted
                  |                   |
                  v                   | docker push
      maven-central-remote            v
                  |              docker-local
                  v                   |
            Maven Central             v
                              Self-Hosted Artifactory
```

---

# Self-Hosted Maven Repository Configuration

The self-hosted Artifactory contains the Maven repositories:

```text
maven-central-remote
maven-virtual
```

`maven-central-remote` proxies Maven Central.

`maven-virtual` provides Maven with a single repository endpoint and contains the remote repository.

The dependency flow is:

```text
Jenkins
   |
   v
Maven Wrapper
   |
   v
Self-Hosted Artifactory
   |
   v
maven-virtual
   |
   v
maven-central-remote
   |
   v
Maven Central
```

The self-hosted Maven configuration is stored separately:

```text
.mvn/jfrog-selfhosted-settings.xml
```

Its configuration uses runtime environment variables:

```xml
<server>
    <id>jfrog-selfhosted</id>
    <username>${env.ARTIFACTORY_USERNAME}</username>
    <password>${env.ARTIFACTORY_PASSWORD}</password>
</server>
```

and:

```xml
<mirror>
    <id>jfrog-selfhosted</id>
    <name>Self-hosted JFrog Artifactory</name>
    <url>${env.ARTIFACTORY_URL}/artifactory/maven-virtual/</url>
    <mirrorOf>*</mirrorOf>
</mirror>
```

Credentials and environment-specific addresses are therefore kept outside source control.

---

# Self-Hosted Jenkins Pipeline

The bonus uses a separate pipeline definition:

```text
Jenkinsfile.selfhosted
```

The Jenkins job can be configured with:

```text
Job name:    spring-petclinic-selfhosted
Definition:  Pipeline script from SCM
SCM:         Git
Repository:  https://github.com/issambenameur85-wq/spring-petclinic-jfrog-cicd.git
Credentials: None
Branch:      */main
Script Path: Jenkinsfile.selfhosted
```

The self-hosted pipeline follows:

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
   |
   v
Docker Push
```

The resulting Docker image is tagged:

```text
spring-petclinic:selfhosted
```

---

# Docker Image Publishing to Self-Hosted Artifactory

The self-hosted implementation also uses a local Docker repository:

```text
docker-local
```

The image publishing flow is:

```text
Jenkins
   |
   v
Docker Build
   |
   v
spring-petclinic:selfhosted
   |
   v
Docker Login
   |
   v
Docker Tag
   |
   v
Docker Push
   |
   v
Self-Hosted JFrog Artifactory
   |
   v
docker-local
   |
   v
spring-petclinic:selfhosted
```

## Build

Jenkins builds the image:

```bash
docker build -t spring-petclinic:selfhosted .
```

## Authentication

Jenkins retrieves the Artifactory username and password from Jenkins Credentials.

```text
jfrog-selfhosted-credentials
     |
     +--> ARTIFACTORY_USERNAME
     |
     +--> ARTIFACTORY_PASSWORD
```

No Artifactory password or access token is stored in the repository.

## Tag

The image is tagged with the self-hosted Artifactory Docker destination:

```text
spring-petclinic:selfhosted
             |
             | docker tag
             v
Self-Hosted Artifactory
             |
             v
docker-local/spring-petclinic:selfhosted
```

## Push

The Docker client then publishes the image:

```text
Docker Engine
      |
      | docker push
      v
Self-Hosted Artifactory
      |
      v
docker-local
      |
      v
spring-petclinic:selfhosted
```

The self-hosted Artifactory therefore performs two separate roles:

```text
              SELF-HOSTED ARTIFACTORY
                 /            \
                /              \
               v                ^
         Maven Dependencies   Docker Images
               |                |
               v                |
             Jenkins -----------+
```

---

# Jenkins Credentials for Self-Hosted Artifactory

The self-hosted pipeline uses Jenkins-managed configuration:

```text
jfrog-selfhosted-credentials
    -> ARTIFACTORY_USERNAME
    -> ARTIFACTORY_PASSWORD

jfrog-selfhosted-url
    -> ARTIFACTORY_URL
```

The Artifactory endpoint is injected at runtime instead of being hard-coded in `Jenkinsfile.selfhosted`.

For the local Docker Desktop environment, Jenkins reaches services exposed by the host through:

```text
host.docker.internal
```

This is required because Jenkins runs inside a Docker container. `localhost` inside the Jenkins container refers to the Jenkins container itself, not services exposed by the host.

---

# Self-Hosted Verification

Connectivity from the Jenkins container to Artifactory is verified before relying on the complete pipeline.

Maven output can be used to confirm dependency resolution through the self-hosted virtual repository with entries such as:

```text
Downloading from jfrog-selfhosted:
<self-hosted-artifactory>/artifactory/maven-virtual/...
```

The complete target pipeline is:

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
   |
   v
Docker Push
   |
   v
Self-Hosted Artifactory
```

After a successful Docker Push stage, the resulting image is stored under:

```text
docker-local/spring-petclinic:selfhosted
```

This demonstrates the ability to switch between JFrog Cloud and self-hosted Artifactory while keeping the application build lifecycle consistent.

The primary differences are:

```text
Repository endpoints
Maven settings
Jenkins credentials
Docker registry destination
```

---

# Self-Hosted Production Considerations

The self-hosted deployment is intentionally a local demonstration.

For a production deployment, additional controls should include:

- TLS/HTTPS.
- Dedicated least-privilege service identities or access tokens.
- Managed persistent storage.
- PostgreSQL backups.
- Artifactory data backups.
- Monitoring and health checks.
- Controlled Jenkins build agents.
- Docker registry access controls.
- High-availability and disaster-recovery planning.

The Artifactory runtime data, database credentials, generated configuration, and secrets are intentionally not committed to this repository.

---

# Assignment Requirements

| Requirement | Implementation |
|---|---|
| Spring PetClinic source | Official Spring PetClinic project |
| Jenkins Cloud pipeline | `Jenkinsfile` |
| Compile code | Maven `compile` stage |
| Run tests | Maven `test` stage |
| Package application | Maven `package` stage |
| Runnable Docker image | `spring-petclinic:assignment` |
| Dependencies through JFrog Cloud | `maven-virtual` Maven mirror |
| Maven Central proxy | `maven-central-remote` |
| JFrog Cloud Docker repository | `docker-local` |
| Cloud Docker publishing | `spring-petclinic:assignment` pushed to JFrog Cloud |
| Secure JFrog authentication | Jenkins Credentials |
| Dockerfile | Root `Dockerfile` |
| Documentation | `README.md` |
| Runnable image deliverable | `spring-petclinic-assignment.tar.gz` |
| Self-hosted pipeline | `Jenkinsfile.selfhosted` |
| Self-hosted Artifactory | Artifactory OSS 7.161.15 with PostgreSQL |
| Self-hosted Maven resolution | Self-hosted `maven-virtual` |
| Self-hosted Docker repository | `docker-local` |
| Self-hosted Docker publishing | `spring-petclinic:selfhosted` published to self-hosted Artifactory |

---

# Final Architecture Summary

The project demonstrates the same CI/CD pattern using both JFrog Cloud and a self-hosted Artifactory instance.

## JFrog Cloud

```text
                       JFrog Cloud
                       /         \
                      v           ^
              Maven Dependencies  |
                      |       Docker Image
                      v           |
                    Jenkins ------+
```

## Self-Hosted Artifactory

```text
                  Self-Hosted Artifactory
                       /         \
                      v           ^
              Maven Dependencies  |
                      |       Docker Image
                      v           |
                    Jenkins ------+
```

In both implementations:

```text
Dependencies:
Maven Central -> Artifactory -> Maven -> Jenkins

Application artifact:
Jenkins -> Docker Build -> Docker Push -> Artifactory
```

This provides centralized dependency management and centralized storage of the Docker application artifact while keeping credentials outside the source repository.

---

# Author

Issam Ben Ameur
