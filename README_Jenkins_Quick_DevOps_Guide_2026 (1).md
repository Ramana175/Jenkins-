# Jenkins Quick DevOps Guide (2026)

Simple notes for beginners. Only practical steps.

## 1. Crontab (Schedule Tasks)

**Syntax**

``` text
* * * * * command
- - - - -
| | | | |
| | | | + Day (0-6)
| | | + Month
| | + Day of Month
| + Hour
+ Minute
```

**Useful commands**

``` bash
crontab -l
crontab -e
crontab -r
```

**Example**

``` cron
* * * * * /home/ec2-user/hello.sh >> /home/ec2-user/hello.log
```

`>>` keeps old logs.

------------------------------------------------------------------------

## 2. Jenkins Build Triggers

### SCM Polling

-   Checks Git regularly.
-   Builds only if new commits are found.

Example:

``` text
H/5 * * * *
```

### Periodic Build

-   Runs on schedule even without code changes.

Example:

``` text
H 2 * * *
```

### GitHub Webhook (Recommended)

-   GitHub immediately tells Jenkins to build.

**Steps**

1.  Jenkins Job → Configure.
2.  Enable **GitHub hook trigger**.
3.  GitHub → Repository → Settings → Webhooks.
4.  Add:

``` text
http://YOUR-JENKINS-IP:8080/github-webhook/
```

5.  Content Type → `application/json`.
6.  Save.

------------------------------------------------------------------------

## 3. JaCoCo (Code Coverage)

Purpose: Check how much code is tested.

Target example: **80%**

Report location:

``` text
target/site/jacoco/
```

------------------------------------------------------------------------

## 4. Discard Old Builds

Job → Configure → **Discard old builds**

Set:

-   Number of builds
-   Days to keep

------------------------------------------------------------------------

## 5. Restart Jenkins

Linux:

``` bash
sudo systemctl restart jenkins
```

Browser (if enabled):

``` text
http://JENKINS-IP:8080/restart
http://JENKINS-IP:8080/safeRestart
```

**Difference**

-   Restart → Stops running jobs.
-   Safe Restart → Waits until jobs finish.

------------------------------------------------------------------------

## 6. Useful Plugins

  Plugin                Purpose
  --------------------- --------------------------
  Maven Integration     Maven jobs
  Deploy to Container   Deploy WAR
  JaCoCo                Code coverage
  Stage View            Pipeline stages
  Audit Trail           Track Jenkins activity
  Build Name Setter     Custom build names
  Next Build Number     Change next build number
  SSH Agent             SSH credentials
  ThinBackup            Jenkins backup

------------------------------------------------------------------------

## 7. Audit Trail

Steps:

1.  Install plugin.
2.  Manage Jenkins → System.
3.  Audit Trail.
4.  Log file:

``` text
/var/lib/jenkins/audit-trail.log
```

Watch logs:

``` bash
tail -f /var/lib/jenkins/audit-trail.log
```

------------------------------------------------------------------------

## 8. Build with Parameters

Steps:

1.  Job → Configure.
2.  Enable **This project is parameterized**.
3.  Add **Choice Parameter**.

Example:

``` text
BranchName
main
development
qa
```

Git branch:

``` text
*/${BranchName}
```

------------------------------------------------------------------------

## 9. Slack Notifications (2026)

### Part A — Slack Setup from the Beginning

#### Step 1: Open Slack

1. Open your browser.
2. Go to https://slack.com/
3. Click **Get Started** or **Sign Up**.
4. Create an account using your email address or sign in with an existing account.

**Why?** A Slack account is required to create and access a workspace.

#### Step 2: Create a Workspace

1. After signing in, look for the option to create a workspace. If you don't see it, open Slack's workspace creation page.
2. Click **Create a Workspace** or the available equivalent.
3. Enter your email address if requested and verify it.
4. Enter a workspace name.

   Example: `DevOps-Lab`

5. Follow the instructions to complete the workspace setup.

**What is a workspace?**

A workspace is a place where a team communicates. It contains channels, members, and messages.

Example: `DevOps-Lab` is your workspace.

#### Step 3: Create a Channel

1. Open your newly created workspace.
2. Find **Channels** in the sidebar.
3. Click the **+** icon or the option to add a channel.
4. Select **Create a channel**.
5. Enter a channel name.

   Example: `new-channel`

6. Choose **Public** or **Private**.
7. Click **Create**.
8. Add members if needed, or skip this step.

**What is a channel?**

A channel is a place inside a workspace where people discuss a particular topic.

Example: Jenkins can send build notifications to `#new-channel`.

