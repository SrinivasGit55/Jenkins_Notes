# Jenkins Day 4 – Master and Slave Architecture

## 1. Jenkins Service After an EC2 Restart

When Jenkins is installed on an EC2 instance, the Jenkins service runs in the background. The Jenkins dashboard is commonly accessed using:

```text
http://<EC2-Public-IP>:8080
```

### Scenario: Jenkins dashboard is not accessible

**Situation:** The EC2 instance was stopped and started again. After restarting it, the Jenkins dashboard is no longer accessible using the previous URL.

**Possible causes:**

1. The Jenkins service did not start.
2. The EC2 public IP address changed.
3. Port 8080 is not allowed through the EC2 Security Group.
4. Jenkins is running, but there is another connectivity or application issue.

**Investigation:**

Check the Jenkins service:

```bash
sudo systemctl status jenkins
```

If the service is stopped, start it:

```bash
sudo systemctl start jenkins
```

Enable Jenkins to start automatically when the operating system boots:

```bash
sudo systemctl enable jenkins
```

If Jenkins is already running, check the EC2 public IP and Security Group rules. Port 8080 must be allowed from the intended source.

**Important distinction:**

- `systemctl start jenkins` starts the service now.
- `systemctl enable jenkins` configures it to start automatically during boot.
- An EC2 stop/start can change its public IPv4 address unless an Elastic IP is associated with the instance.

Older Linux administration material may use `chkconfig` for service startup configuration. On modern systemd-based Amazon Linux systems, use `systemctl`.

**Learning:** When a Jenkins URL stops working after an EC2 restart, do not immediately assume Jenkins is broken. Check the service, IP address and network access separately.

---

## 2. Jenkins Master and Slave Architecture

Jenkins can distribute work across multiple machines instead of executing every build on the machine where Jenkins itself is installed.

The architecture traditionally uses the terms **Master** and **Slave**.

| Traditional term | Current terminology | Responsibility |
|---|---|---|
| Master | Controller | Manages Jenkins, jobs, configuration and scheduling |
| Slave | Agent | Executes build tasks assigned by the controller |
| Node | Node | A machine or environment connected to Jenkins |
| Executor | Executor | A slot that allows a node to run a task |

The traditional terminology is useful when understanding older Jenkins demonstrations and documentation. Current Jenkins documentation generally uses **Controller** and **Agent**.

### 2.1 What does the Master (Controller) do?

The controller is the central Jenkins server.

Its responsibilities include:

- Providing the Jenkins dashboard.
- Managing jobs and configurations.
- Managing credentials and plugins.
- Scheduling builds.
- Assigning work to available agents.
- Tracking build results.

The controller coordinates the work; it does not necessarily need to perform the actual build.

### 2.2 What does a Slave (Agent) do?

An agent is a separate machine connected to the Jenkins controller.

It executes assigned tasks such as:

- Downloading source code from Git.
- Running Maven builds.
- Executing tests.
- Generating application packages.
- Running deployment scripts.

For example, an agent can be an EC2 Linux instance. Agents can also use other supported operating systems, such as Windows and macOS.

### 2.3 Why do we need agents?

Consider a Jenkins controller that handles ten projects.

If every project builds on the controller:

- Build processes consume controller CPU and memory.
- Multiple builds can compete for resources.
- Heavy workloads can affect the Jenkins dashboard and other controller operations.
- Scaling build capacity becomes difficult.

Instead, Jenkins can distribute builds across multiple agents.

**Example:**

```text
                    Jenkins Controller
                           |
              +------------+------------+
              |            |            |
           Agent 1       Agent 2       Agent 3
              |            |            |
           Java Build   Test Jobs    Other Builds
```

The controller manages the jobs, while the agents provide the execution capacity.

### 2.4 Understanding Executors

An executor represents a task execution slot on a Jenkins node.

Suppose an agent has four executors. It can execute up to four eligible tasks concurrently, subject to available resources and other scheduling restrictions.

