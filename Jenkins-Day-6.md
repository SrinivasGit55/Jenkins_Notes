# Jenkins – Day 6: Pipeline Variables, Post-Build Actions and RBAC

## 1. Introduction

This session covers three important Jenkins concepts:

1. Jenkins Pipeline Variables
2. Post-Build Actions
3. Role-Based Access Control (RBAC)

These concepts help manage information inside Pipelines, perform actions after builds finish, and control what different users can do in Jenkins.

---

## 2. Jenkins Pipeline Variables

### 2.1 What are Variables?

A variable is a named value that can be accessed and reused during program or Pipeline execution. In Jenkins, variables can hold application names, environment names, build information, deployment locations and configuration values.

It is important to distinguish Groovy variables, Pipeline environment variables and Jenkins-provided environment variables.

### 2.2 Groovy Variables

Jenkins Pipeline uses Groovy as its scripting foundation. Variables can be declared using Groovy syntax.

```groovy
pipeline {
    agent any

    stages {
        stage('Variables') {
            steps {
                script {
                    def name = 'Ramesh'
                    def location = 'Hyderabad'

                    echo "Name: ${name}"
                    echo "Location: ${location}"
                }
            }
        }
    }
}
```

**Understanding:**

- `def` declares a Groovy variable.
- `name` and `location` are user-defined variables.
- `script {}` allows Groovy scripting inside a Declarative Pipeline.
- `${name}` and `${location}` insert variable values into the output.

Groovy variables follow Groovy's scope rules. A variable declared inside a block may not be accessible outside that block.

### 2.3 Pipeline Environment Variables

Declarative Pipeline provides the `environment` directive to define environment variables.

```groovy
pipeline {
    agent any

    environment {
        NAME = 'Ramesh'
        LOCATION = 'Hyderabad'
    }

    stages {
        stage('Display Variables') {
            steps {
                echo "Name: ${env.NAME}"
                echo "Location: ${env.LOCATION}"

                sh 'echo "Build Number: $BUILD_NUMBER"'
            }
        }
    }
}
```

**Understanding:**

- `environment` defines environment variables.
- `NAME` and `LOCATION` are defined by the user.
- `env.NAME` accesses the value from Pipeline Groovy.
- `$BUILD_NUMBER` accesses the environment variable from the shell.

There are two important scopes:

- **Pipeline-level environment:** Variables declared directly inside the Pipeline's `environment` block are available across its stages.
- **Stage-level environment:** Variables declared inside a stage's `environment` block are available to steps within that stage.

These variables can also be passed to external processes, such as shell scripts executed by Jenkins.

### 2.4 Global Environment Variables

Jenkins administrators can configure environment variables at the controller level.

A typical configuration location is:

**Manage Jenkins → System → Global properties → Environment variables**

These variables can be made available to builds across jobs, depending on the configuration.

For example:

```text
ENVIRONMENT=Development
```

**Important distinction:**

- Groovy variables are declared in Pipeline code.
- Pipeline environment variables are configured using the `environment` directive.
- Global environment properties are configured at the Jenkins controller level.
- Jenkins-provided environment variables are automatically supplied by Jenkins.

Do not store passwords, access keys or tokens as ordinary global environment variables. Use Jenkins Credentials for sensitive information.

### 2.5 Jenkins-Provided Environment Variables

Jenkins automatically provides environment variables containing information about the current job and build.

| Variable | Description |
|---|---|
| `BUILD_NUMBER` | Current build number |
| `BUILD_ID` | Current build ID; identical to `BUILD_NUMBER` in modern Jenkins |
| `JOB_NAME` | Name of the current Jenkins job |
| `WORKSPACE` | Absolute path of the current job's workspace |
| `BUILD_URL` | URL of the current build |
| `JENKINS_URL` | Jenkins URL, when configured |
| `NODE_NAME` | Name of the node executing the build |

These variables do not need to be manually declared.

Example:

```groovy
pipeline {
    agent any

    stages {
        stage('Display Jenkins Information') {
            steps {
                echo "Build Number: ${env.BUILD_NUMBER}"
                echo "Job Name: ${env.JOB_NAME}"
                echo "Workspace: ${env.WORKSPACE}"
                echo "Node Name: ${env.NODE_NAME}"
            }
        }
    }
}
```

