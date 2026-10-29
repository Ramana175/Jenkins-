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

### Part A -- Slack

1.  Go to **Slack**.
2.  Create a channel (Example: `#new-channel`).
3.  Go to **api.slack.com/apps**.
4.  Create a new app.
5.  Enable **Incoming Webhooks**.
6.  Add webhook to your channel.
7.  Copy the **Bot User OAuth Token** (`xoxb-...`).

### Part B -- Jenkins

1.  Install **Slack Notification Plugin**.
2.  Go to **Manage Jenkins → Credentials**.
3.  Add Credential.
    -   Kind: **Secret Text**
    -   Secret: Paste `xoxb-...` token.
4.  Go to **Manage Jenkins → System**.
5.  Slack section:
    -   Workspace ID
    -   Credential
    -   Default Channel: `#new-channel`
6.  Click **Test Connection**.
7.  Save.

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
