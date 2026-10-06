# Spring PetClinic – Jenkins, Docker & JFrog CI/CD

This repository contains a CI/CD implementation for the Spring PetClinic application using Jenkins, Docker, Maven, and JFrog Artifactory.

The pipeline compiles the application, runs automated tests, packages the Spring Boot application, and builds a runnable Docker image.

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

JFrog Artifactory is used as the Maven repository manager so that project dependencies are resolved through Artifactory rather than directly from Maven Central.

Target dependency flow:

```text
Jenkins
   |
   v
Maven
   |
   v
JFrog Artifactory
   |
   v
Maven Central
```

## Technologies

- Java 17
- Spring Boot
- Maven / Maven Wrapper
- Jenkins
- Docker
- JFrog Artifactory
- GitHub

## Repository Contents

The main CI/CD files are:

```text
.
├── Jenkinsfile
├── Dockerfile
├── README.md
├── pom.xml
├── mvnw
├── mvnw.cmd
└── src/
```

## Jenkins Pipeline

The `Jenkinsfile` defines the CI pipeline.

The pipeline currently contains the following stages:

### 1. Checkout

Retrieves the source code from GitHub.

### 2. Compile

Compiles the application using the Maven Wrapper:

```bash
./mvnw compile
```

### 3. Test

Runs the automated test suite:

```bash
./mvnw test
```

### 4. Package

Packages the Spring Boot application as an executable JAR:

```bash
./mvnw package -DskipTests
```

Tests are skipped during this stage because they have already been executed in the dedicated Test stage.

The generated application artifact is:

```text
target/spring-petclinic-4.0.0-SNAPSHOT.jar
```

### 5. Docker Build

Builds the runnable Docker image:

```bash
docker build -t spring-petclinic:assignment .
```

The resulting image is:

```text
spring-petclinic:assignment
```

## Docker Image

The `Dockerfile` uses Eclipse Temurin Java 17 as the runtime environment.

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY target/spring-petclinic-4.0.0-SNAPSHOT.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Run the Docker Image

If the image has already been built locally:

```bash
docker run --rm -p 8081:8080 spring-petclinic:assignment
```

The application is then available at:

```text
http://localhost:8081
```

Port `8081` is used on the host to avoid conflicting with Jenkins, which is running on port `8080`.

The port mapping is:

```text
Host                 Container
8081  -------------> 8080
                      Spring PetClinic
```

Verify the application with:

```bash
curl -I http://localhost:8081
```

A successful application startup should return an HTTP `200` response.

## Load and Run the Submitted Docker Image

A runnable Docker image is provided separately as:

```text
spring-petclinic-assignment.tar.gz
```

Load the image:

```bash
gunzip -c spring-petclinic-assignment.tar.gz | docker load
```

Verify that the image was loaded:

```bash
docker images spring-petclinic
```

Run the application:

```bash
docker run --rm -p 8081:8080 spring-petclinic:assignment
```

Then open:

```text
http://localhost:8081
```

or verify it from the command line:

```bash
curl -I http://localhost:8081
```

## Build and Run Manually

Clone the repository:

```bash
git clone https://github.com/issambenameur85-wq/spring-petclinic-jfrog-cicd.git
cd spring-petclinic-jfrog-cicd
```

Compile:

```bash
./mvnw compile
```

Run the tests:

```bash
./mvnw test
```

Package the application:

```bash
./mvnw package -DskipTests
```

Build the Docker image:

```bash
docker build -t spring-petclinic:assignment .
```

Run it:

```bash
docker run --rm -p 8081:8080 spring-petclinic:assignment
```

## JFrog Artifactory

JFrog Artifactory is used as the repository manager for Maven dependencies.

The intended dependency resolution flow is:

```text
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

The remote repository acts as a proxy for Maven Central and caches downloaded dependencies.

The virtual repository provides Maven with a single repository endpoint.

Maven is configured to use the Artifactory virtual repository as its mirror, ensuring dependency resolution is routed through JFrog Artifactory.

> JFrog Cloud configuration and verification will be completed before final submission.

## Security

Credentials and access tokens must not be committed to this repository.

JFrog credentials are managed through Jenkins credentials and injected into the pipeline only when required.

The repository contains configuration only and does not contain JFrog passwords, access tokens, or other secrets.

## Verification

The pipeline has been validated through the following flow:

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
Docker Image
    |
    v
Docker Container
    |
    v
HTTP 200
```

The generated Docker image was started locally and the Spring PetClinic application successfully responded with:

```text
HTTP/1.1 200
```

## Bonus – Self-Hosted JFrog Artifactory

A self-hosted Artifactory environment will be provided as an additional demonstration of running the pipeline against a locally deployed Artifactory instance.

This bonus configuration is kept separate from the primary JFrog Cloud implementation so that it does not affect the required pipeline.

## Author

Issam Ben Ameur
