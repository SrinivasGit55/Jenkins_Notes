# Jenkins Day 2 – Job Parameters, Build Management and Administration

## 1. Introduction

In Day 2, I continued learning Jenkins job configuration, build management and administration.

The main topics covered are:

- Jenkins installation on Ubuntu and Amazon Linux.
- Creating Freestyle jobs.
- Configuring job parameters.
- Managing multiple Java versions in Jenkins.
- Configuring Maven as a Jenkins build tool.
- Changing the Jenkins port.
- Managing concurrent builds.
- Recovering deleted job configurations.
- Introduction to Role-Based Access Control (RBAC).
- Jenkins service troubleshooting.

---

## 2. Jenkins Installation

Jenkins is an automation server used to automate software development activities such as building, testing and deploying applications.

Jenkins requires a supported Java runtime to start and operate.

For this setup, Java 21 is used as the Jenkins runtime.

### 2.1 Install Jenkins on Ubuntu

**Step 1: Update package information**

```bash
sudo apt update
```

**Step 2: Install Java 21**

```bash
sudo apt install -y fontconfig openjdk-21-jre
```

**Step 3: Verify Java**

```bash
java -version
```

**Step 4: Add Jenkins repository key**

```bash
sudo mkdir -p /etc/apt/keyrings

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \\
https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
```

**Step 5: Add Jenkins repository**

```bash
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \\
| sudo tee /etc/apt/sources.list.d/jenkins.list
```

**Step 6: Install Jenkins**

```bash
sudo apt update
sudo apt install -y jenkins
```

**Step 7: Enable and start Jenkins**

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

**Step 8: Check Jenkins status**

```bash
sudo systemctl status jenkins
```

### 2.2 Install Jenkins on Amazon Linux 2023

Amazon Linux 2023 uses DNF as its package manager.

**Step 1: Update packages**

```bash
sudo dnf update -y
```

**Step 2: Install Java 21 and wget**

```bash
sudo dnf install -y java-21-amazon-corretto-devel wget
```

**Step 3: Verify Java**

```bash
java -version
```

**Step 4: Add Jenkins repository**

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo \\
https://pkg.jenkins.io/rpm-stable/jenkins.repo
```

**Step 5: Import Jenkins repository key**

```bash
sudo rpm --import https://pkg.jenkins.io/rpm-stable/jenkins.io-2026.key
```

**Step 6: Install Jenkins**

```bash
sudo dnf install -y jenkins
```

**Step 7: Reload systemd**

```bash
sudo systemctl daemon-reload
```

**Step 8: Enable and start Jenkins**

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

**Step 9: Check Jenkins status**

```bash
sudo systemctl status jenkins
```

### 2.3 Access Jenkins Dashboard

Jenkins normally listens on port `8080`.

Open the following URL in your browser:

```text
http://YOUR_EC2_PUBLIC_IP:8080
```

Make sure the EC2 Security Group allows inbound TCP traffic on port `8080` from your trusted IP address.

To retrieve the initial administrator password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Enter the password in the Jenkins setup wizard and complete the initial configuration.

---

## 3. Creating a Freestyle Job

A Freestyle project is a Jenkins job that allows us to configure source code management, build steps and build triggers through the Jenkins dashboard.

### 3.1 Create a Job

1. Open Jenkins Dashboard.
2. Click **New Item**.
3. Enter the job name, for example `Netflix`.
4. Select **Freestyle project**.
5. Click **OK**.

### 3.2 Configure Source Code Management

1. Open the job configuration.
2. Navigate to **Source Code Management**.
3. Select **Git**.
4. Enter the GitHub repository URL.
5. Select the required branch.

Example:

```text
Repository URL:
https://github.com/username/repository.git

