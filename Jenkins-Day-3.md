# Jenkins – Day 3: Cron Jobs, Poll SCM, Webhooks, Build Throttling and Remote Triggers

## 1. Jenkins Setup

Jenkins must be installed, running and reachable before configuring build triggers.

Useful service commands on a systemd-based Linux server:

~~~bash
sudo systemctl status jenkins
sudo systemctl start jenkins
sudo systemctl restart jenkins
sudo systemctl enable jenkins
sudo journalctl -u jenkins -xe
~~~

For a standard Linux package installation, the initial administrator password is commonly stored at:

~~~bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
~~~

This path assumes the default Jenkins home directory. The actual location can differ if JENKINS_HOME has been customized.

Jenkins normally listens on TCP port 8080 unless its port has been changed. Restrict access to trusted networks. For production environments, use HTTPS and appropriate firewall or security-group rules.

## 2. Scheduling Jenkins Jobs with Build Periodically

The **Build periodically** option runs a Jenkins job automatically according to a time schedule. It is useful when a task must run even if no source-code change has occurred.

Common scenarios include:
- Running cleanup tasks every night.
- Generating daily reports.
- Performing scheduled maintenance.
- Running periodic backups or validations.

### 2.1 Cron expression format

Jenkins uses a five-field cron expression:

~~~text
MINUTE HOUR DAY_OF_MONTH MONTH DAY_OF_WEEK
~~~

| Field | Allowed values | Meaning |
|---|---:|---|
| Minute | 0–59 | Minute of the hour |
| Hour | 0–23 | Hour of the day |
| Day of month | 1–31 | Calendar day |
| Month | 1–12 | Month |
| Day of week | 0–7 | Day of the week; Sunday can be 0 or 7 |

### 2.2 Common cron operators

| Operator | Meaning | Example |
|---|---|---|
| * | Every permitted value | * in the hour field means every hour |
| */n | Every n units | */10 in the minute field means every 10 minutes |
| - | A range of values | 1-5 means Monday through Friday in the day-of-week field |
| , | A list of selected values | 1,3,5 means selected values |

### 2.3 Practical cron examples

| Cron expression | Schedule |
|---|---|
| 5 11 * * * | Every day at 11:05 AM |
| 45 16 * * * | Every day at 4:45 PM |
| 30 20 * * 0 | Every Sunday at 8:30 PM |
| */10 * * * * | Every 10 minutes |
| 0 9 * * 1-5 | Every weekday at 9:00 AM |
| 0 */2 * * * | Every 2 hours, at minute 0 |

For example, */10 * * * * runs every ten minutes. It does not mean every six minutes.

### 2.4 Jenkins-specific H notation

Jenkins supports H, a hash-based value used to spread scheduled jobs across different times.

Example:

~~~text
H/10 * * * *
~~~

This schedules the job at a stable, hash-selected offset within each ten-minute interval. It helps avoid many jobs starting at exactly the same minute and creating unnecessary load on the Jenkins controller.

Use H when an exact clock time is not required. Use a specific minute or hour when the job must run at a particular time.

### 2.5 Configure Build periodically

1. Open the Jenkins dashboard.
2. Select the required Freestyle job.
3. Choose **Configure**.
4. Find the **Build Triggers** section.
5. Enable **Build periodically**.
6. Enter the cron expression.
7. Save the job.

Jenkins schedules future builds according to the configured expression.

### 2.6 Jenkins time zone

Cron schedules are interpreted using the time zone configured for the Jenkins controller. Do not assume every Jenkins installation uses UTC.

Check the Linux server's time settings:

~~~bash
date
timedatectl
~~~

Also confirm the time zone configured for the Jenkins controller.

For example, if the controller uses UTC and a job must run at 11:00 AM India Standard Time (IST), convert the time:

~~~text
11:00 AM IST = 05:30 UTC
~~~

The corresponding cron expression on a UTC-configured controller is:

~~~text
30 5 * * *
~~~

If the controller uses IST, the expression would instead be:

~~~text
0 11 * * *
~~~

Always verify the controller's effective time zone before relying on a schedule.

## 3. Poll SCM

**Poll SCM** periodically checks a configured source-code repository for changes.

When Jenkins detects a relevant change, it schedules a build. If there are no changes, Jenkins does not start a new build.

This is useful when a team wants Jenkins to check a repository regularly without configuring a webhook.

