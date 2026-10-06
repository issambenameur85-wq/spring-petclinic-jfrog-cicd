# Spring PetClinic – Jenkins, Docker & JFrog CI/CD

This repository contains a CI/CD implementation for the Spring PetClinic application using Jenkins, Maven, Docker, GitHub, and JFrog Artifactory.

The pipeline:

1. Checks out the source code from GitHub.
2. Compiles the application.
3. Runs the automated tests.
4. Packages the Spring Boot application.
5. Builds a runnable Docker image.

All Maven dependency resolution performed by the CI pipeline is routed through JFrog Cloud Artifactory rather than directly to Maven Central.

The project is based on the official Spring PetClinic application:

https://github.com/spring-projects/spring-petclinic

---

# CI/CD Architecture

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

Maven dependency resolution follows this path:

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

JFrog Artifactory acts as the controlled repository layer between the build environment and the public Maven repository.

---

# Technologies

- Java 17
- Spring Boot
- Maven / Maven Wrapper
- Jenkins
- Docker
- JFrog Artifactory
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

`.mvn/jfrog-selfhosted-settings.xml` configures the separate self-hosted Artifactory bonus path.

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

The `jenkins_home` volume keeps Jenkins configuration persistent even if the Jenkins container is recreated.

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

This enables the pipeline to execute:

```bash
docker build -t spring-petclinic:assignment .
```

This configuration is intentionally limited to the local assignment environment. See the Security Considerations section for production considerations.

---

# Configure Jenkins with GitHub

The Jenkins job uses **Pipeline script from SCM**.

This allows Jenkins to retrieve both the application source code and the pipeline definition directly from GitHub.

Create a Jenkins **Pipeline** job with:

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

The `Jenkinsfile` defines the required JFrog Cloud CI pipeline.

`Jenkinsfile.selfhosted` defines the optional self-hosted Artifactory bonus pipeline.

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

Compiles the application while resolving Maven dependencies through JFrog Artifactory:

```bash
./mvnw -s .mvn/jfrog-settings.xml compile
```

Using the Maven Wrapper ensures the expected Maven version can be used without requiring Maven to be installed manually on the Jenkins environment.

## 3. Test

Runs the automated test suite using the same JFrog Maven configuration:

```bash
./mvnw -s .mvn/jfrog-settings.xml test
```

The test stage is kept separate so that failures are clearly visible in the Jenkins pipeline.

The validated pipeline execution produced:

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

Tests are skipped during this stage because they have already been executed in the dedicated Test stage.

The generated artifact is:

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

# JFrog Cloud Artifactory

JFrog Cloud Artifactory is used as the repository manager for Maven dependencies.

The required dependency flow is:

```text
Jenkins
   |
   v
Maven
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

## Repository Architecture

Two Maven repositories are used in JFrog Cloud.

### Remote Repository

```text
maven-central-remote
```

This is a remote Maven repository that proxies:

```text
https://repo1.maven.org/maven2/
```

When an artifact is not already available from Artifactory's remote cache, Artifactory retrieves it from Maven Central.

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
JFrog remote cache
   |
   v
Maven
```

Future requests for cached artifacts can be served by Artifactory without retrieving the artifact from Maven Central again.

### Virtual Repository

```text
maven-virtual
```

The virtual repository contains:

```text
maven-central-remote
```

Maven communicates with the virtual repository rather than directly with the remote repository.

This provides a single repository endpoint to the build:

```text
Maven
   |
   v
maven-virtual
   |
   v
maven-central-remote
```

This architecture also makes it possible to add internal Maven repositories later without changing the repository endpoint used by Maven clients.

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

This instructs Maven to route repository requests through the JFrog mirror rather than using repository definitions such as Maven Central directly.

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

The Jenkins credential ID used by the pipeline is:

```text
jfrog-credentials
```

The credential contains:

```text
Username: JFrog username
Password: JFrog access token
```

The pipeline injects these values only around Maven operations.

Conceptually:

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

Jenkins masks the token in console output.

The repository must never contain:

```text
JFrog passwords
JFrog access tokens
Jenkins secrets
GitHub private keys
```

---

# Verifying JFrog Dependency Resolution

A successful Maven build alone does not prove that dependencies are being resolved through Artifactory.

The Jenkins console output was therefore inspected to verify the repository being used.

The pipeline produced entries such as:

```text
Downloading from jfrog:
https://petclinictestcicd.jfrog.io/artifactory/maven-virtual/...

Downloaded from jfrog:
https://petclinictestcicd.jfrog.io/artifactory/maven-virtual/...
```

This verifies that Maven is using the configured JFrog virtual repository.

The effective dependency flow is therefore:

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
Maven local repository (~/.m2)
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

Port `8083` is used on the host because Jenkins uses port `8080` and the local self-hosted Artifactory environment uses ports `8081` and `8082`.

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
REPOSITORY             TAG
spring-petclinic       assignment
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

# Verification

The complete required CI/CD flow has been validated:

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
Docker Container
   |
   v