Branch:
*/main
```

For private repositories, configure the required Git credentials in Jenkins.

### 3.3 Configure the Build Step

For a Maven-based Java project:

1. Navigate to **Build Steps**.
2. Select **Invoke top-level Maven targets**.
3. Select the configured Maven installation.
4. Enter the Maven goals:

```bash
clean package
```

Alternatively, use an Execute Shell build step:

```bash
mvn clean package
```

### 3.4 Execute the Job

1. Save the job configuration.
2. Click **Build Now**.
3. Open **Build History**.
4. Select the latest build.
5. Open **Console Output**.

The Console Output displays execution logs and helps identify build errors.

### 3.5 Understanding Maven Build Output

The command:

```bash
mvn clean package
```

performs two important operations:

- `clean`: Removes previous build output, normally from the `target` directory.
- `package`: Executes the Maven lifecycle up to the package phase, including earlier lifecycle phases such as compilation and testing, subject to the project configuration.

For a typical Maven project, the generated artifact is stored in:

```text
target/
```

For example:

```text
target/application.jar
```

The actual artifact name and format depend on the project's Maven configuration.

### 3.6 Jenkins Workspace

Jenkins creates a separate workspace for each job.

For example:

```text
/var/lib/jenkins/workspace/Netflix
```

The workspace contains checked-out project files and build output.

The workspace is not a permanent artifact repository. Important build artifacts should be archived or published to an artifact repository such as Nexus or Artifactory.

---

## 4. Jenkins Job Parameters

Parameters allow users to provide values when starting a Jenkins job.

They are useful when the same job needs to run with different configurations.

For example, an application may need deployment to different environments:

- Development
- QA
- UAT
- Production

### 4.1 Enable Parameters

1. Open the Jenkins job.
2. Click **Configure**.
3. Select **This project is parameterized**.
4. Click **Add Parameter**.
5. Choose the required parameter type.

### 4.2 Choice Parameter

A Choice Parameter allows users to select one value from a predefined list.

Example:

```text
Name: ENVIRONMENT

Choices:
dev
qa
uat
prod
```

When the job is executed, Jenkins displays the available choices.

### 4.3 Multiple Choice Parameters

A multiple-choice parameter allows users to select multiple values, depending on the parameter implementation or plugin being used.

Example:

```text
Environment:
dev
qa
uat
prod

Client:
client1
client2
client3
```

Multiple selections are not the same as Jenkins' standard Choice Parameter. Additional functionality may require a suitable plugin.

### 4.4 String Parameter

A String Parameter accepts a single-line text value.

Example:

```text
Name: CLIENT

Default Value: client1
```

A user can enter:

```text
client3
```

Jenkins passes the entered value to the build as a parameter.

If the user enters:

```text
prod,client3
```

Jenkins treats it as one string unless the build logic separates the values.

### 4.5 Multi-line String Parameter

A Multi-line String Parameter accepts text containing multiple lines.

Example:

```text
Name: DEPLOYMENT_DETAILS

Value:
Environment=QA
Application=Netflix
Version=1.0
```

This can be useful when a build requires multiple lines of input.

### 4.6 Boolean Parameter

A Boolean Parameter provides a true/false selection.

Example:

```text
Name: RUN_TESTS

Default Value: true
```

It can be used to control whether a particular build step should execute.

### 4.7 File Parameter

A File Parameter allows a user to upload a file when starting a job.

Example uses:

- Configuration files.
- Input data files.
- Deployment-related files.

The uploaded file can then be accessed by the build process.

---

## 5. Managing Multiple Java Versions in Jenkins

Different Java projects may require different JDK versions.

For example:

| Project | Required JDK |
|---|---|
| Project A | Java 8 |
| Project B | Java 11 |
| Project C | Java 17 |
| Project D | Java 21 |

It is important to distinguish between:

**Jenkins runtime Java:** The Java version used to run Jenkins itself.

**Application build Java:** The Java version selected to compile, test and package an application.

For current Jenkins releases:

- Java 21 is supported.
- Java 25 is also supported.
- Java 17, Java 11 and Java 8 are no longer supported for running current Jenkins releases.

For this learning setup, Java 21 is used as the Jenkins runtime, while older JDKs are installed for application builds.

### 5.1 Install Multiple JDKs on Ubuntu

The following command uses Ubuntu OpenJDK packages.

```bash
# Install Java 8, 11, 17 and 21
sudo apt install -y openjdk-8-jdk-headless openjdk-11-jdk-headless openjdk-17-jdk-headless openjdk-21-jdk-headless

# View installed JDK directories
ls /usr/lib/jvm/
```

**Important package availability note:**

The availability of these packages depends on the Ubuntu release and its configured repositories.

OpenJDK 8 and 11 may not be available in the default repositories of some Ubuntu releases. If APT cannot locate a package, an appropriate vendor repository or another supported JDK distribution may be required.

### 5.2 Install Multiple JDKs on Amazon Linux 2023

Amazon Linux provides Amazon Corretto packages.

```bash
# Install Java 8, 11, 17 and 21
sudo dnf install -y java-1.8.0-amazon-corretto-devel java-11-amazon-corretto-devel java-17-amazon-corretto-devel java-21-amazon-corretto-devel

