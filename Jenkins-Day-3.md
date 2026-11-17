# Jenkins – Day 3: Cron Jobs, Poll SCM, Webhooks, Build Throttling and Remote Triggers

## 1. Jenkins Setup Using a Script

A shell script can automate Jenkins installation and service setup on an EC2 Linux instance. Review the script and confirm that its commands match the operating system and Jenkins release before running it.

~~~bash
sudo su
vi jenkins.sh
sh jenkins.sh
~~~

For a standard Linux package installation, the initial administrator password is commonly available at:

~~~bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
~~~

Access Jenkins at http://<EC2-Public-IP>:8080 when port 8080 is configured and reachable. The password path may differ if JENKINS_HOME is customized. Restrict access to trusted networks and use HTTPS in production.

## 2. Cron Job – Scheduling Jenkins Builds

A cron schedule runs a Jenkins job at a configured time without someone manually selecting **Build Now**.

### 2.1 Practical scenario: Generate a WAR file while away

Imagine tomorrow is Saturday and you have planned to watch a movie. Your company asks you to generate a WAR file from the application at 11:00 AM. You do not need to sit in front of your laptop and click **Build Now** at that time. Configure Jenkins in advance, and Jenkins will execute the configured build steps according to the schedule.

### 2.2 Five-field cron syntax

Jenkins uses five fields:

~~~text
MINUTE HOUR DAY_OF_MONTH MONTH DAY_OF_WEEK
  *     *       *         *       *
~~~

| Position | Field | Values | Example |
|---|---|---|---|
| 1 | Minute | 0–59 | 5, 15, 30, 45 |
| 2 | Hour | 0–23 | 0 = midnight, 11 = 11 AM, 16 = 4 PM, 20 = 8 PM |
| 3 | Day of month | 1–31 | 8, 10, 25 |
| 4 | Month | 1–12 | 1 = January, 9 = September, 12 = December |
| 5 | Day of week | 0–7 | 0 or 7 = Sunday, 1 = Monday, ..., 6 = Saturday |

Read the expression from left to right. Spaces separate the five fields.

### 2.3 Examples

**11:05 AM every day**

~~~text
5 11 * * *
~~~

**4:45 PM every day**

~~~text
45 16 * * *
~~~

**8:30 PM every Sunday**

~~~text
30 20 * * 0
~~~

**9:00 AM Monday to Friday**

~~~text
0 9 * * 1-5
~~~

Cron uses a 24-hour clock. Sunday can be represented by 0 or 7.

### 2.4 Cron operators

| Operator | Meaning | Example |
|---|---|---|
| * | Every permitted value | * * * * * = every minute |
| */n | Every n units | */10 * * * * = every 10 minutes |
| - | Range | 1-5 = Monday through Friday |
| , | Selected values | 1,3,5 = selected days |

**Correction:** */10 in the minute field means every 10 minutes (minutes 0, 10, 20, 30, 40 and 50). It does not mean every 6 minutes.

Jenkins also supports H, a hash-based value that spreads jobs across different times:

~~~text
H/10 * * * *
~~~

This uses a stable, hash-selected offset within each ten-minute interval and helps prevent many jobs from starting at the same minute.

### 2.5 Time zone: IST and UTC

Cron expressions are interpreted using the Jenkins controller's configured time zone. Do not assume every Jenkins server always uses UTC.

Check the Linux server time:

~~~bash
date
timedatectl
~~~

Also confirm the Jenkins controller's effective time zone.

If the controller uses UTC and the required schedule is 11:05 AM IST:

~~~text
11:05 AM IST = 05:35 UTC
~~~

Use this expression on a UTC-configured controller:

~~~text
35 5 * * *
~~~

If the controller uses IST, use:

~~~text
5 11 * * *
~~~

Always calculate the cron time using the controller's configured time zone.

### 2.6 Configure a scheduled Freestyle job

Use a Java application configured to produce a WAR file.