| Node | Configured executors | Maximum concurrent tasks from executor capacity |
|---|---:|---:|
| Slave 1 | 3 | 3 |
| Slave 2 | 4 | 4 |
| Slave 3 | 3 | 3 |

These are the example values used in this setup.

**Important:** More executors do not automatically mean better performance. Each build consumes resources such as CPU, memory and disk space. The executor count should be selected according to the machine's capacity and workload.

This also connects to the concurrent-build concept covered earlier: executors determine how many tasks a node can run at the same time. They are not the same as the total number of builds Jenkins can queue.

---

## 3. Preparing EC2 Instances as Jenkins Agents

The demonstration uses three EC2 instances as worker machines.

```text
EC2 Instance 1: Jenkins Controller
EC2 Instance 2: Slave 1
EC2 Instance 3: Slave 2
EC2 Instance 4: Slave 3
```

The Jenkins controller is already installed. The remaining three instances will be connected to it as agents.

### 3.1 Agent prerequisites

Before configuring an agent in Jenkins, check the following:

1. The agent instance is running.
2. The controller can reach the agent over the network.
3. SSH access is available for Linux agents using the SSH launch method.
4. The SSH username is correct for the selected EC2 AMI.
5. The private key matches the key pair used when creating the EC2 instance.
6. A compatible Java runtime is installed on the agent.

The controller and agent must also use compatible Jenkins-supported Java runtimes. The Java runtime used to launch the Jenkins agent is not necessarily the same Java version required by the application being built.

### 3.2 Network connectivity

For an SSH-launched Linux agent, the controller needs to connect to the agent on TCP port 22.

A typical EC2 Security Group arrangement is:

| Instance | Required access |
|---|---|
| Jenkins Controller | Dashboard access on port 8080, restricted to trusted users or networks |
| Agent instances | SSH on port 22 from the Jenkins controller |

Do not open SSH access to the entire internet unnecessarily. Prefer allowing SSH only from the controller's private IP address or its Security Group where the network design supports it.

The agent does not need a public IP if the controller can reach it through private networking.

---

## 4. Configuring Permanent Agents in Jenkins

Navigate to:

```text
Manage Jenkins → Nodes
```

Create a new node and select the permanent agent option.

The configuration contains several important fields.

### 4.1 Node name

The node name identifies the agent in Jenkins.

Example:

```text
slave-one
```

A meaningful name makes it easier to identify where a build is running.

### 4.2 Number of executors

This determines how many tasks the agent can execute concurrently.

For the demonstration:

```text
Slave 1: 3
Slave 2: 4
Slave 3: 3
```

These values are configuration examples, not recommended defaults for every EC2 instance.

### 4.3 Remote root directory

Example from the demonstration:

```text
/tmp
```

This is the directory on the agent where Jenkins stores its remote working data.

It is not the same thing as the Jenkins controller's workspace directory.

For a more controlled setup, use a dedicated directory, for example:

```bash
sudo mkdir -p /home/ec2-user/jenkins-agent
sudo chown ec2-user:ec2-user /home/ec2-user/jenkins-agent
```

Then configure:

```text
/home/ec2-user/jenkins-agent
```

Using `/tmp` is convenient for a demonstration, but temporary directories can be cleaned or cleared. A dedicated directory is easier to manage for a persistent agent.

### 4.4 Labels

A label identifies an agent or group of agents.

Examples:

```text
slave-one
slave-two
slave-three
```

Labels allow Jenkins jobs to select the machine on which they should execute.

For example, a job configured with the label `slave-one` will be restricted to agents matching that label.

Labels can also represent capabilities rather than machine names:

```text
linux
java
maven
docker
```

This is useful when several agents have similar environments.

### 4.5 Usage

The node usage setting controls whether Jenkins can use an agent for any eligible job or only jobs that explicitly request it.

The demonstration uses the agent for assigned builds.

When configuring production agents, choose the usage strategy according to the workload and the purpose of the machine.

### 4.6 Launch method: Launch agents via SSH

For the Linux EC2 instances, select:

```text
Launch agents via SSH
```

Provide the following information:

- **Host:** Private IP address or hostname of the agent.
- **Credentials:** SSH credentials for the Linux user.
- **Host Key Verification Strategy:** A strategy for verifying the agent's SSH host key.

The SSH credential can use the private key associated with the EC2 key pair.

For example, if the EC2 instance was created with a key pair named `jenkins-worker.pem`, the corresponding private key is used to authenticate the SSH connection.

**Security note:** The private key is sensitive. Store it in Jenkins Credentials rather than exposing it in job scripts or configuration files.

The demonstration selects a host key verification option that does not verify the host key. This may simplify an initial connection, but it weakens SSH security because Jenkins cannot properly confirm the identity of the remote host.

For a secure setup, use a trusted known-hosts configuration or another supported host key verification method.

---

## 5. Running the Netflix Job on Slave 1

The demonstration uses an existing Freestyle job named:

```text
Netflix
```

The job uses:

- Git to retrieve source code.
- Maven to build the application.
- Jenkins to coordinate the build.

Initially, the job runs on the built-in node. The next step is to move its execution to Slave 1.

### 5.1 Restricting the job to Slave 1

Open:

```text
Netflix → Configure
```

Enable the option that restricts where the project can run and enter the agent label:

```text
slave-one
```

Save the configuration and run the job.

Jenkins now looks for an available node matching that label.

### 5.2 Why did the build fail?

An agent being online does not guarantee that it has all the tools required by a job.

For example, the controller may have Git, Java and Maven installed, but the agent may not.

When a job is assigned to an agent, commands are executed in the agent's environment. Jenkins does not automatically transfer the controller's installed software to the agent.

This is an important distinction:

- Jenkins controller: manages and schedules the job.
- Jenkins agent: executes the job and needs the required build tools.

### 5.3 Installing the build dependencies

Check the tools required by the job on the agent.

For the Netflix Maven job, the environment needs:

- Git, to retrieve the repository.
- A compatible Java runtime/JDK, according to the Jenkins and application requirements.
- Maven, to execute the build.

For Amazon Linux 2023, the package manager is DNF. For example, Java 21 Corretto can be installed using:

```bash
sudo dnf install java-21-amazon-corretto-devel -y
```

Git and Maven can also be installed using the appropriate packages available for the operating system.

Check the installations:

```bash
git --version
java -version
mvn -version
```

After installing the required tools, run the job again.

**Learning:** A build that succeeds on the controller can fail on an agent because the two machines have different software environments. Every agent must be prepared for the workloads it is expected to execute.

---

## 6. Agent Troubleshooting Scenarios

The three agents in this demonstration expose different problems. They should not all be treated as the same failure.

### Scenario 1: Slave 1 – Agent is connected, but the build fails

**Situation:** Slave 1 has been configured in Jenkins and appears online. However, the Netflix job fails after being assigned to it.

**What this tells us:**

The agent connection is working. Jenkins can communicate with the machine and assign work to it.

The problem may be in the build environment rather than the Jenkins-to-agent connection.

**Investigation:**

1. Open the failed build's Console Output.
2. Identify the command that failed.
3. Check whether the command is available on the agent.
4. Connect to the agent and verify Git, Java and Maven.
5. Install missing dependencies and rerun the job.

**Root cause:** The agent does not have all the tools required by the job.

**Resolution:** Install the missing tools on Slave 1 and execute the build again.

**Why this matters:** An online agent only confirms that Jenkins can use the node. It does not confirm that the node is ready to build every application.

### Scenario 2: Slave 2 – Agent launch fails because Java is missing

**Situation:** Slave 2 is configured in Jenkins with its SSH connection details, but the agent does not successfully come online.

**Investigation:**

1. Open the node configuration.
2. Check the agent launch status.
3. Inspect the agent log or connection error.
4. Verify whether Java is installed on the agent.
5. Check the Java version and whether the executable is available.

**Root cause:** Jenkins needs a compatible Java runtime on the agent to launch the Jenkins agent process. If that runtime is missing or incompatible, the agent cannot start correctly.

**Resolution:**

