![Alt text](
# CICD-Using-Jenkins-Nexus-Sonarqube-Slack

## Prerequisites
- AWS account with necessary IAM permissions
- Jenkins, Nexus, SonarQube installed on EC2 instances
- Docker, AWS CLI, Maven installed on Jenkins
- Slack Webhook URL for notifications
- GitHub repository set up with code

## Steps

### 1. GitHub Webhook Configuration
- Update GitHub repository webhook with the new Jenkins IP to trigger builds on code push.

### 2. Copy Docker Files
- Copy the necessary Docker files from the `vprofile` repository to your repository to enable Docker-based builds.

### 3. Prepare Jenkinsfiles
- Create two separate `Jenkinsfile`s in your repository: one for **Staging** and one for **Production** environments. These files will define the build and deployment steps.

### 4. AWS Setup
- **IAM Setup**: Create necessary IAM roles and policies for Jenkins to access AWS resources.
- **ECR Setup**: Set up an Elastic Container Registry (ECR) repository for storing Docker images.

### 5. Jenkins Setup
- **Install Required Plugins**: Install the following Jenkins plugins:
  - **Amazon ECR Plugin**
  - **Docker Plugin**
  - **Pipeline Plugin**

- **Install Docker Engine & AWS CLI**: On the Jenkins instance, install Docker and AWS CLI to interact with ECR and ECS.

### 6. Write Jenkinsfile for Build & Publish Docker Image to ECR
- The `Jenkinsfile` for both Staging and Production defines the following steps:
  - **Build**: Compile the application using Maven and package the artifact.
  - **Unit Test**: Run unit tests on the application.
  - **Integration Test**: Run integration tests.
  - **Code Analysis**:
    - Use **Checkstyle** for coding style checks.
    - Use **SonarQube** for static code analysis and quality checks.
  - **Publish to Nexus**: Upload the artifact to Nexus Repository Manager for versioned storage.
  - **Build Docker Image**: Use Docker to build the application image.
  - **Push Docker Image to ECR**: Push the Docker image to AWS ECR for deployment.

### 7. ECS Setup
- **ECS Cluster**: Create an ECS cluster to manage Docker containers.
- **Task Definition**: Define a task in ECS specifying the Docker image and its configurations.
- **Service**: Create an ECS service to run the Docker containers.

### 8. Deploy Docker Image to ECS
- Deploy the built Docker image from ECR to the ECS cluster using the ECS task definition and service.

### 9. Repeat for Production
- Follow the same process for the **Production ECS cluster**.

### 10. Promoting Docker Image for Production
- After successfully deploying to the **Staging** environment, promote the Docker image to **Production** for deployment.

### 11. Slack Notifications
- Set up Slack Webhook in Jenkins to notify the team of build and deployment statuses.

## Scripts

### Jenkins Setup Script

```bash
#!/bin/bash
sudo apt update
sudo apt install openjdk-11-jdk -y
sudo apt install maven wget unzip -y

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install jenkins -y
```

### Jenkinsfile Example

```groovy
pipeline {
    agent any
    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "172.31.40.209:8081"
        NEXUS_REPOSITORY = "vprofile-release"
        NEXUS_REPOGRP_ID = "vprofile-grp-repo"
        NEXUS_CREDENTIAL_ID = "nexuslogin"
        ARTVERSION = "${env.BUILD_ID}"
    }

    stages {
        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Archiving artifacts...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Checkstyle analysis completed'
                }
            }
        }

        stage('CODE ANALYSIS WITH SONARQUBE') {
            environment {
                scannerHome = tool 'sonarscanner4'
            }

            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile-repo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('PUBLISH TO NEXUS') {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml";
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    artifactPath = filesByGlob[0].path;
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: NEXUS_REPOGRP_ID,
                            version: ARTVERSION,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId, classifier: '', file: artifactPath, type: pom.packaging]
                            ]
                        );
                    } else {
                        error "Artifact could not be found";
                    }
                }
            }
        }
    }
}
```

### Nexus Setup Script

```bash
#!/bin/bash
# Nexus setup script
sudo rpm --import https://yum.corretto.aws/corretto.key
sudo curl -L -o /etc/yum.repos.d/corretto.repo https://yum.corretto.aws/corretto.repo

sudo yum install -y java-17-amazon-corretto-devel wget -y

mkdir -p /opt/nexus/
mkdir -p /tmp/nexus/
cd /tmp/nexus/
NEXUSURL="https://download.sonatype.com/nexus/3/latest-unix.tar.gz"
wget $NEXUSURL -O nexus.tar.gz
tar xzvf nexus.tar.gz
NEXUSDIR=$(ls /tmp/nexus/ | head -n 1)
cp -r /tmp/nexus/* /opt/nexus/
useradd nexus
chown -R nexus.nexus /opt/nexus 
```

### SonarQube Setup Script

```bash
#!/bin/bash
# SonarQube setup script
sudo apt-get update -y
sudo apt-get install openjdk-11-jdk -y
sudo apt-get install postgresql postgresql-contrib -y

# Configure SonarQube
sudo curl -O https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.3.0.34182.zip
sudo unzip sonarqube-8.3.0.34182.zip -d /opt/
sudo mv /opt/sonarqube-8.3.0.34182/ /opt/sonarqube
```


# Sonar-Analysis-Properties
sonar.projectKey=vprofile
sonar.projectName=vprofile-repo
sonar.projectVersion=1.0
sonar.sources=src/
sonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/
sonar.junit.reportsPath=target/surefire-reports/
sonar.jacoco.reportsPath=target/jacoco.exec
sonar.java.checkstyle.reportPaths=target/checkstyle-result.xml

## Conclusion
This CI pipeline allows for seamless integration between code repositories, build servers, code analysis tools, artifact repositories, and cloud services. By leveraging Jenkins, Nexus, SonarQube, and Slack, teams can automate the process of building, testing, analyzing, and deploying their applications efficiently and reliably.




