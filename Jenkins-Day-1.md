# Jenkins – Day 1: Fundamentals, CI/CD, Installation and First CI Job

## 1. Introduction to Jenkins

Jenkins is an open-source automation server used to automate software development workflows. It is commonly used for Continuous Integration (CI) and Continuous Delivery or Continuous Deployment (CD).

Jenkins coordinates activities such as source-code checkout, builds, tests, quality checks, artifact packaging and deployments. It integrates with tools such as Git and Maven through jobs, pipelines, plugins, credentials, agents and triggers.

| Tool | Purpose |
|---|---|
| Git | Source Code Management |
| Maven | Java build automation and dependency management |
| Jenkins | CI/CD automation |

## 2. Understanding CI/CD

**Continuous Integration (CI)** is the practice of frequently integrating developers' code changes into a shared repository and validating them through automated builds and tests.

Typical workflow:
1. A developer changes application code.
2. Changes are pushed to a shared Git repository.
3. Jenkins retrieves the source code.
4. Jenkins builds the application.
5. Automated tests run.
6. Jenkins reports the build result.

CI helps identify integration problems early.

**Continuous Delivery** automatically builds, tests and prepares application changes for release. Production release may require manual approval.

**Continuous Deployment** automatically releases changes to production after they pass the required checks, without a separate manual production-release decision.

~~~text
Developer changes code
        |
        v
Push code to Git
        |
        v
Jenkins checkout
        |
        v
Build → Test → Quality checks
        |
        v
Package artifact
        |
        v
Deploy according to release process
~~~

## 3. Understanding a Pipeline

A pipeline is a sequence of connected steps used to complete a process. A Jenkins pipeline commonly includes:

1. **Checkout:** Retrieve source code from Git.
2. **Build:** Compile and package the application.
3. **Test:** Run automated tests.
4. **Code quality:** Perform configured quality checks.
5. **Deploy:** Release the application to a target environment.

A Jenkins Pipeline can be defined in a Jenkinsfile and stored with the application's source code.

Example Declarative Pipeline structure:

~~~groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps { echo 'Checkout stage' }
        }
        stage('Build') {
            steps { echo 'Build stage' }
        }
        stage('Test') {
            steps { echo 'Test stage' }
        }
    }
}
~~~

This is a structural example. Actual commands depend on the project.

## 4. Application Environments

| Environment | Purpose |
|---|---|
| DEV | Developers integrate and test changes. |
| QA | Testers validate application behavior and identify defects. |
| UAT | Clients or business users validate business requirements. |
| PROD | Live environment used by actual customers or end users. |

DEV, QA and UAT are commonly grouped as non-production or pre-production environments. PROD is the live environment. These environments may be separate servers, containers, cloud resources or logical configurations.

## 5. Jenkins Overview

- Open-source automation server.
- Java-based application requiring a supported Java runtime.
- Extensible through plugins.
- Integrates with source control, build tools, testing and deployment systems.
- Commonly uses TCP port 8080 for the web interface with the default configuration.

A plugin extends Jenkins functionality or integrates it with an external tool or service. Install only the plugins required by the workflow and keep them updated.

For cloud installations, restrict access to Jenkins to trusted IP addresses or private networks. Avoid exposing the administration interface to the entire internet.

## 6. Jenkins Installation on Ubuntu

The following steps install Jenkins LTS on Ubuntu using the official Jenkins APT repository. The commands use OpenJDK 21.

### 6.1 Prepare the Ubuntu server

1. Create or start an Ubuntu VM or cloud instance.
2. Connect through a terminal or SSH.
3. Ensure the account has sudo privileges.
4. Configure network access for SSH and Jenkins.

Example SSH command:

~~~bash
ssh -i your-key.pem ubuntu@YOUR_SERVER_PUBLIC_IP
~~~

### 6.2 Update Ubuntu packages

~~~bash
sudo apt update
sudo apt upgrade -y
~~~

### 6.3 Install Java and Fontconfig

~~~bash
sudo apt install -y fontconfig openjdk-21-jre
java -version
~~~

Jenkins requires a supported Java runtime. The Java runtime used to run Jenkins and the JDK used to build an application are separate concerns.

### 6.4 Add the Jenkins repository signing key

~~~bash
sudo install -d -m 0755 /etc/apt/keyrings

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
~~~

### 6.5 Add the Jenkins LTS repository

