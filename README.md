# Jenkins Notes

A structured collection of my Jenkins learning notes, CI/CD concepts, practical examples, commands and troubleshooting points.

## Learning Progress

### Day 1 – Jenkins Fundamentals, CI/CD and First CI Job

[Read Jenkins Day 1 Notes](Jenkins-Day-1.md)

Covered topics:

- Introduction to Jenkins and its role in CI/CD
- Continuous Integration and Continuous Delivery/Deployment
- Understanding DEV, QA, UAT and PROD environments
- Jenkins installation and initial configuration on AWS EC2
- Jenkins dashboard and initial administrator setup
- Freestyle jobs and build configuration
- Connecting a Jenkins job to a Git repository
- Maven build execution using `mvn clean package`
- Jenkins workspace, build output and generated artifacts
- Manual build triggering and basic CI workflow

### Day 2 – Job Parameters, Build Management and Administration

[Read Jenkins Day 2 Notes](Jenkins-Day-2.md)

Covered topics:

- Parameterized Freestyle jobs and build-time inputs
- Configuring jobs for different application examples
- Maven job configuration and tool integration
- Changing the Jenkins service port
- Executors and concurrent build execution
- Jenkins service management and troubleshooting
- EC2 stop/start behaviour and public IP changes
- Elastic IP and stable server access
- Job configuration recovery using Job Configuration History
- Introduction to Jenkins role-based access control (RBAC)
- Jenkins scripts and configuration files

### Day 3 – Build Triggers, Scheduling and Workspace

[Read Jenkins Day 3 Notes](Jenkins-Day-3.md)

Covered topics:

- Jenkins build triggers and scheduling
- Cron syntax and scheduled builds
- Poll SCM and periodic source-code checks
- GitHub webhook integration for push events
- Difference between Poll SCM and webhook-based triggering
- Throttle Concurrent Builds and build concurrency
- Remote build triggering and authentication considerations
- Jenkins workspace and build execution flow

### Day 4 – Jenkins Controller and Agent Architecture

[Read Jenkins Day 4 Notes](Jenkins-Day-4.md)

Covered topics:

- Jenkins controller and agent architecture (formerly Master and Slave)
- Distributed builds using AWS EC2 instances
- SSH connectivity and agent prerequisites
- Java runtime requirements for Jenkins agents
- Configuring agents, executors and labels
- Assigning jobs to specific agents
- Installing Git, Maven and required tools on agents
- Running a Freestyle job on an agent
- Agent monitoring and troubleshooting
- Workspace and disk-space management
- Linking Jenkins jobs and understanding downstream execution
- Introduction to SonarQube and code quality analysis

### Day 5 – Jenkins Pipeline and Pipeline as Code

[Read Jenkins Day 5 Notes](Jenkins-Day-5.md)

Covered topics:

- Jenkins Pipeline fundamentals and terminology
- Scripted Pipeline and Declarative Pipeline
- Declarative Pipeline structure: `pipeline`, `agent`, `stages`, `stage` and `steps`
- Creating and executing a basic Pipeline
- Running Linux commands through Pipeline `sh` steps
- Single-stage and multi-stage Pipelines
- Connecting Pipeline jobs to GitHub repositories
- Using the Pipeline Syntax Generator
- Integrating Maven build commands into a Pipeline
- Pipeline as Code and the purpose of a `Jenkinsfile`
- Executing multiple commands in a Pipeline step
- Selecting Jenkins agents using `agent any` and labels
- Agent execution, Java compatibility and workspace verification
- Understanding the end-to-end Jenkins Pipeline workflow

### Day 6 – Pipeline Variables, Post-Build Actions and RBAC

[Read Jenkins Day 6 Notes](Jenkins-Day-6.md)

Covered topics:

- Understanding Groovy variables and Pipeline environment variables
- Pipeline-level and stage-level environment variable scopes
- Jenkins-provided variables such as `BUILD_NUMBER`, `JOB_NAME` and `WORKSPACE`
- Using build numbers to identify generated artifacts
- Declarative Pipeline `post` section
- `always`, `success` and `failure` post conditions
- Pipeline-level and stage-level post actions
- Troubleshooting runtime failures versus Pipeline syntax failures
- Jenkins authentication and authorization
- Role-Based Authorization Strategy plugin
- Global roles, item roles and agent roles
- Managing permissions for Fresher, Developer and Administrator roles
- Reusing roles when new employees join
- Least Privilege and Jenkins access control
- Jenkins service startup using modern `systemctl` commands

## Additional Learning Resource

- [SonarQube for DevOps](https://github.com/CloudWithVarJosh/SonarQube-For-DevOps) — Additional learning resource for code quality, static analysis and DevSecOps integration.

## Learning Approach

These notes are organized by learning day and focus on understanding Jenkins through CI/CD concepts, practical job configuration, Pipeline execution, distributed build scenarios, access management and troubleshooting. The examples build a foundation for automating application builds and integrating quality checks into DevOps workflows.