# View installed JDK directories
ls /usr/lib/jvm/
```

### 5.3 Verify Installed JDKs

List the installed Java directories:

```bash
ls /usr/lib/jvm/
```

Check the version of a specific Java installation by using its full path.

Example:

```bash
/usr/lib/jvm/java-17-openjdk-amd64/bin/java -version
```

The actual path depends on the Linux distribution and installed JDK package.

### 5.4 Configure JDKs in Jenkins

After installing the JDKs:

1. Open Jenkins Dashboard.
2. Navigate to **Manage Jenkins → Tools**.
3. Find **JDK installations**.
4. Click **Add JDK**.
5. Add each JDK separately.

Example configuration:

| Name | Java Version |
|---|---|
| JDK8 | Java 8 |
| JDK11 | Java 11 |
| JDK17 | Java 17 |
| JDK21 | Java 21 |

For every entry:

- Enter a meaningful name.
- Enter the corresponding JDK installation directory in `JAVA_HOME`.
- Uncheck **Install automatically** because the JDK was installed manually.
- Save the configuration.

To find the correct installation paths:

```bash
ls /usr/lib/jvm/
```

The directory names can differ between Ubuntu and Amazon Linux.

**Important:** Enter the JDK directory, not the path to the `java` executable.

### 5.5 Select a JDK for a Jenkins Job

When configuring a Freestyle job:

1. Open the job configuration.
2. Find the JDK selection option.
3. Select the required JDK.
4. Save the configuration.
5. Run the build.

For example:

- A Java 8 application can select `JDK8`.
- A Java 11 application can select `JDK11`.
- A Java 17 application can select `JDK17`.
- A Java 21 application can select `JDK21`.

This allows different Jenkins jobs to build applications with different Java versions.

**Agent consideration:** Tools must be available on the machine where the build actually runs. If Jenkins uses separate build agents, install or provision the required JDKs on those agents as well.

---

## 6. Configure Maven in Jenkins

Maven is a Java build automation tool used to manage project dependencies, compile source code, run tests and package applications.

Jenkins can manage Maven installations through its Tools configuration.

### 6.1 Configure Maven

1. Open Jenkins Dashboard.
2. Navigate to **Manage Jenkins → Tools**.
3. Find **Maven installations**.
4. Click **Add Maven**.

Example:

| Setting | Value |
|---|---|
| Name | Maven-3.9 |
| Install automatically | Enabled |
| Version | Select an available Maven version |

Save the configuration.

Jenkins can download and manage Maven automatically through the configured installer.

### 6.2 Why Configure Maven in Jenkins?

Different projects may use different Maven versions.

Configuring Maven in Jenkins allows build jobs to select the required Maven installation.

Example:

```text
Java Project
     |
     v
Jenkins Job
     |
     +---- JDK17
     |
     +---- Maven-3.9
     |
     v
mvn clean package
     |
     v