1. Jenkins dashboard → **New Item**.
2. Name the job **cronjob**.
3. Select **Freestyle project** → **OK**.
4. Under **Source Code Management**, select **Git**.
5. Enter the repository URL, for example https://github.com/devopsraham/jenkins-java-project, and configure the correct branch.
6. Under **Build Steps**, choose **Execute shell** and enter the Maven command below, or choose **Invoke top-level Maven targets** and enter the Maven goals below.

~~~bash
mvn clean package
~~~

7. Under **Build Triggers**, select **Build periodically**.
8. Enter the cron expression and save.

For a quick test, choose a time shortly in the future using the controller's time zone. After that time, check the build history and **Console Output**. The build should run the configured Maven goals and produce the artifact defined by the project, such as a WAR file.

### 2.7 Use cases

- Generate a WAR or JAR file at a planned time.
- Run nightly maintenance or cleanup.
- Generate scheduled reports.
- Perform periodic validation.
- Run jobs that do not depend on a new Git commit.

## 3. Limitation of Cron Jobs – Poll SCM

### 3.1 Why a scheduled build can waste resources

A cron job runs at the configured time whether or not the application source code has changed.

**Scenario:** A developer says the code will be updated before 11:00 AM tomorrow. You schedule Jenkins to create a WAR file at 11:00 AM. The developer does not make the change. Jenkins still runs the build at 11:00 AM, potentially packaging the same source code again. This consumes build time and server resources without validating a new change.

### 3.2 Solution: Poll SCM

**Poll SCM** periodically checks the configured source-code repository. If Jenkins detects relevant changes since the previous check, it schedules a build. If no relevant changes are detected, it does not schedule a build for that check.

Poll SCM is a recurring repository check, not a one-time deadline or a time window that ends at a specified time.

~~~text
Developer changes code and pushes to GitHub
                  |
                  v
          Next Poll SCM check
                  |
                  v
        Jenkins checks repository
                  |
          +-------+-------+
          |               |
      Changes           No changes
      detected          detected
          |               |
          v               v
    Build scheduled    No new build
~~~

### 3.3 Configure Poll SCM

1. Create a Freestyle job named **pollscm-project**.
2. Configure the Git repository URL and correct branch.
3. Configure the Maven build step, such as the goals **clean package**.
4. Under **Build Triggers**, select **Poll SCM**.
5. Enter a schedule, for example:

~~~text
H/5 * * * *
~~~

6. Save the job.

This checks the repository at a hash-selected interval of approximately every five minutes.

### 3.4 Practical tests

Use an application file such as **webapp/index.jsp**.

**Test 1 – Code changed:** Edit the displayed application version from 1.0 to 2.0, commit and push the change. At the next polling check, Jenkins should detect the new commit and schedule a build.

**Test 2 – No code change:** Do not make a new commit. At the next polling check, Jenkins should find no relevant change and should not schedule a build for that check.

**Test 3 – Code changed again:** Change the version from 2.0 to 3.0, commit and push. At the next polling check, Jenkins should detect the change and schedule another build.

### 3.5 Build periodically vs Poll SCM

| Build periodically | Poll SCM |
|---|---|
| Runs according to a time schedule. | Checks the repository according to a schedule. |
| Runs even if no source-code change exists. | Schedules a build when relevant changes are detected. |
| Useful for reports, cleanup and maintenance. | Useful when builds should depend on source changes. |

Poll SCM avoids builds when there is no relevant change, but a commit can still wait until the next polling interval.

## 4. Limitation of Poll SCM – GitHub Webhooks

### 4.1 Why Poll SCM can delay a build

**Scenario:** A developer pushes code at 7:00 AM, but the next Poll SCM check is scheduled for 9:00 AM. Jenkins may not detect the change until 9:00 AM. The build is delayed by the polling interval.

### 4.2 Solution: GitHub webhook

A webhook allows GitHub to send an HTTP notification to Jenkins when a configured repository event occurs, such as a push.