HTTP 200
```

The Maven logs also verify dependency resolution through:

```text
https://petclinictestcicd.jfrog.io/artifactory/maven-virtual/
```

The application Docker image was successfully started and returned:

```text
HTTP/1.1 200
```

This verifies that the Docker image produced by the CI process is runnable.

---

# Security Considerations

This assignment uses a local Jenkins environment designed to keep the setup simple and reproducible.

## Secrets

Secrets are not committed to Git.

JFrog credentials are stored using Jenkins Credentials and injected into Maven operations only when required.

The access token is masked by Jenkins in console output.

## Docker Socket

For this local environment, Jenkins runs as `root` and has access to:

```text
/var/run/docker.sock
```

Mounting the Docker socket gives the Jenkins container significant control over the host Docker daemon.

This approach was intentionally chosen for the local assignment environment so Jenkins can build Docker images without introducing additional infrastructure.

For a production environment, I would instead use dedicated Jenkins build agents with controlled permissions and an appropriately secured container build strategy rather than exposing the host Docker socket directly to the Jenkins controller.

## GitHub

Jenkins uses anonymous HTTPS read access because the GitHub repository is public.

Developer write access is handled separately using SSH.

---

# Bonus – Self-Hosted JFrog Artifactory

The optional self-hosted Artifactory bonus has also been completed and validated end to end.

JFrog Artifactory OSS 7.161.15 is deployed locally with PostgreSQL using JFrog's Docker Compose distribution. The self-hosted implementation is intentionally isolated from the required JFrog Cloud pipeline so the primary solution remains independently functional.

## Bonus Architecture

```text
GitHub
   |
   v
Jenkins
   |
   v
Maven Wrapper
   |
   v
Self-Hosted JFrog Artifactory
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

## Self-Hosted Repository Configuration

The Artifactory instance exposes port `8082` and contains two Maven repositories:

- `maven-central-remote` — remote repository proxying Maven Central at `https://repo1.maven.org/maven2/`.
- `maven-virtual` — virtual repository containing `maven-central-remote` and providing a single endpoint to Maven clients.

The self-hosted Maven configuration is stored separately from the Cloud configuration:

```text
.mvn/jfrog-selfhosted-settings.xml
```

Its mirror uses runtime environment variables rather than committed credentials or host-specific configuration:

```xml
<server>
    <id>jfrog-selfhosted</id>
    <username>${env.ARTIFACTORY_USERNAME}</username>
    <password>${env.ARTIFACTORY_PASSWORD}</password>
</server>

<mirror>
    <id>jfrog-selfhosted</id>
    <name>Self-hosted JFrog Artifactory</name>
    <url>${env.ARTIFACTORY_URL}/artifactory/maven-virtual/</url>
    <mirrorOf>*</mirrorOf>
</mirror>
```

## Self-Hosted Jenkins Pipeline

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

The bonus pipeline executes the same application lifecycle while using the self-hosted repository manager:

```text
Checkout -> Compile -> Test -> Package -> Docker Build
```

The resulting bonus image is tagged:

```text
spring-petclinic:selfhosted
```

## Jenkins Credentials for Self-Hosted Artifactory

The self-hosted pipeline uses two Jenkins credentials:

```text
jfrog-selfhosted-credentials
    -> ARTIFACTORY_USERNAME
    -> ARTIFACTORY_PASSWORD

jfrog-selfhosted-url
    -> ARTIFACTORY_URL
```

The Artifactory URL is injected at runtime instead of being hard-coded in `Jenkinsfile.selfhosted`. This keeps environment-specific configuration outside source control and avoids triggering Spring PetClinic's NoHttp Checkstyle rule for the local HTTP endpoint.

For this local Docker Desktop environment, the injected URL is:

```text
http://host.docker.internal:8082
```

`host.docker.internal` is required because Jenkins runs inside a container; `localhost` inside that container would refer to Jenkins itself rather than the Artifactory service exposed by the host.

## Bonus Verification

Connectivity from the Jenkins container to Artifactory was verified before executing the full pipeline. Maven output then confirmed dependency resolution through the self-hosted virtual repository with entries such as:

```text
Downloading from jfrog-selfhosted:
http://host.docker.internal:8082/artifactory/maven-virtual/...
```

The complete self-hosted Jenkins pipeline was successfully executed:

```text
Checkout       PASS
Compile        PASS
Test           PASS
Package        PASS
Docker Build   PASS
```

This demonstrates that the application build can switch between JFrog Cloud and self-hosted Artifactory without redesigning the pipeline. The Maven settings, Jenkins credentials, and repository endpoint are the primary configuration differences.

## Self-Hosted Production Considerations

The self-hosted deployment is intentionally a local demonstration. For a production deployment I would additionally use TLS/HTTPS, a dedicated least-privilege service identity or access token instead of an administrator account, managed persistent storage and PostgreSQL backups, monitoring and health checks, and an appropriate high-availability/disaster-recovery strategy.

The Artifactory Docker Compose distribution, database credentials, runtime data, and generated environment files are intentionally not committed to this repository.

---

# Assignment Requirements

| Requirement | Implementation |
|---|---|
| Spring PetClinic source | Official Spring PetClinic project |
| Jenkins pipeline | `Jenkinsfile` |
| Compile code | Maven `compile` stage |
| Run tests | Maven `test` stage |
| Package application | Maven `package` stage |
| Runnable Docker image | `spring-petclinic:assignment` |
| Dependencies through JFrog Cloud | `maven-virtual` Maven mirror |
| Secure JFrog authentication | Jenkins Credentials |
| Dockerfile | Root `Dockerfile` |
| Documentation | `README.md` |
| Runnable image deliverable | `spring-petclinic-assignment.tar.gz` |
| Self-hosted Artifactory | Completed optional bonus using Artifactory OSS 7.161.15 + PostgreSQL |

---

# Author

Issam Ben Ameur