#### Step 4: Create a Slack App

1. Open https://api.slack.com/apps
2. Sign in using the account associated with your workspace.
3. Click **Create New App**.
4. Choose **From scratch**, if prompted.
5. Enter an app name.

   Example: `Jenkins-Notifications`

6. Select your `DevOps-Lab` workspace.
7. Click **Create App**.

**What is a Slack app?**

A Slack app allows another tool, such as Jenkins, to interact with Slack.

#### Step 5: Enable Incoming Webhooks

1. Open your `Jenkins-Notifications` app settings.
2. Find **Incoming Webhooks** in the left sidebar.
3. Turn **Activate Incoming Webhooks** on.
4. Click **Add New Webhook to Workspace**.
5. Select your `#new-channel`.
6. Click **Allow**.

This creates a webhook connected to the selected channel.

**Important:** A webhook URL and a Bot User OAuth Token (`xoxb-...`) are different things. Incoming Webhooks are used to post messages through a webhook URL. The Jenkins Slack Notification Plugin commonly uses a bot token, so configure the method required by your Jenkins plugin setup.

#### Step 6: Get the Bot User OAuth Token

If you are using the Jenkins Slack Notification Plugin with a bot token:

1. Open your Slack app at https://api.slack.com/apps
2. Select `Jenkins-Notifications`.
3. Open **OAuth & Permissions**.
4. Under **Bot User OAuth Token**, copy the token beginning with `xoxb-`, if one is available.
5. Keep the token private. Do not paste it into a README, source code, or a public GitHub repository.

If the token is not shown, complete the app's required OAuth setup and installation steps first.

### Part B — Jenkins Configuration

After completing the Slack setup:

1. Install the **Slack Notification Plugin** in Jenkins.
2. Open **Manage Jenkins → Credentials**.
3. Add a credential:
   - Kind: **Secret text**
   - Secret: Your bot token (`xoxb-...`)
   - ID: For example, `slack-token`
4. Open **Manage Jenkins → System**.
5. Find the **Slack** section.
6. Enter the workspace/team information requested by your plugin version.
7. Select the credential you created.
8. Enter the default channel: `#new-channel`.
9. Click **Test Connection**.
10. If the test succeeds, click **Save**.

### Part C — Use Slack in a Freestyle Project

1. Open your Jenkins Freestyle project.
2. Click **Configure**.
3. Find the build or post-build notification options provided by the installed plugin.
4. Configure notifications for successful builds and failed builds, if those options are available.
5. Save the job and run a build.
6. Open `#new-channel` in Slack and check whether the notification arrived.

**Expected result:** Jenkins sends build notifications to the selected Slack channel.
### Freestyle Job

1.  Job → Configure.
2.  Post Build Actions.
3.  Slack Notifications.
4.  Select Success and Failure.
5.  Save.

### Pipeline

``` groovy
slackSend channel: '#new-channel',
          color: 'good',
          message: "Build Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
```

Failure:

``` groovy
slackSend channel: '#new-channel',
          color: 'danger',
          message: "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
```

------------------------------------------------------------------------

## 10. Enable or Disable Job

-   Job → **Disable Project**
-   Enable again → **Enable**

------------------------------------------------------------------------

## 11. Build Name

Install **Build Name Setter**.

Example:

``` text
my-app-#${BUILD_NUMBER}
```

------------------------------------------------------------------------

## 12. Next Build Number

Install **Next Build Number** plugin.

-   Job → Configure.
-   Set next build number.

------------------------------------------------------------------------

## 13. Jenkins Security

### User Location

Usually inside:

``` text
/var/lib/jenkins/users/
```

### Create User

Manage Jenkins → Users → Create User

### Give Limited Access

1.  Install **Role-Based Authorization Strategy**.
2.  Manage Jenkins → Security.
3.  Create roles.
4.  Assign users.

Example:

  User        Role
  ----------- -------------
  Admin       Full Access
  Developer   Build Only
  Viewer      Read Only

------------------------------------------------------------------------

## Quick Commands

``` bash
crontab -l
crontab -e
sudo systemctl restart jenkins
tail -f /var/lib/jenkins/audit-trail.log
```

## Quick Revision

-   Cron → Schedule Linux tasks
-   SCM Polling → Check Git repeatedly
-   Periodic Build → Run on schedule
-   Webhook → GitHub triggers Jenkins
-   JaCoCo → Code coverage
-   Nexus → Store artifacts
-   Tomcat → Deploy WAR
-   Slack → Build notifications
-   Safe Restart → Waits for running jobs