Instead of waiting for the next scheduled repository check, GitHub notifies Jenkins when the push event happens. Jenkins processes the event and can trigger source-control polling and schedule a build when relevant changes are detected.

A webhook normally reduces the delay, but it does not guarantee that a build starts at the exact instant of a push. Network delivery, Jenkins processing and queue availability can introduce delay.

- **Poll SCM:** Jenkins checks GitHub at configured intervals.
- **Webhook:** GitHub sends an event notification to Jenkins after a selected event.

A webhook is generally preferable when GitHub can securely reach Jenkins. Poll SCM remains useful when webhook connectivity is unavailable.

### 4.3 Webhook request flow

~~~text
Developer commits and pushes code
                  |
                  v
           GitHub push event
                  |
                  v
      GitHub sends HTTP notification
                  |
                  v
         Jenkins webhook endpoint
                  |
                  v
       Jenkins checks configured Git SCM
                  |
                  v
      Build scheduled if changes detected
~~~

### 4.4 Part A – Configure GitHub

1. Open the repository where you have permission to configure webhooks.
2. Go to **Settings → Webhooks → Add webhook**.
3. Set the Payload URL to:

~~~text
https://<JENKINS-DOMAIN>/github-webhook/
~~~

4. Set Content type to **application/json**.
5. Select the **push** event.
6. Save the webhook.

The endpoint includes the hyphen in github-webhook and the trailing slash. Replace the placeholder with the actual Jenkins domain.

A fork is not inherently required. Appropriate repository permissions are required to configure a webhook. Fork a repository when you need your own copy or write access to a project you do not control; forking is not a universal webhook requirement.

Jenkins must be reachable from GitHub. A private Jenkins instance needs a secure network design that allows webhook delivery. Use HTTPS and configure webhook-secret validation where supported. Do not expose the Jenkins administration interface publicly just to receive webhook events.

### 4.5 Part B – Configure the Jenkins Freestyle job

1. Jenkins dashboard → **New Item**.
2. Name the job **webhook**.
3. Select **Freestyle project** → **OK**.
4. Under **Source Code Management**, select **Git**.
5. Enter the repository URL and correct branch.
6. Under **Build Steps**, select **Invoke top-level Maven targets**.
7. Enter **clean package** in Goals.
8. Under **Build Triggers**, select **GitHub hook trigger for GITScm polling**.
9. Save the job.

The Git repository configuration tells Jenkins where the source code is located. The GitHub hook trigger enables the job to react to the webhook notification. Both parts must be configured.

### 4.6 Test the webhook with application changes

Use **webapp/index.jsp** as the sample application file.

1. Note the current build number of the Jenkins **webhook** job.
2. Open the GitHub repository and navigate to **webapp/index.jsp**.
3. Change the displayed application version to **3.0** and commit.
4. Check the Jenkins job's build history.
5. Repeat with versions **4.0** and **5.0**, committing each change separately.

Each push should send a webhook event. If Jenkins receives and processes the event and detects the relevant change, another build should be scheduled without manually selecting **Build Now**.

Check **Console Output** to confirm whether Maven completed successfully. A new build number alone does not prove that the build passed.

### 4.7 Webhook troubleshooting