### 3.1 Example

~~~text
H/5 * * * *
~~~

This asks Jenkins to check the repository at a hash-selected interval of approximately every five minutes.

### 3.2 Configure Poll SCM

1. Open the Jenkins job.
2. Select **Configure**.
3. Confirm the Git repository URL and branch under **Source Code Management**.
4. Under **Build Triggers**, enable **Poll SCM**.
5. Enter a polling schedule, such as H/5 * * * *.
6. Save the job.

Jenkins checks the repository according to the configured schedule and starts a build when it detects changes.

### 3.3 Build periodically vs Poll SCM

| Build periodically | Poll SCM |
|---|---|
| Runs the job according to a time schedule. | Checks the source repository according to a schedule. |
| A build can run even when no source change exists. | A build is normally scheduled only when relevant changes are detected. |
| Useful for reports, cleanup and maintenance. | Useful for checking Git changes when webhooks are not configured. |

Poll SCM is not a one-time deadline or a time window. It is a recurring repository check.

## 4. GitHub Webhook Integration with Jenkins

A GitHub webhook allows GitHub to notify Jenkins when a selected repository event occurs, such as a push.

Instead of Jenkins repeatedly checking the repository on a timer, GitHub sends an HTTP request to Jenkins when the event occurs. Jenkins processes the notification and can trigger source-control polling and schedule a build when changes are detected.

A webhook usually reduces the delay between a Git push and the start of a Jenkins build, although it does not guarantee an instantaneous build.

### 4.1 Webhook request flow

~~~text
Developer commits and pushes code
             |
             v
        GitHub repository
             |
             v
       Push event occurs
             |
             v
 GitHub sends webhook HTTP request
             |
             v
     Jenkins webhook endpoint
             |
             v
 Jenkins checks configured Git SCM
             |
             v
 Build is scheduled when changes
             are detected
~~~

### 4.2 Configure Jenkins

1. Ensure the Jenkins Git and GitHub integration plugins required by the job are installed and up to date.
2. Open the Jenkins job and select **Configure**.
3. Configure the Git repository URL and the correct branch under **Source Code Management**.
4. Under **Build Triggers**, enable **GitHub hook trigger for GITScm polling**.
5. Save the job.

### 4.3 Configure GitHub

1. Open the GitHub repository.
2. Go to **Settings → Webhooks**.
3. Select **Add webhook**.
4. Enter the Jenkins webhook endpoint:

~~~text
https://<JENKINS-DOMAIN>/github-webhook/
~~~

5. Set the content type to:

~~~text
application/json
~~~

6. Select the push event, or choose the specific events required by the workflow.
7. Save the webhook.

Replace the placeholder with the actual Jenkins domain. Jenkins must be reachable by GitHub over the network. A private Jenkins server needs an appropriate, secure connectivity arrangement; simply creating a webhook in GitHub is not enough.

Use HTTPS and configure webhook-secret validation where supported by the integration. Do not expose the Jenkins administration interface publicly just to make webhooks work.

### 4.4 Verify webhook delivery

After pushing a commit:

1. Open **Settings → Webhooks** in the GitHub repository.
2. Select the configured webhook.
3. Inspect **Recent Deliveries**.
4. Check the HTTP response and delivery details.
5. Confirm that Jenkins received the event and that the expected job was scheduled.

If the build does not start, check:
- Whether the webhook URL is correct.
- Whether the Jenkins server is reachable from GitHub.
- Whether the required Jenkins plugin is installed and enabled.
- Whether the job has the GitHub hook trigger selected.
- Whether the repository URL and branch are correct.
- Whether the push event is selected.
- Whether the webhook delivery returned an error.

A successful webhook HTTP response confirms that the endpoint handled the request; it does not by itself prove that the intended job completed successfully. Check Jenkins build history and console output as well.

### 4.5 Poll SCM vs GitHub webhook

| Poll SCM | GitHub webhook |
|---|---|
| Jenkins checks the repository repeatedly. | GitHub sends a notification when a selected event occurs. |
| Builds can be delayed until the next polling check. | Usually starts the trigger flow sooner after a push. |
| Requires a polling schedule. | Requires a reachable Jenkins webhook endpoint and GitHub configuration. |
| Useful when webhook connectivity is unavailable. | Useful when GitHub can securely reach Jenkins. |

## 5. Throttle Concurrent Builds