Install a Jenkins-supported Java runtime on the agent. For the verified environment in this setup, Java 21 is an appropriate example:

```bash
sudo dnf install java-21-amazon-corretto-headless -y
```

Check the result:

```bash
java -version
```

Then reconnect or relaunch the agent from Jenkins.

**Important distinction:**

- Java required to launch the Jenkins agent is a Jenkins runtime requirement.
- Java required to compile an application is a build-toolchain requirement.

They may be the same version, but they do not have to be. A project may need a different JDK from the one used to run the Jenkins agent, provided the environment is configured correctly.

### Scenario 3: Slave 3 – SSH credential and Java problems

**Situation:** Slave 3 is added as another permanent agent. The initial connection fails with an SSH-related error. After addressing the SSH configuration, the agent still needs its Java runtime.

**Investigation – SSH problem:**

Check:

- Is the agent's private IP correct?
- Is port 22 reachable from the controller?
- Is the SSH username correct?
- Does the configured private key match the EC2 key pair?
- Does the Linux user have permission to access the required directory?

A wrong username, incorrect private key or blocked SSH connection can prevent Jenkins from connecting to the machine.

**Resolution:** Correct the host, username, key or network configuration and retry the connection.

**Investigation – Java problem:**

Once SSH connectivity is working, inspect the agent launch log again. If Java is missing or incompatible, the agent process will still fail.

Install a compatible Java runtime and relaunch the agent.

#### Amazon Linux 2 versus Amazon Linux 2023

The demonstration also highlights a common package-management issue.

| Operating system | Package manager |
|---|---|
| Amazon Linux 2 | YUM; older setup instructions may use `amazon-linux-extras` |
| Amazon Linux 2023 | DNF |

For Amazon Linux 2023, use:

```bash
sudo dnf install java-21-amazon-corretto-headless -y
```

For development tools such as a JDK:

```bash
sudo dnf install java-21-amazon-corretto-devel -y
```

Do not copy an Amazon Linux 2 installation command blindly onto Amazon Linux 2023. First identify the operating system:

```bash
cat /etc/os-release
```

Then choose the correct package manager and package names.

**Why this matters:** Two EC2 instances can both run Amazon Linux but use different releases and package-management procedures. Always identify the OS version before troubleshooting installation commands.

---

## 7. Monitoring Jenkins Nodes

Jenkins provides node-management views that help administrators understand the state and workload of connected machines.

Navigate to:

```text
Manage Jenkins → Nodes
```

Depending on the Jenkins version and installed plugins, node pages may expose information such as:

- Node status and availability.
- Number of executors.
- Current or recent builds.
- System information.
- Build history.
- Node statistics.

### Why monitor agents?

Imagine three agents are connected, but one agent is running several long builds while the others are idle.

Monitoring helps an administrator investigate:

- Whether an agent is online.
- Whether its executors are occupied.
- Whether builds are distributed as expected.
- Whether a particular node is overloaded.
- Whether a build is failing only on one agent.

This is useful when diagnosing build delays and deciding whether to adjust executor counts or add more agent capacity.

---

## 8. Troubleshooting Low Disk Space on an Agent

### Scenario: Agent has insufficient disk space

**Situation:** The agent's available disk space is very low. The demonstration identifies approximately 0.4 GB available in `/tmp`.

When Jenkins executes builds, it may download source code, dependencies and build files. These operations consume disk space.

If the agent runs out of space, builds may fail during source checkout, dependency download, compilation or packaging.

**Investigation:**

Check filesystem usage:

```bash
df -h
```

Check which directories are consuming space:

```bash
sudo du -sh /tmp/* 2>/dev/null
```

Inspect the Jenkins agent's working directory and identify old workspaces or temporary files that can safely be removed.

The demonstration uses:

```bash
cd /tmp
rm -rf *
```

This command is destructive. It removes matching entries in the current directory and should not be copied blindly into a production environment.

It can delete files that are still required by running processes or other applications.

**Safer approach:**