They can also be accessed inside shell commands:

```groovy
sh '''
    echo "Build Number: $BUILD_NUMBER"
    echo "Job Name: $JOB_NAME"
'''
```

The Linux `env` command displays environment variables available to the current process:

```bash
env
```

For troubleshooting, inspect only the required variables rather than printing the entire environment. Environment output may contain sensitive information.

### 2.6 Practical Scenario – Identifying Build Artifacts

**Situation:** A development team builds a Java application using Jenkins. Every successful build produces an application JAR file.

**Problem:** If every build produces an artifact with the same name, such as `application.jar`, it becomes difficult to identify which build generated a particular artifact.

**Investigation:** Jenkins already provides a `BUILD_NUMBER` variable that identifies the current build.

**Resolution:** Use the build number when naming the artifact.

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'

                sh '''
                    cp target/*.jar application-${BUILD_NUMBER}.jar
                '''
            }
        }
    }
}
```

For example:

```text
application-1.jar
application-2.jar
application-3.jar
```

**Why this is useful:** Build numbers help identify which Jenkins execution generated an artifact. This is useful when investigating deployment problems, tracking releases and comparing build outputs.

---

## 3. Jenkins Post-Build Actions

### 3.1 What are Post-Build Actions?

Post-build actions are operations performed after a Jenkins build or Pipeline finishes.

During a CI/CD workflow, stages may perform source checkout, compilation, testing, code quality analysis, artifact generation and deployment. After execution, Jenkins may need to perform additional actions depending on the final result.

Examples include:

- Sending a success notification.
- Sending a failure notification.
- Collecting diagnostic information.
- Performing cleanup operations.

In a Declarative Pipeline, these actions are configured using the `post` section.

### 3.2 Post Section Syntax

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building application'
                sh 'echo Build completed'
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution has finished'
        }

        success {
            echo 'Pipeline completed successfully'
        }

        failure {
            echo 'Pipeline failed'
        }
    }
}
```

The three conditions demonstrated in this session are:

| Condition | Execution | Purpose |
|---|---|---|
| `always` | Regardless of the final result | Perform actions that should normally run after completion |
| `success` | When the Pipeline succeeds | Perform success-related actions |
| `failure` | When the Pipeline fails | Perform failure-related actions |

### 3.3 Understanding the Conditions

**`always`** executes after the relevant Pipeline or stage completes, regardless of whether it succeeded or failed.

Typical uses include logging, cleanup and collecting diagnostic information.

```groovy
post {
    always {
        echo 'Build execution finished'
    }
}
```

**`success`** executes when the relevant Pipeline or stage completes successfully.

```groovy
post {
    success {
        echo 'Application built successfully'
    }
}
```

Typical uses include success notifications and release-related actions when appropriate.

**`failure`** executes when the relevant Pipeline or stage fails.

```groovy
post {
    failure {
        echo 'Application build failed'
    }
}
```

Typical uses include failure notifications, diagnostic logging and alerts to the development team.

Jenkins also supports other conditions, including `unstable`, `aborted`, `changed`, `fixed`, `regression`, `unsuccessful` and `cleanup`.

### 3.4 Pipeline-Level and Stage-Level Post Actions

The `post` section can be configured at two levels.

**Pipeline-level post:** Used for actions that should happen after the overall Pipeline finishes.

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building application'
            }
        }

        stage('Test') {
            steps {
                echo 'Testing application'
            }
        }
    }

    post {
        always {
            echo 'Pipeline has finished'
        }
    }
}
```

**Stage-level post:** Used for actions associated with a particular stage.

```groovy
pipeline {
    agent any

    stages {
        stage('Test') {
            steps {
                sh 'echo Running tests'
            }

            post {
                always {
                    echo 'Test stage has finished'
                }
            }
        }
    }
}
```

The choice depends on whether the action belongs to an individual stage or the entire Pipeline.

### 3.5 Practical Scenario – Build Failure Notification

**Situation:** A Java application is built using Jenkins. Sometimes the application fails during compilation because of errors in the source code.

**Problem:** The development team may not immediately know that the build has failed.

**Investigation:** Jenkins maintains the execution status of the Pipeline. The `post` section can use that status to decide which action to perform.

**Resolution:**

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }

    post {
        success {
            echo 'Application build completed successfully'
        }

        failure {
            echo 'Application build failed. Check the console output.'
        }

        always {
            echo 'Build execution finished'
        }
    }
}
```