The **Throttle Concurrent Builds** plugin controls how many builds can run at the same time.

This is useful when several jobs or builds compete for limited CPU, memory, disk I/O or other resources. For example, a Jenkins server with limited capacity may become overloaded if many heavy builds run simultaneously.

The plugin can apply limits to concurrent builds, depending on the configured categories and options.

### 5.1 How throttling works

Suppose a job is configured to allow only two concurrent builds:

- The first build starts.
- The second build can also start.
- A third build waits in the Jenkins queue while the concurrency limit is reached.
- When one running build finishes, Jenkins can start the waiting build if capacity is available.

Throttling controls **simultaneous running builds**. It does not enforce a fixed number of builds per hour or guarantee a fixed time gap between builds.

For example, limiting a job to three concurrent builds does not mean that only three builds can run in one hour.

### 5.2 Typical configuration approach

1. Install the **Throttle Concurrent Builds** plugin.
2. Open the job configuration.
3. Locate the throttling options provided by the plugin.
4. Configure the required category or job-level limits.
5. Set the maximum concurrent builds according to available resources.
6. Save and observe the build queue and running builds.

The exact options depend on the installed plugin version. Choose limits based on the workload and capacity of the Jenkins controller or agents.

## 6. Trigger a Jenkins Build Remotely

A Jenkins job can be triggered by an external script or system. This is useful when another application, deployment process or automation tool needs to start a Jenkins build.

A Freestyle job provides a **Trigger builds remotely (e.g., from scripts)** option. It allows a configured trigger token to be used with the job's build endpoint.

A token-based URL may look like this:

~~~text
https://<JENKINS-DOMAIN>/job/<JOB-NAME>/build?token=<TOKEN>
~~~

This is useful for understanding the remote-trigger concept, but a token in a URL should not be treated as a replacement for proper authentication and authorization. URLs can be recorded in logs, browser history and monitoring systems.

### 6.1 Use the Jenkins Remote Access API

For scripts and automation, use a Jenkins account with the required job permissions and an API token.

Example for a job without parameters:

~~~bash
curl -X POST \
  --user "jenkins-user:YOUR_API_TOKEN" \
  "https://jenkins.example.com/job/job1/build"
~~~

Replace the username, API token, Jenkins URL and job path with the actual values.

### 6.2 Trigger a parameterized job

If the job accepts parameters, use the buildWithParameters endpoint.

~~~bash
curl -X POST \
  --user "jenkins-user:YOUR_API_TOKEN" \
  --data "ENVIRONMENT=dev" \
  "https://jenkins.example.com/job/job1/buildWithParameters"
~~~

The parameter name must match the parameter configured in the Jenkins job.

### 6.3 Security considerations

- Use HTTPS.
- Use a dedicated Jenkins account with only the permissions required to trigger the job.
- Use an API token instead of a user's password.
- Store the token in a secure secret store or protected CI/CD credential.
- Do not commit tokens to Git or expose them in scripts, logs or public URLs.
- Confirm that the Jenkins user has permission to build the target job.

Jenkins API tokens are intended for API authentication and are exempt from the normal CSRF crumb requirement. Other authentication methods and endpoints may have different CSRF requirements.

## 7. Jenkins Workspace

A Jenkins workspace is the working directory used by a job or build agent.

It is where Jenkins checks out source code and runs build steps. Build tools such as Maven can create their output inside this directory.

For a standard Linux package installation using the default Jenkins home directory, a Freestyle job's workspace commonly looks like this:

~~~text
/var/lib/jenkins/workspace/<JOB-NAME>/
~~~

For example:

~~~text
/var/lib/jenkins/workspace/Netflix-CI-Job/
~~~

A Maven project may generate its build output under:

~~~text
/var/lib/jenkins/workspace/Netflix-CI-Job/target/
~~~

The actual workspace location can differ when:
- JENKINS_HOME is customized.
- A job uses a custom workspace.
- The build runs on a Jenkins agent rather than the controller.
- The agent has a different operating system or workspace configuration.

The workspace is a working directory, not permanent artifact storage. Workspaces can be cleaned or reused. Archive important artifacts in Jenkins or publish them to an artifact repository such as Nexus Repository or JFrog Artifactory when they must be retained and shared with later pipeline stages.

## 8. Choosing the Right Build Trigger