1. Identify which filesystem is full.
2. Find the directories consuming the most space.
3. Identify which files belong to Jenkins and which are used by other processes.
4. Confirm that affected builds are no longer running.
5. Remove only confirmed obsolete workspaces or temporary files.
6. Check disk usage again.

For example, if a specific old workspace has been confirmed as safe to remove:

```bash
sudo rm -rf /path/to/confirmed/old-workspace
```

Use the exact path only after verifying it.

**Learning:** Disk-space problems are infrastructure problems, not necessarily Jenkins configuration problems. Monitor agent storage and clean up only files that are safe to remove.

---

## 9. Linked Jobs – Connecting Netflix and Deployment

Jenkins can trigger one job after another. This allows a workflow to be divided into separate jobs instead of putting every operation into a single job.

The demonstration creates another Freestyle job:

```text
Deployment
```

The existing job is:

```text
Netflix
```

### 9.1 Configure the relationship

Open:

```text
Netflix → Configure → Post-build Actions
```

Select:

```text
Build other projects
```

Enter:

```text
Deployment
```

Save the configuration.

Now, after the Netflix job completes successfully under the configured trigger condition, Jenkins can start the Deployment job.

### 9.2 Upstream and downstream jobs

These terms describe the relationship between the jobs.

| Term | Job | Meaning |
|---|---|---|
| Upstream job | Netflix | The job that triggers another job |
| Downstream job | Deployment | The job triggered by the upstream job |

```text
Netflix (Upstream)
       |
       | Build completes successfully
       v
Deployment (Downstream)
```

### 9.3 Why split jobs?

Consider a basic application workflow:

1. Retrieve source code.
2. Build the application.
3. Run tests.
4. Prepare the deployment package.
5. Deploy the application.

The build and deployment activities can be separated into different jobs.

This provides clearer responsibilities and makes it easier to identify where a workflow failed.

For example:

- If Netflix fails, investigate the build.
- If Netflix succeeds but Deployment fails, investigate the deployment job.

**Important:** Triggering a downstream job does not automatically deploy an application. The downstream job must contain the required deployment commands or deployment configuration.

Linked Freestyle jobs are one way to connect jobs. More complex CI/CD workflows can also be represented using Jenkins Pipeline.

---

## 10. Introduction to SonarQube

SonarQube is a code quality and static code analysis platform.

It can help identify issues such as:

- Bugs.
- Code smells.
- Security-related code findings.
- Maintainability problems.
- Duplicated code.

### How does it fit into a CI/CD workflow?

A typical workflow might look like this:

```text
Developer pushes code
        |
        v
Jenkins retrieves source code
        |
        v
Build application
        |
        v
Run tests
        |
        v
SonarQube code analysis
        |
        v
Continue toward deployment
```

SonarQube analysis and application packaging are different activities.

- Maven can compile, test and package a Java application.
- SonarQube analyzes code quality and reports findings.
- Jenkins coordinates the steps and can use the analysis result when deciding whether the workflow should continue.

SonarQube is introduced here as a related DevOps tool; it is a separate topic from configuring Jenkins agents.

---

## 11. Key Technical Understanding

After configuring multiple Jenkins agents, the important concepts are:

1. **Controller and Agent:** The controller manages Jenkins; agents execute assigned work.
2. **Executors:** Each executor is a task execution slot. Executor count affects concurrency and resource consumption.
3. **Labels:** Labels allow jobs to select suitable agents.
4. **SSH connectivity:** SSH credentials, usernames, host details and network access must be correct for SSH-launched Linux agents.
5. **Agent runtime:** A compatible Java runtime is required to launch the Jenkins agent process.
6. **Build dependencies:** Git, Maven and the required application JDK must be available on the node that executes the job.
7. **Operating system differences:** Amazon Linux 2 and Amazon Linux 2023 use different package-management approaches.
8. **Disk management:** Agent workspaces and temporary files consume storage; cleanup must be deliberate.
9. **Linked jobs:** An upstream job can trigger a downstream job, separating build and deployment responsibilities.
10. **Code quality:** SonarQube adds code analysis to the CI/CD workflow.