In a real CI/CD environment, the messages can be replaced with email, Slack or other notification integrations.

**Why this is useful:** Post-build actions automate the response to build outcomes. Developers do not need to manually check every Jenkins job to know whether it succeeded or failed.

### 3.6 Troubleshooting – Why Did the `always` Condition Not Execute?

There is an important difference between a runtime failure and a Pipeline syntax or compilation failure.

**Case 1 – Runtime command failure**

```groovy
steps {
    sh 'this-command-does-not-exist'
}
```

The Pipeline syntax is valid, but the shell command fails during execution. Jenkins has already started executing the Pipeline. The Pipeline can normally enter its `post` section, where `always` can execute.

**Case 2 – Pipeline syntax or compilation failure**

A Jenkinsfile may contain missing closing braces or an invalid Declarative Pipeline structure. If Jenkins cannot compile or construct a valid Pipeline execution, normal post-processing cannot be relied upon.

**Important understanding:**

- Runtime failure means a Pipeline step fails after execution has started.
- Syntax or compilation failure means Jenkins cannot correctly construct the Pipeline execution.
- An abrupt controller shutdown or forced termination may also prevent normal post-processing.

When troubleshooting missing notifications or cleanup actions, first determine whether Jenkins successfully started the Pipeline and reached execution.

---

## 4. Role-Based Access Control (RBAC)

### 4.1 What is RBAC?

Role-Based Access Control (RBAC) is an authorization approach in which permissions are assigned to roles, and users are assigned to those roles.

Instead of configuring permissions separately for every employee, an administrator can create reusable roles.

Example:

```text
Jenkins
 |
 |-- Fresher Role
 |      |-- Overall Read
 |      |-- Job Read
 |
 |-- Developer Role
 |      |-- Overall Read
 |      |-- Job Read
 |      |-- Job Build
 |
 |-- Administrator Role
        |-- Overall Administer
```

This makes user access easier to manage when multiple employees use the same Jenkins environment.

### 4.2 Authentication vs Authorization

| Term | Meaning | Example |
|---|---|---|
| Authentication | Identifies who is accessing Jenkins | Ramesh logs in using his account |
| Authorization | Determines what the user is allowed to do | Ramesh can view jobs but cannot build them |

Jenkins must first identify the user through its configured security realm. Authorization then determines which actions that user can perform.

The Role-Based Authorization Strategy plugin manages authorization. It does not itself create or authenticate user accounts.

Jenkins can use its built-in user database or integrate with external identity systems such as LDAP, Active Directory or an identity provider.

### 4.3 Creating Jenkins Users

When Jenkins uses its built-in user database, users can be created through:

**Manage Jenkins → Users → Create User**

For example:

- Ramesh
- Suresh
- Mahesh

Each user receives an individual account and authentication credentials. The user-creation process may differ when Jenkins uses an external authentication provider.

### 4.4 Role-Based Authorization Strategy Plugin

The session demonstrates the **Role-Based Authorization Strategy** plugin.

This plugin allows administrators to:

- Create global roles.
- Create item-specific roles.
- Create agent-specific roles.
- Assign roles to users and groups.
- Manage permissions through reusable roles.

**Installation and configuration:**

1. Open **Manage Jenkins → Plugins**.
2. Search for **Role-based Authorization Strategy**.
3. Install a version compatible with the installed Jenkins version.
4. Open **Manage Jenkins → Security** or **Configure Global Security**, depending on the Jenkins UI.
5. Find the Authorization section.
6. Select **Role-Based Strategy**.
7. Save the configuration.
8. Open the role-management page to create and assign roles.