- Verify that the payload URL ends in **/github-webhook/**.
- Confirm the content type is **application/json**.
- Confirm the push event is selected.
- Ensure Jenkins is reachable from GitHub.
- Verify the required Jenkins integration plugin is installed and enabled.
- Confirm **GitHub hook trigger for GITScm polling** is enabled in the job.
- Verify the repository URL and branch.
- Inspect GitHub **Recent Deliveries**.
- Review Jenkins logs and build history.

A successful HTTP delivery does not prove that the application build passed. Check the Jenkins build result and console output separately.

## 5. Build Throttling – Controlling How Frequently Jobs Can Run

### 5.1 Practical scenario: Too many Build Now clicks

Imagine that you accidentally select **Build Now** eight times for the same Jenkins job. Multiple build requests may be queued or executed, depending on the job's concurrency configuration and available executors.

If several resource-intensive builds run together on a small Jenkins server, CPU, memory and disk usage can increase significantly. Jenkins may become slow or unstable.

A build-rate limit can help prevent repeated requests from starting too frequently.

### 5.2 Understand the demonstrated settings

The configuration described in the practical example contains fields such as:

- **Maximum Total Builds**
- **Time Period**, such as an hour

A setting of three builds per hour describes a **build-rate limit**: no more than the configured number of builds are allowed during the configured period.

Do not confuse this with the **Throttle Concurrent Builds** plugin's concurrency controls. A concurrency limit controls the number of builds running at the same time. A rate limit controls how many builds can be started during a period. They solve different problems.

The exact plugin or extension providing a **Maximum Total Builds / Time Period** interface must be identified from the installed Jenkins plugin. Do not assume this interface is supplied by the Throttle Concurrent Builds plugin.

### 5.3 Understand a three-build-per-hour limit

Suppose a rate-limiting configuration allows a maximum of three builds in an hour.

- A build request is accepted while capacity is available.
- Further requests may be held or rejected when the configured limit is reached.
- Another request becomes eligible when the relevant time window has enough capacity again.

This means three builds in the configured hour window. It does **not** automatically guarantee a fixed 20-minute gap between every build. The exact release time depends on how the installed plugin defines and evaluates its time window.

### 5.4 Configuration approach

If the installed plugin provides the rate-limit controls shown in the job configuration:

1. Create a Freestyle job, for example **job1**.
2. Open **Configure**.
3. Locate the plugin's build-throttling or rate-limiting section.
4. Set **Maximum Total Builds** to 3.
5. Set **Time Period** to hour.
6. Save the job.
7. Trigger builds and observe whether additional requests are queued, delayed or rejected according to that plugin's behavior.

If these fields are not visible, identify and install the plugin that provides that specific rate-limiting feature. Do not substitute a concurrency plugin and expect it to enforce a per-hour quota.

### 5.5 Concurrent-build throttling is different

The **Throttle Concurrent Builds** plugin controls how many builds can run simultaneously. For example, a maximum of two concurrent builds allows two builds to run at once; additional builds wait for an available slot.

| Build-rate limit | Concurrent-build limit |
|---|---|
| Controls how many builds can start within a time period. | Controls how many builds can run at the same time. |
| Example: maximum three builds per hour. | Example: maximum two running builds. |
| Helps prevent excessive build frequency. | Helps prevent resource overload from parallel builds. |

Choose the control that matches the problem. A Jenkins environment may need both rate controls and concurrency controls, depending on workload and plugin capabilities.

## 6. Trigger Jenkins Builds Remotely – Using a URL or Script

### 6.1 Practical scenario: Trigger a build away from the dashboard

Imagine that you are away from your laptop or do not want to open the Jenkins dashboard, but you need to start a job from another system.

A remote build trigger allows an external script or system to send a request to Jenkins to start a job.

### 6.2 Configure the Freestyle job

1. Open the **job1** Freestyle job.
2. Select **Configure**.
3. Open **Build Triggers**.
4. Enable **Trigger builds remotely (e.g., from scripts)** if this option is available in the job configuration.
5. Enter a strong, unique trigger token.
6. Save the job.

A token-based endpoint may have this form:

~~~text
https://<JENKINS-DOMAIN>/job/<JOB-NAME>/build?token=<TOKEN>
~~~

For a job named job1:

~~~text
https://jenkins.example.com/job/job1/build?token=<TOKEN>
~~~

The exact endpoint and authentication behavior depend on the Jenkins version and job configuration. Use the URL displayed by Jenkins rather than assuming every installation generates the same URL.

### 6.3 Test the remote trigger

1. Send the request from an authorized browser or script.
2. Open Jenkins and check the job's build history.
3. Confirm that a new build number appears.
4. Open **Console Output** to check the build result.

If the previous build number was 9 and a new request is accepted, the next build may appear as Build #10. A further accepted request may create Build #11.

A successful trigger request means Jenkins accepted the request; it does not mean the build itself succeeded.

### 6.4 Use the Jenkins Remote Access API

For automation, prefer an authenticated API request using a Jenkins account with the required build permission and an API token.

For a job without parameters:

~~~bash
curl -X POST \
  --user "jenkins-user:YOUR_API_TOKEN" \
  "https://jenkins.example.com/job/job1/build"
~~~

For a parameterized job:

~~~bash
curl -X POST \
  --user "jenkins-user:YOUR_API_TOKEN" \
  --data "ENVIRONMENT=dev" \
  "https://jenkins.example.com/job/job1/buildWithParameters"
~~~

The parameter name must match the parameter configured in the job.

### 6.5 Security considerations

- Use HTTPS.
- Use a dedicated Jenkins account with only the required permissions.
- Prefer an API token for script-based authentication rather than a user's password.
- Store tokens in a secure secret store or Jenkins Credentials.
- Do not commit tokens to Git or expose them in public URLs, scripts or logs.
- Do not assume that a token in a URL removes the need for proper authentication and authorization.
- Check the HTTP response and Jenkins build history when troubleshooting.

Jenkins API tokens are intended for API authentication and are exempt from the normal CSRF crumb requirement. Other authentication methods and endpoints may have different CSRF requirements.

## 7. Jenkins Default Workspace Path

A Jenkins workspace is the working directory where Jenkins checks out application source code and executes the configured build steps.

For a standard Linux package installation using the default Jenkins home directory, a job workspace commonly looks like:

~~~text
/var/lib/jenkins/workspace/<JOB-NAME>/
~~~

For a job named job1:

~~~bash
cd /var/lib/jenkins/workspace/job1
ls
ls target/
~~~

For a job named Netflix:

~~~text
/var/lib/jenkins/workspace/Netflix/
~~~

If the Maven project is configured to package a WAR file, the generated artifact commonly appears in the Maven target directory:

~~~text
/var/lib/jenkins/workspace/job1/target/
~~~

The exact workspace path can differ when JENKINS_HOME is customized, a custom workspace is configured, or the build runs on a separate Jenkins agent.

The workspace is a working directory, not permanent artifact storage. Workspaces can be cleaned or reused. Archive important artifacts in Jenkins or publish them to an artifact repository such as Nexus Repository or JFrog Artifactory when they must be retained.

## 8. Choosing the Right Jenkins Build Automation

| Concept | Purpose | Configuration | Main benefit |
|---|---|---|---|
| Cron / Build periodically | Run a job at scheduled times | **Build Triggers → Build periodically** | Automates time-based work |
| Poll SCM | Check for repository changes at intervals | **Build Triggers → Poll SCM** | Avoids builds when no relevant source change is detected |
| GitHub webhook | Notify Jenkins about repository events | GitHub webhook plus **GitHub hook trigger for GITScm polling** | Reduces the wait for the next polling interval |
| Build-rate limit | Limit how frequently builds are accepted | Plugin-specific maximum builds and time-period settings | Controls excessive build requests |
| Concurrent-build throttle | Limit builds running simultaneously | Throttle Concurrent Builds plugin configuration | Controls parallel resource usage |
| Remote build trigger | Start a job from an external system | Remote trigger or Jenkins API | Allows authorized external automation |

These mechanisms reduce the need for manual **Build Now** actions, but they serve different purposes. Select the trigger or limit that matches the job's actual requirement.

## 9. Troubleshooting Checklist

### Scheduled job does not run

- Confirm **Build periodically** is enabled.
- Check the five cron fields and their order.
- Verify the Jenkins controller's time zone.
- Confirm Jenkins is running and has available build capacity.
- Review the job's build history and Jenkins logs.

### Poll SCM does not start a build

- Verify the Git repository URL and branch.
- Confirm Jenkins can access the repository.
- Check the polling schedule.
- Confirm that a relevant commit was pushed to the branch being monitored.
- Review polling logs and build history.

### GitHub webhook does not trigger a job

- Verify the **/github-webhook/** endpoint.
- Confirm **application/json** and the push event selection.
- Check GitHub **Recent Deliveries**.
- Confirm Jenkins is reachable from GitHub.
- Verify the Jenkins integration plugin and job trigger setting.
- Review Jenkins logs and the job's build history.

### Build requests are delayed or held

- Identify whether the configured control is a build-rate limit or a concurrent-build limit.
- Check the plugin name and its configuration.
- Review the job queue and available executors.
- Check CPU, memory and disk usage.
- Confirm whether another job or category applies additional limits.

### Remote build request fails

- Verify the Jenkins URL and job path.
- Confirm the account, token and build permission.
- Check whether the job requires parameters.
- Use HTTPS.
- Inspect the HTTP response and Jenkins logs.

### WAR file is not found in the workspace

- Confirm that the Maven build completed successfully.
- Check the Maven packaging type in pom.xml.
- Inspect the job's actual workspace.
- Check the target directory.
- If the job runs on an agent, inspect the agent workspace rather than assuming the controller path.

## 10. Key Takeaways

- Cron scheduling runs a Jenkins job at configured times without manual intervention.
- Jenkins cron expressions contain five fields: minute, hour, day of month, month and day of week.
- The controller's configured time zone determines how scheduled times are interpreted.
- Build periodically runs according to time; it does not check whether source code changed.
- Poll SCM checks the repository periodically and schedules a build when relevant changes are detected.
- A GitHub webhook notifies Jenkins about a repository event, reducing the wait associated with periodic polling.
- A build-rate limit and a concurrent-build limit are different controls.
- A remote build trigger allows an authorized external script or system to start a Jenkins job.
- The Jenkins workspace is where source code and build steps are executed; it should not be treated as permanent artifact storage.

## 11. Interview Preparation

**1. What is a cron job in Jenkins?**

It is a scheduled trigger that runs a Jenkins job at configured times without requiring someone to select Build Now.

**2. How many fields are present in a Jenkins cron expression?**

Five: minute, hour, day of month, month and day of week.

**3. What does */10 * * * * mean?**