~~~bash
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
https://pkg.jenkins.io/debian-stable binary/ | \
sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
~~~

The debian-stable repository provides the Jenkins LTS release line for Debian-based distributions, including Ubuntu.

### 6.6 Install Jenkins

~~~bash
sudo apt update
sudo apt install -y jenkins
~~~

### 6.7 Enable and start Jenkins

~~~bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
~~~

A running service should report Active: active (running).

Useful commands:

~~~bash
sudo systemctl restart jenkins
sudo systemctl stop jenkins
sudo journalctl -u jenkins -xe
~~~

### 6.8 Configure network access

Jenkins commonly listens on TCP port 8080. If UFW is enabled, allow access only from a trusted IP:

~~~bash
sudo ufw allow from YOUR_TRUSTED_IP to any port 8080 proto tcp
sudo ufw status
~~~

For an AWS EC2 Ubuntu instance, also allow TCP port 8080 in the EC2 security group only from a trusted IP address or network.

### 6.9 Open Jenkins and complete setup

For a remote server:

~~~text
http://YOUR_SERVER_IP:8080
~~~

For Jenkins on the same Ubuntu computer:

~~~text
http://localhost:8080
~~~

Retrieve the initial administrator password:

~~~bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
~~~

Then unlock Jenkins, install suggested or required plugins, create the first administrator account, confirm the Jenkins URL and finish the setup wizard.

The password path is the standard location when Jenkins uses the default JENKINS_HOME.

## 7. Jenkins Installation on AWS EC2 – Amazon Linux 2023

These commands are for Amazon Linux 2023, which uses DNF.

### 7.1 Prepare the instance

Launch an EC2 instance with Amazon Linux 2023, configure an SSH key pair and security group, then connect:

~~~bash
ssh -i your-key.pem ec2-user@YOUR_EC2_PUBLIC_IP
~~~

Allow SSH (TCP 22) and Jenkins web access (TCP 8080) only from trusted sources.

### 7.2 Update packages and install Java

~~~bash
sudo dnf update -y
sudo dnf install java-21-amazon-corretto -y
java -version
~~~

### 7.3 Add the Jenkins stable repository

~~~bash
sudo wget -O /etc/yum.repos.d/jenkins.repo \
https://pkg.jenkins.io/rpm-stable/jenkins.repo

sudo rpm --import \
https://pkg.jenkins.io/rpm-stable/jenkins.io-2026.key
~~~

### 7.4 Install and start Jenkins

~~~bash
sudo dnf install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
~~~

Open the Jenkins URL using the instance public IP and port 8080. Retrieve the initial password with:

~~~bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
~~~

Complete the setup wizard and restrict port 8080 to trusted sources in the EC2 security group.

## 8. Configure Build Tools in Jenkins

For a Maven-based Java project, the build execution environment needs a compatible JDK, Maven and Git access to the source repository.

Tool installations can be configured from **Manage Jenkins → Tools**. Maven must be available where the build actually runs. Installing Maven only on the controller does not automatically make it available on a separate agent.

## 9. Create the First Jenkins Freestyle CI Job

A Freestyle Project is a Jenkins job type configured mainly through the web interface.

1. Open the Jenkins dashboard and select **New Item**.
2. Enter a job name, such as **Netflix-CI-Job**.
3. Select **Freestyle project** and click **OK**.
4. Under **Source Code Management**, select **Git**.
5. Enter the repository URL.
6. Select the correct branch, such as main or master.
7. Under **Build Steps**, select **Add build step → Execute shell**.
8. Enter the Maven command:

~~~bash
mvn clean package
~~~

The clean phase removes previous Maven build output. The package phase runs the Maven lifecycle through packaging, normally including earlier compilation and test phases unless project configuration changes that behavior.

The generated artifact depends on the packaging configuration in pom.xml; it may be a JAR, WAR or another supported package type.

Save the job, select **Build Now**, open the build entry in **Build History**, and inspect **Console Output**.

A successful Maven build commonly displays:

~~~text
BUILD SUCCESS
Finished: SUCCESS
~~~

## 10. Generated Artifacts and Workspace

Jenkins checks out source code into the job workspace and runs build steps there.

For a job named Netflix-CI-Job, the workspace is commonly:

~~~text
/var/lib/jenkins/workspace/Netflix-CI-Job/
~~~

Maven output is commonly generated under:

~~~text
/var/lib/jenkins/workspace/Netflix-CI-Job/target/
~~~

The artifact name and extension depend on the Maven project configuration. The workspace is a working directory, not permanent artifact storage. Configure Jenkins to archive required artifacts or publish them to an artifact repository such as Nexus Repository or JFrog Artifactory.

## 11. How the First CI Job Works

~~~text
Build Now
   |
   v
Jenkins starts the job
   |
   v
Git checkout
   |
   v
Maven clean
   |
   v
Compile and test lifecycle phases
   |
   v
Maven package
   |
   v
Build result and artifact
~~~

Selecting **Build Now** starts a build manually. Configuring a Git repository does not automatically trigger a build after every commit. For automatic builds, configure a trigger such as a GitHub webhook or Poll SCM.

## 12. Freestyle Project and Pipeline

| Freestyle Project | Pipeline |
|---|---|
| Configured mainly through the Jenkins UI | Commonly defined as code in a Jenkinsfile |
| Useful for simple jobs and learning fundamentals | Useful for multi-stage CI/CD workflows |
| Build steps are configured in the job | Stages and steps are described in the pipeline |
| Less convenient to review as code | Pipeline definition can be version-controlled |

## 13. Troubleshooting

### Jenkins web page does not open

~~~bash
sudo systemctl status jenkins
sudo ss -lntp | grep 8080
~~~

Check the operating-system firewall and, for EC2, the security group's inbound rules.

### Jenkins service fails to start

~~~bash
sudo journalctl -u jenkins -xe
java -version
~~~

Check service logs and confirm the installed Jenkins release supports the Java runtime.

### Git checkout fails

Check the repository URL, branch name, repository visibility, credentials for private repositories and network connectivity from the Jenkins agent.

### Maven command is not found

Check that Maven is installed on the machine or agent executing the job and is available in the execution user's PATH. Alternatively, configure Maven under **Manage Jenkins → Tools**.

### Build fails during tests

Inspect Console Output and test reports. Identify the first meaningful error and check project dependencies, Java compatibility and Maven configuration.

## 14. Important Commands Reference

### Ubuntu

~~~bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
sudo apt install -y jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
~~~

### Amazon Linux 2023

~~~bash
sudo dnf update -y
sudo dnf install java-21-amazon-corretto -y
sudo dnf install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
~~~

### Maven build

~~~bash
mvn clean package
~~~

## 15. Interview Preparation

**1. What is Jenkins?**  
Jenkins is an open-source automation server used to automate software build, test and delivery workflows.

**2. What is Continuous Integration?**  
It is the practice of frequently integrating code changes into a shared repository and validating them with automated builds and tests.

**3. What is Continuous Delivery?**  
It automatically builds, tests and prepares software for release so it can be deployed when approved.

**4. What is Continuous Deployment?**  
It automatically releases changes to production after they pass the required checks.

**5. What is a Jenkins pipeline?**  
A sequence of connected stages and steps that automates software delivery.

**6. What is a Jenkins plugin?**  
An extension that adds Jenkins functionality or integrates it with external tools and services.

**7. What is a Jenkins workspace?**  
The working directory where Jenkins checks out source code and executes a job's build steps.

**8. What does mvn clean package do?**  
It removes previous Maven build output and runs the lifecycle through the package phase, producing the configured artifact when successful.

**9. Does configuring a Git repository automatically trigger a build after every commit?**  
No. An automatic trigger, such as a webhook or Poll SCM, must also be configured.

## 16. Day 1 Learning Summary

In this session, I learned the fundamentals of Jenkins and its role in a CI/CD workflow.

- Git manages source code.
- Maven builds Java applications and manages dependencies.
- Jenkins automates source checkout, building and testing.
- CI validates frequently integrated code changes.
- Continuous Delivery prepares software for release; Continuous Deployment automates production release after successful checks.
- A pipeline connects checkout, build, test and deployment activities.
- DEV, QA and UAT are commonly used for non-production validation, while PROD is the live environment.
- Jenkins can integrate with external tools through plugins.
- A Freestyle job can retrieve source code from Git and run a Maven build.
- Build logs help identify whether a job succeeded or failed.
- Build artifacts depend on project configuration and can be archived or published for later deployment.

The first Freestyle CI job provides a foundation for understanding automated application builds. More advanced workflows can be implemented using Jenkins Pipeline and a Jenkinsfile.