The plugin version must be compatible with the Jenkins controller version.

### 4.5 Types of Roles

The Role Strategy plugin supports three important role categories.

**1. Global Roles**

Global roles apply permissions across Jenkins items.

Example: A developer may receive Overall/Read, Job/Read and Job/Build. This allows the developer to access and build jobs according to the configured permissions.

**2. Item Roles**

Item roles restrict permissions to specific Jenkins jobs or folders.

Example:

```text
Jenkins
 |
 |-- Development
 |      |-- Java-App
 |      |-- Python-App
 |
 |-- Production
        |-- Payment-Service
        |-- Customer-Service
```

A developer may need access to Development jobs but should not be allowed to access production deployment jobs.

An item role can be configured to match the required job or folder names. This provides more targeted access than granting permissions across all Jenkins jobs.

**3. Agent Roles**

Agent roles control permissions associated with Jenkins agents. They are useful when access to particular build nodes needs to be restricted.

### 4.6 Important Jenkins Permissions

| Permission | Meaning |
|---|---|
| Overall/Read | Allows access to the Jenkins interface |
| Job/Read | Allows users to view jobs |
| Job/Build | Allows users to trigger builds |
| Job/Configure | Allows users to modify job configuration |
| Job/Create | Allows users to create jobs |
| Job/Delete | Allows users to delete jobs |
| Overall/Administer | Grants full Jenkins administrator privileges |

**Important distinction:** Overall/Read and Job/Read are different permissions. A user may need Overall/Read to access Jenkins and Job/Read to view particular jobs.

---

## 5. Practical RBAC Demonstration – Fresher and Experienced Employees

### 5.1 Scenario Introduction

Imagine a company where multiple developers use the same Jenkins controller.

- **Ramesh:** Fresher
- **Suresh:** Experienced employee

Both employees need Jenkins access, but their responsibilities are different.

**Problem:** Giving every employee full administrator access creates unnecessary security risks. For example, a new employee could accidentally delete an important job or modify a deployment configuration.

The administrator therefore creates separate roles according to employee responsibilities.

### 5.2 Creating the Fresher Role

The administrator creates a role named `Fresher`.

Initially, the role receives:

- Overall/Read
- Job/Read

The administrator assigns this role to Ramesh.

**Result:** Ramesh can log in and view Jenkins jobs, but he cannot build, configure or delete them.

### 5.3 Creating the Experienced Role

The administrator creates another role for Suresh. The role receives the permissions required for his development responsibilities.

The original demonstration gives Suresh administrator access to show the difference between restricted and full access. However, in a real environment, an experienced developer should not automatically receive `Overall/Administer`. Administrator access should be restricted to trusted Jenkins administrators.

### 5.4 Gradually Increasing Ramesh's Permissions

As Ramesh gains experience, the administrator can gradually increase his permissions.

**Initial permissions:**

- Overall/Read
- Job/Read

**After additional responsibility:**

- Job/Build

**Further responsibilities, when required:**

- Job/Create
- Job/Configure

This demonstrates how access can be expanded without creating a new role for every permission change.

If multiple users share the same role, changes to that role affect all users assigned to it.

### 5.5 Practical Scenario – Mahesh Joins the Company

**Situation:** Mahesh joins the company and needs the same Jenkins permissions as Ramesh.

**Problem:** Creating a separate role with exactly the same permissions would create unnecessary administration work.

**Resolution:**

1. Create Mahesh's user account through the configured security realm.
2. Open the role-management page.
3. Assign the existing Fresher role to Mahesh.
4. Save the configuration.
5. Log in as Mahesh and verify his access.

**Result:** Mahesh receives the same permissions as Ramesh because both users are assigned to the same role. If the administrator later modifies the Fresher role, the changes apply to both users.

**Why this is useful:** Roles make access management easier when employees have similar responsibilities. Administrators can update permissions centrally instead of configuring each user separately.

### 5.6 Recommended Permission Structure

The following is an example of a more controlled permission model.