It schedules the job every ten minutes.

**4. What is the difference between Build periodically and Poll SCM?**

Build periodically runs according to a time schedule regardless of source changes. Poll SCM checks the repository periodically and schedules a build when relevant changes are detected.

**5. What is the limitation of Poll SCM?**

Jenkins may not detect a new commit until the next scheduled repository check, which can delay the build.

**6. What is a GitHub webhook?**

It is an HTTP notification sent by GitHub to a configured endpoint when a selected repository event, such as a push, occurs.

**7. Why must the webhook be configured in both GitHub and Jenkins?**

GitHub must know where to send the event, and Jenkins must be configured to receive and process the event for the relevant job.

**8. Is forking a repository mandatory for a GitHub webhook?**

No. Appropriate permissions are required to configure a webhook. Forking is useful when a separate copy or write access is needed, but it is not a universal webhook requirement.

**9. What is the difference between a build-rate limit and a concurrent-build limit?**

A rate limit controls how many builds can be accepted during a time period. A concurrency limit controls how many builds can run at the same time.

**10. Does a three-build-per-hour limit guarantee a 20-minute gap between builds?**

No. It limits the number of builds within the configured time window. The exact timing depends on how the plugin implements that window.

**11. How can an external script trigger a Jenkins job?**

It can send an authenticated request to the Jenkins Remote Access API using an account with build permission and an API token.

**12. What is the default Linux Jenkins workspace path?**

For a standard package installation with the default Jenkins home directory, it commonly starts at /var/lib/jenkins/workspace/<JOB-NAME>/. Custom workspaces and agents can use different paths.

**13. Where is a Maven WAR artifact commonly generated?**

In the Maven project's target directory inside the job's workspace, provided the project is configured to package a WAR file and the build succeeds.