| Requirement | Suitable trigger |
|---|---|
| Run a report every morning | Build periodically |
| Run a cleanup task every night | Build periodically |
| Check a repository every few minutes | Poll SCM |
| Start a build after a GitHub push | GitHub webhook |
| Start a job from an external script | Remote build trigger / Jenkins API |

A project can use different trigger strategies for different jobs. For example, a CI job can use a GitHub webhook while a separate maintenance job runs on a nightly schedule.

## 9. Troubleshooting Checklist

### Scheduled build does not run

- Confirm that **Build periodically** is enabled.
- Check the cron expression and its five fields.
- Verify the Jenkins controller's time zone.
- Check the Jenkins system log and build queue.
- Confirm that Jenkins is running.

### Poll SCM does not detect a change

- Verify the repository URL and branch.
- Confirm that Jenkins has access to the repository.
- Check the Poll SCM schedule.
- Confirm that the commit changes a branch or path monitored by the job.
- Review the job's polling and build logs.

### GitHub webhook does not trigger a build

- Inspect the webhook's Recent Deliveries in GitHub.
- Confirm that the Jenkins endpoint is reachable over HTTPS.
- Verify the webhook URL and selected event.
- Confirm the required plugin and job trigger configuration.
- Check Jenkins logs and build history.

### Builds remain in the queue

- Check the job's concurrency settings.
- Review the Throttle Concurrent Builds plugin configuration.
- Check whether executors are available on the controller or agents.
- Review CPU, memory and disk capacity.
- Check whether another job or category is applying a concurrency limit.

### Remote build request fails

- Verify the Jenkins URL and job path.
- Confirm that the Jenkins account and API token are valid.
- Ensure that the account has permission to build the job.
- Check whether the job requires parameters.
- Use HTTPS and inspect the HTTP response returned by the request.

## 10. Key Takeaways

- **Build periodically** runs a Jenkins job according to a cron schedule, even when source code has not changed.
- A cron expression uses five fields: minute, hour, day of month, month and day of week.
- Jenkins cron schedules use the controller's configured time zone.
- **Poll SCM** periodically checks a source repository and schedules a build when changes are detected.
- A **GitHub webhook** sends an event notification to Jenkins after a selected repository event, such as a push.
- **Throttle Concurrent Builds** limits simultaneous running builds; it does not limit the number of builds per hour.
- A **remote build trigger** allows an external script or system to start a Jenkins job.
- The **workspace** is a working area for source code and build execution, not a reliable long-term artifact repository.
- Secure Jenkins integrations with HTTPS, restricted permissions and protected credentials.

## 11. Interview Preparation

**1. What is the difference between Build periodically and Poll SCM?**

Build periodically runs a job according to a schedule regardless of source-code changes. Poll SCM checks the repository on a schedule and normally triggers a build only when changes are detected.

**2. What is the cron expression format used by Jenkins?**

It uses five fields: minute, hour, day of month, month and day of week.

**3. What does */10 * * * * mean?**

It means the job is scheduled every ten minutes.

**4. What is the purpose of H in a Jenkins cron expression?**

H represents a stable hash-based value that helps distribute job schedules and avoid many jobs starting at the same exact time.

**5. Which time zone does Jenkins use for cron schedules?**

The schedule uses the Jenkins controller's configured time zone. It should be checked rather than assumed to be UTC.

**6. What is a GitHub webhook?**

It is an HTTP notification sent by GitHub to a configured endpoint when a selected repository event occurs.

**7. Why might a GitHub webhook fail to trigger a Jenkins build?**

Possible causes include an incorrect webhook URL, network connectivity problems, missing plugins, an unconfigured job trigger, an incorrect repository or branch, or an unselected event.

**8. What does Throttle Concurrent Builds do?**

It limits how many builds can run simultaneously. Builds that exceed the configured limit wait in the queue until capacity becomes available.

**9. Does throttling three concurrent builds mean only three builds can run per hour?**

No. It limits simultaneous builds, not the total number of builds per hour.

**10. How can an external script trigger a Jenkins job?**

It can use the Jenkins Remote Access API with a Jenkins account and API token, provided the account has permission to build the job.

**11. What is a Jenkins workspace?**

It is the working directory where Jenkins checks out source code and executes the job's build steps.

**12. Should important build artifacts be stored only in the workspace?**

No. Workspaces can be cleaned or reused. Important artifacts should be archived or published to an artifact repository.