Build Artifact
```

### 6.3 Using Maven with Different JDKs

A Jenkins Pipeline can reference configured tools by name.

Example:

```groovy
pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven-3.9'
    }

    stages {
        stage('Check Tools') {
            steps {
                sh 'java -version'
                sh 'javac -version'
                sh 'mvn -version'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

The names in the Pipeline must match the names configured under Jenkins Tools.

**Compatibility consideration:** The Jenkins Maven Integration Plugin has an additional requirement: its Maven build process must use Java at least as new as the Jenkins controller and agent runtime.

For older Java application builds, a Pipeline or shell-based Maven invocation can be used to select the required JDK without relying on that plugin's Maven build step.

---

## 7. Changing the Jenkins Port

Jenkins normally uses port `8080`.

Sometimes we may need to change the port because another application is already using it or because a different port is required.

For example, change the Jenkins port from:

```text
8080
```

to:

```text
8090
```

### 7.1 Use a systemd Override

Modern Jenkins Linux packages use systemd to manage the Jenkins service.

Instead of editing the original service file directly, use a systemd override.

Run:

```bash
sudo systemctl edit jenkins
```

Add the following configuration:

```ini
[Service]
Environment="JENKINS_PORT=8090"
```

Save and close the editor.

### 7.2 Restart Jenkins

```bash
sudo systemctl restart jenkins
```

### 7.3 Check Jenkins Status

```bash
sudo systemctl status jenkins
```

If required, inspect the effective service configuration:

```bash
sudo systemctl cat jenkins
```

### 7.4 Access Jenkins Using the New Port

```text
http://YOUR_EC2_PUBLIC_IP:8090
```

Also update the EC2 Security Group to allow inbound TCP traffic on port `8090` from a trusted source.

**Why use a systemd override?**

The original Jenkins service file is managed by the package installer and should not be edited directly. A systemd override stores custom settings separately and is preserved when the Jenkins package is upgraded.

---

## 8. Managing Concurrent Builds

Concurrent builds allow Jenkins to execute multiple builds at the same time.

This is useful when multiple developers or projects trigger builds simultaneously.

### 8.1 Enable Concurrent Builds for a Job

1. Open the Jenkins job.
2. Click **Configure**.
3. Find **Execute concurrent builds if necessary**.
4. Enable the option.
5. Save the configuration.

This allows the job to have multiple builds running at the same time, subject to available executor capacity.

### 8.2 Configure Jenkins Executors

1. Open **Manage Jenkins**.
2. Navigate to **Nodes**.
3. Select the built-in node.
4. Click **Configure**.
5. Find the number of executors.
6. Set the required number.

For example:

```text
Number of executors: 4
```

The number of executors determines how many tasks Jenkins can execute simultaneously on that node.

### 8.3 Job Concurrency vs Executors

These settings serve different purposes:

| Setting | Purpose |
|---|---|
| Execute concurrent builds | Allows multiple builds of the same job |
| Number of executors | Determines how many tasks can run simultaneously on a node |

Increasing executors does not automatically improve performance.

Too many concurrent builds can consume excessive:

- CPU
- RAM
- Disk space
- Network resources

Executor capacity should be based on the available system resources.

---

## 9. Jenkins Troubleshooting

During Jenkins administration, some common issues may occur.

### 9.1 Jenkins Service Is Not Running

Check the service:

```bash
sudo systemctl status jenkins
```

Start Jenkins:

```bash
sudo systemctl start jenkins
```

Restart Jenkins:

```bash
sudo systemctl restart jenkins
```

Stop Jenkins:

```bash
sudo systemctl stop jenkins
```

Enable Jenkins at system startup:

```bash
sudo systemctl enable jenkins
```

### 9.2 Jenkins Is Not Accessible

Check the following:

1. Jenkins service is running.
2. Correct port is configured.
3. EC2 Security Group allows the Jenkins port.
4. The instance has a reachable public IP or appropriate private network access.
5. No other service is occupying the configured port.

### 9.3 Public IP Changes After EC2 Restart

When an EC2 instance is stopped and started, its automatically assigned public IPv4 address can change.

If Jenkins is accessed using the old public IP, the dashboard may no longer be reachable.

An Elastic IP can provide a persistent public IPv4 address for an EC2 instance, subject to AWS pricing and allocation rules.

### 9.4 Check Jenkins Logs

For service-related troubleshooting:

```bash
sudo journalctl -u jenkins
```

To follow logs continuously:

```bash
sudo journalctl -u jenkins -f
```

These commands are useful for identifying startup errors, configuration problems and service failures.

---

## 10. Recovering Deleted Jenkins Jobs

Jenkins jobs contain configuration information such as:

- Source code repository.
- Build steps.
- Build parameters.
- Build triggers.
- Other job settings.

If a job is deleted accidentally, its configuration may need to be recovered.

### 10.1 Job Configuration History Plugin

The demonstration uses the **Job Configuration History** plugin.

Installation steps:

1. Open Jenkins Dashboard.
2. Navigate to **Manage Jenkins → Plugins**.
3. Open the Available plugins section.
4. Search for `Job Configuration History`.
5. Install the plugin.
6. Configure the plugin if required.

The plugin maintains historical copies of job and system configuration files.

It can provide:

- Configuration history.
- Comparison between previous configurations.
- Information about configuration changes.
- Restoration of older job configurations.
- Access to deleted-job configuration history.

**Important:** The plugin's configuration history is not a replacement for complete Jenkins backups.

Important Jenkins data should be backed up separately, including `JENKINS_HOME`, job configurations, credentials and required plugin data.

---

## 11. Introduction to Role-Based Access Control (RBAC)

RBAC stands for **Role-Based Access Control**.

It is an access management approach in which permissions are assigned according to user roles.

For example, an organization may have:

| Role | Responsibility |
|---|---|
| Administrator | Manages Jenkins configuration and permissions |
| Developer | Creates and executes application builds |
| Tester | Executes and monitors testing jobs |
| Viewer | Views job status and build information |

RBAC helps organizations control who can access Jenkins and what actions they can perform.

The topic is introduced here; detailed RBAC configuration was not demonstrated.

---

## 12. Important Java Runtime Clarification

Current Jenkins releases support Java 21 and Java 25.

Java 17 support ended with the Jenkins 2.541.x LTS line, while Java 8 and Java 11 support ended much earlier.

The Java version used by Jenkins does not have to be the same as the Java version used to build an application.

For example:

```text
Jenkins Controller
       |
       +---- Java 21 Runtime
       |
       +---- Build Job 1 → JDK 8
       |
       +---- Build Job 2 → JDK 11
       |
       +---- Build Job 3 → JDK 17
       |
       +---- Build Job 4 → JDK 21
```

Java 25 can also be selected as a Jenkins runtime where appropriate.

The important point is to use a supported Java version for the Jenkins controller and agents, while selecting application build JDKs according to project requirements.

---

## 13. Important Jenkins Commands

| Command | Purpose |
|---|---|
| `sudo systemctl start jenkins` | Starts Jenkins |
| `sudo systemctl stop jenkins` | Stops Jenkins |
| `sudo systemctl restart jenkins` | Restarts Jenkins |
| `sudo systemctl status jenkins` | Checks Jenkins status |
| `sudo systemctl enable jenkins` | Enables Jenkins at startup |
| `sudo systemctl daemon-reload` | Reloads systemd configuration |
| `sudo systemctl edit jenkins` | Creates or edits a Jenkins service override |
| `sudo systemctl cat jenkins` | Displays the effective Jenkins service configuration |
| `sudo journalctl -u jenkins` | Displays Jenkins service logs |
| `sudo journalctl -u jenkins -f` | Follows Jenkins logs continuously |
| `java -version` | Checks the default Java runtime |
| `mvn -version` | Checks Maven version |
| `ls /usr/lib/jvm/` | Lists common JDK installation directories |

---

## 14. Key Learning Points

Through these activities, I learned:

1. How Jenkins is installed and started on Ubuntu and Amazon Linux.
2. How to create a Freestyle job and connect it to a Git repository.
3. How Maven commands are configured as Jenkins build steps.
4. How Jenkins parameters make jobs reusable with different inputs.
5. Why multiple JDK versions are useful when building different Java applications.
6. How to configure JDK and Maven installations through Jenkins Tools.
7. How to change the Jenkins port using a systemd override.
8. How concurrent builds and executors affect Jenkins build execution.
9. How Jenkins configuration history can help recover job configurations.
10. Why RBAC is important for controlling Jenkins access.
11. Why Jenkins runtime Java and application build Java must be considered separately.

### Practical DevOps Understanding

In a real DevOps environment, Jenkins may support multiple development teams and applications.

Different applications may require different Java and Maven versions. By configuring these tools centrally, Jenkins jobs can select the appropriate build environment.

Job parameters can make deployment jobs reusable across development, QA, UAT and production environments.

Concurrent builds can improve build throughput when sufficient system resources are available, while access control helps protect Jenkins administration and deployment operations.

These capabilities are important when managing CI/CD automation for multiple projects.

---

## 15. Official Documentation Used for Technical Verification

The following official documentation was used to verify the updated installation procedures, Java compatibility and Jenkins administration practices.

1. [Jenkins – Java Support Policy](https://www.jenkins.io/doc/book/platform-information/support-policy-java/)
2. [Jenkins – Linux Installation](https://www.jenkins.io/doc/book/installing/linux/)
3. [Jenkins – Managing Tools](https://www.jenkins.io/doc/book/managing/tools/)
4. [Jenkins – Managing systemd Services](https://www.jenkins.io/doc/book/system-administration/systemd-services/)
5. [Jenkins – Job Configuration History Plugin](https://plugins.jenkins.io/jobConfigHistory/)
6. [AWS – Amazon Corretto Installation](https://docs.aws.amazon.com/corretto/latest/corretto-21-ug/amazon-linux-install.html)
7. [AWS – Java in Amazon Linux 2023](https://docs.aws.amazon.com/linux/al2023/ug/java.html)