| Permission | Fresher | Developer | Jenkins Administrator |
|---|---:|---:|---:|
| Overall/Read | Yes | Yes | Yes |
| Job/Read | Yes | Yes | Yes |
| Job/Build | No | Yes | Yes |
| Job/Configure | No | As required | Yes |
| Job/Create | No | As required | Yes |
| Job/Delete | No | Normally restricted | Yes |
| Overall/Administer | No | No | Yes |

The exact permissions should be based on responsibilities and organizational requirements.

**Security principle – Least Privilege**

Least Privilege means giving each user only the permissions required to perform their assigned work. This reduces accidental changes, unauthorized access and security risks.

For larger Jenkins environments, item roles and folder-based access restrictions can help separate development and production workloads.

---

## 6. Jenkins Service Startup – Additional Administration Note

The older command demonstrated for configuring Jenkins to start automatically is:

```bash
chkconfig jenkins on
```

Its purpose is to configure Jenkins to start automatically when the Linux server boots.

For modern systemd-based Linux installations, use `systemctl`.

```bash
# Enable Jenkins to start automatically during system boot
sudo systemctl enable jenkins

# Start Jenkins immediately
sudo systemctl start jenkins

# Check Jenkins service status
sudo systemctl status jenkins

# Restart Jenkins
sudo systemctl restart jenkins
```

To enable and start Jenkins together:

```bash
sudo systemctl enable --now jenkins
```

**Understanding:**

- `systemctl enable jenkins` configures automatic startup during boot.
- `systemctl start jenkins` starts the service immediately.
- `systemctl enable --now jenkins` performs both operations.

---

## 7. Important Technical Understanding

1. Jenkins Pipeline variables help store and reuse information during Pipeline execution.
2. Groovy variables, Pipeline environment variables and Jenkins-provided variables have different purposes and scopes.
3. Jenkins provides build information such as `BUILD_NUMBER`, `JOB_NAME` and `WORKSPACE`.
4. The `post` section allows Jenkins to perform actions after a Pipeline or stage finishes.
5. The `always`, `success` and `failure` conditions help automate actions according to execution results.
6. Runtime failures and Pipeline syntax failures are different and must be investigated separately.
7. RBAC allows administrators to assign permissions through reusable roles.
8. Authentication identifies users, while authorization controls what they can access.
9. Global, item and agent roles provide different levels of access control.
10. Reusing roles makes it easier to manage permissions for employees with similar responsibilities.
11. Least Privilege is an important security principle when managing Jenkins users.
12. Modern Linux Jenkins installations use `systemctl` for service management.

---

## 8. Important Terminology

| Term | Meaning |
|---|---|
| Variable | A named value used to store and reuse information |
| Groovy Variable | A variable declared and used in Pipeline Groovy code |
| Environment Variable | A key-value setting passed to processes |
| Pipeline-level Environment | Environment variables available across Pipeline stages |
| Stage-level Environment | Environment variables available within a particular stage |
| Jenkins Environment Variables | Values automatically provided by Jenkins |
| `BUILD_NUMBER` | Current Jenkins build number |
| `JOB_NAME` | Current Jenkins job name |
| `WORKSPACE` | Job's working directory on the executing node |
| Post Section | Declarative Pipeline section for completion actions |
| `always` | Executes regardless of the final completion status |
| `success` | Executes when the relevant Pipeline or stage succeeds |
| `failure` | Executes when the relevant Pipeline or stage fails |
| RBAC | Role-Based Access Control |
| Authentication | Identifying a user |
| Authorization | Determining what a user is allowed to access |
| Role | A named collection of permissions |
| Global Role | A role whose permissions apply globally |
| Item Role | A role restricted to matching jobs or folders |
| Agent Role | A role controlling permissions for Jenkins agents |
| Least Privilege | Giving users only the permissions required for their work |

---

## 9. Final Learning Summary

This session connects Jenkins Pipeline development with Jenkins administration and security.

Pipeline variables help manage build information and configuration values. Post-build actions automate what happens after a Pipeline completes, depending on its result. RBAC helps administrators control access when multiple employees and development teams share the same Jenkins environment.

Together, these concepts help make Jenkins Pipelines easier to maintain and Jenkins environments safer to manage.
