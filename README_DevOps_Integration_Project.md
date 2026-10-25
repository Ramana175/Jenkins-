# DevOps Integration Project: GitHub, Jenkins, Maven, SonarQube, Nexus, and Tomcat

A practical guide to building and deploying a Java web application using a simple DevOps workflow.

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [Workflow and Lab Environment](#2-workflow-and-lab-environment)
3. [Why Each Tool Is Connected](#3-why-each-tool-is-connected)
4. [GitHub](#4-github)
5. [Jenkins and Maven](#5-jenkins-and-maven)
6. [SonarQube](#6-sonarqube)
7. [Nexus Repository Manager](#7-nexus-repository-manager)
8. [Tomcat Deployment](#8-tomcat-deployment)
9. [Final Jenkins Job Configuration](#9-final-jenkins-job-configuration)
10. [End-to-End Verification](#10-end-to-end-verification)
11. [Troubleshooting](#11-troubleshooting)
12. [Command Cheat Sheet](#12-command-cheat-sheet)
13. [Security and Cleanup](#13-security-and-cleanup)

---

## 1. Project Overview

The goal is to take a Java web application from source code to a running application.

| Tool | Purpose |
|---|---|
| GitHub | Store and version-control source code |
| Jenkins | Automate checkout, build, analysis, and deployment |
| Maven | Compile, test, package, and optionally publish the project |
| SonarQube | Analyze source-code quality |
| Nexus Repository Manager | Proxy dependencies and store artifacts |
| Apache Tomcat | Run the deployed Java web application |

The web module used in the practical is `tomcat-web-application`. The WAR artifact was `tomcat-web-application-1.0.0.war`.

## 2. Workflow and Lab Environment

```text
Developer -> GitHub -> Jenkins + Maven -> SonarQube
                                |              |
                                |              v
                                |        Quality analysis
                                v
                       Nexus (if configured)
                                |
                                v
                         WAR deployed to Tomcat
                                |
                                v
                         Application in browser
```

| Component | Practical details |
|---|---|
| GitHub repository | `https://github.com/Ramana175/Maven` |
| Jenkins job | `DevOps-Integration-Project` |
| Jenkins Maven tool name | `Maven-3.10.0` |
| SonarQube | Community Build, port `9000` |
| Nexus | `3.97.1-02`, port `8081` |
| Tomcat | `10.1.61`, port `8080` |

Use current EC2 public IP addresses. Public IPs can change after an instance is stopped and started.

## 3. Why Each Tool Is Connected

The tools are connected because each one solves a different part of the delivery process. They are not interchangeable.

1. **GitHub — source-code home:** developers commit and push the Java application and `pom.xml`. Jenkins retrieves the selected branch from GitHub.
2. **Jenkins — automation controller:** Jenkins runs the pipeline/job when requested or triggered. It checks out the source, invokes Maven, reads credentials securely, and coordinates analysis and deployment.
3. **Maven — build and dependency tool:** Maven reads `pom.xml`, downloads required libraries, compiles/tests the code, and packages the web module as a WAR. Maven uses `settings.xml` for machine/user-level repository routing and credentials.
4. **SonarQube — code analysis:** the scanner sends source-code analysis results to SonarQube so you can review bugs, vulnerabilities, code smells, and Quality Gate status. SonarQube does not build or host the WAR.
5. **Nexus — artifact repository:** Maven can download dependencies through Nexus and, when publishing is configured, upload release/snapshot artifacts to Nexus. A Nexus proxy/group repository for downloads and hosted repositories for uploads serve different purposes.
6. **Tomcat — application runtime:** Jenkins sends the built WAR to Tomcat. Tomcat runs the Java web application and serves its HTTP endpoints.

Typical sequence for this lab:

```text
Developer -> GitHub -> Jenkins checks out source
                         |
                         v
                   Maven clean/package
                    /      |       \
          Nexus download  SonarQube   WAR file
          dependencies    analysis      |
                                        v
                              Nexus upload (if configured)
                                        |
                                        v
                               Jenkins deploys WAR
                                        |
                                        v
                                     Tomcat
```

The exact execution order depends on the configured Jenkins build steps. In this practical, the Maven goals were `clean package deploy` followed by the Sonar Maven goal in the same goals field; for clearer diagnostics, teams often separate build/publish, analysis, and deployment into distinct stages. **A successful build, a successful Sonar scanner run, an uploaded artifact, and a working Tomcat application are four different checks.** Verify each one independently.

---

## 4. GitHub

The Jenkins job checks out this repository:

```text
https://github.com/Ramana175/Maven.git
```

From the local project directory in PowerShell:

```powershell
git status
git add .
git commit -m "Update DevOps project"
git push
```

If there is nothing to commit, there may be no new changes. If `git push` is rejected because the remote contains commits that are not local, inspect `git status`, branches, and remote history before resolving the merge. Avoid force-pushing without understanding what would be overwritten.

Confirm the latest source code and `pom.xml` are visible in GitHub before running Jenkins.

## 5. Jenkins and Maven

### 5.1 Jenkins service commands

On the Jenkins Ubuntu EC2 server:

```bash
sudo systemctl status jenkins
sudo systemctl start jenkins
sudo systemctl stop jenkins
sudo systemctl restart jenkins
sudo systemctl enable jenkins
sudo journalctl -u jenkins -n 50 --no-pager
sudo journalctl -u jenkins -f
sudo ss -lntp | grep ':8080'
```

Jenkins URL:

```text
http://<JENKINS_PUBLIC_IP>:8080
```

Allow inbound TCP `8080` only from intended client IPs in the EC2 security group.

### 5.2 Maven installation in Jenkins

Maven was configured through the Jenkins UI. The configured installation name is `Maven-3.10.0`.

Verified Maven home on the **Jenkins server**:

```text
/var/lib/jenkins/tools/hudson.tasks.Maven_MavenInstallation/Maven-3.10.0
```

Verify it:

```bash
sudo -u jenkins /var/lib/jenkins/tools/hudson.tasks.Maven_MavenInstallation/Maven-3.10.0/bin/mvn -version
```

The practical reported Maven `3.10.0`, Java `21.0.12.1`, and Java runtime `/usr/lib/jvm/java-21-openjdk-amd64`.

**Important path distinction:** `/opt/apache-maven-3.10.0` belongs to the separate Maven client EC2 lab, not the Jenkins-managed Maven installation.

### 5.3 Jenkins workspace

The practical workspace was:

```text
/var/lib/jenkins/workspace/DevOps-Integration-Project/
```

Inspect it with:

```bash
sudo ls -la /var/lib/jenkins/workspace/DevOps-Integration-Project/
```

Jenkins checks out the repository into its workspace; source files do not have to be in the Linux user's home directory. Configure jobs through the Jenkins UI rather than editing Jenkins XML files manually.

### 5.4 Configure the Freestyle job

Job name: `DevOps-Integration-Project`

Under **Source Code Management → Git**:
- Repository URL: `https://github.com/Ramana175/Maven.git`
- Branch specifier: `*/main`

Under **Build Steps**, choose **Invoke top-level Maven targets**:
- Maven Version: `Maven-3.10.0`
- Goals used in the practical:

```text
clean package deploy org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar
```

`clean package` cleans previous output and packages the application. `deploy` publishes artifacts only when Maven publishing configuration and repository permissions are correct. The Sonar Maven goal submits analysis and requires a valid SonarQube URL and token.

After saving, select **Build Now** and open **Console Output**. Read the full error output if the build fails.

## 6. SonarQube

### 6.1 Check service and URL

On the SonarQube server:

```bash
sudo systemctl status sonarqube
sudo ss -lntp | grep 9000
```

URL:

```text
http://<SONARQUBE_PUBLIC_IP>:9000
```

Use the current IP and restrict inbound TCP `9000` to intended sources.

### 6.2 Configure the project URL

Add the URL inside the existing `<properties>` section of `pom.xml`. Do not create a duplicate `<properties>` block.

```xml
<properties>
    <sonar.host.url>http://SONARQUBE_PUBLIC_IP:9000</sonar.host.url>
</properties>
```

Replace `SONARQUBE_PUBLIC_IP` with the current address. Do not put the authentication token in `pom.xml` or GitHub.

### 6.3 Create and configure the SonarQube credential in Jenkins

There are two separate settings: the **SonarQube server URL/name** tells Jenkins where SonarQube is, while the **credential** proves that the scanner is allowed to submit analysis. Creating a token in SonarQube does not automatically add it to Jenkins.

**A. Create a token in SonarQube**

1. Sign in to SonarQube using the intended account.
2. Open the user/account menu and find **My Account → Security** (labels can vary by SonarQube version).
3. Generate a token for analysis, give it a clear name, and copy it immediately. Treat it like a password; do not commit it to GitHub or paste it into `pom.xml`.
4. If your SonarQube version offers token types/expiry, choose the least-privileged suitable option and record the expiry securely.

**B. Store the token in Jenkins Credentials**

1. Sign in to Jenkins.
2. Open **Manage Jenkins → Credentials**. Depending on the Jenkins version, choose **System → Global credentials (unrestricted)** or the appropriate folder's credentials store.
3. Select **Add Credentials**.
4. For the common token setup, choose **Kind: Secret text**.
5. Paste the SonarQube token into **Secret**. Set **ID** to a stable ID such as `sonarqube-token` and add a descriptive description such as `SonarQube analysis token`.
6. Save. The secret should be masked and must not be printed in build logs.

The practical referred to this credential by the display description `SonarQube analysis token`. The exact credential **ID** must match whatever your job or SonarQube integration selects; a description is not necessarily the ID.

**C. Configure the SonarQube server in Jenkins**

1. Open **Manage Jenkins → System** and find **SonarQube servers** (this section may require the SonarQube Scanner for Jenkins plugin).
2. Add a server configuration, for example name `SonarQube`, and set the URL to `http://<SONARQUBE_PUBLIC_IP>:9000` or the private/reachable URL appropriate for your network.
3. Select the token credential you stored in Jenkins, then save.
4. If using a freestyle job with the Maven scanner goal, make sure the build step actually passes the token securely. One supported pattern is a **Use secret text(s) or file(s)** build environment binding that maps the `sonarqube-token` Secret text credential to `SONAR_TOKEN`, then run the scanner with `-Dsonar.token=$SONAR_TOKEN` in a shell step. If invoking Maven goals through **Invoke top-level Maven targets**, verify how the installed SonarQube/Jenkins plugin exposes the selected credential and server configuration; do not assume merely creating the credential makes the token available as `SONAR_TOKEN`.

For a freestyle Maven build, a clear alternative is to split the build and scanner into separate build steps: first run `clean package` with the Maven build step, then use a shell/build step with the secret bound as `SONAR_TOKEN` to invoke the scanner. Avoid putting the token literal in the command or source file. If the plugin manages the scanner/server connection for you, follow that plugin's server selection and credential binding instead of duplicating token mechanisms.

### 6.4 Configure the project URL

Add the URL inside the existing `<properties>` section of `pom.xml`. Do not create a duplicate `<properties>` block.

```xml
<properties>
    <sonar.host.url>http://SONARQUBE_PUBLIC_IP:9000</sonar.host.url>
</properties>
```

Replace `SONARQUBE_PUBLIC_IP` with the current address. Do not put the authentication token in `pom.xml` or GitHub. The server URL is configuration, not a secret; the token is a secret and belongs in Jenkins Credentials.

Maven goal used:

```text
org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar
```

After the build, open the project in SonarQube and inspect the latest analysis and Quality Gate. A successful scanner run does not automatically mean the Quality Gate passed. Change default administrator credentials and keep tokens in Jenkins Credentials.

## 7. Nexus Repository Manager

### 7.1 Check Nexus

The practical used Nexus Repository `3.97.1-02`, installed under `/opt/nexus`, with data under `/opt/sonatype-work/nexus3` and web port `8081`.

```bash
sudo systemctl status nexus
sudo ss -lntp | grep 8081
```

URL:

```text
http://<NEXUS_PUBLIC_IP>:8081
```

The UI included repositories such as Maven Central, release, snapshot, `maven-public`, and `maven-central`. The group repository URL used by the Maven client was:

```text
http://<NEXUS_PUBLIC_IP>:8081/repository/maven-public/
```

Use the exact repository name shown in your Nexus UI. Do not assume a repository named `public` exists if the UI shows `maven-public`.

### 7.2 Why `settings.xml` uses a mirror (and why a Nexus IP in `pom.xml` is not the same thing)

**Short answer:** putting the Nexus server IP or username/password into `pom.xml` alone will not reliably route dependency downloads through Nexus. Maven has separate configuration for **where dependencies are downloaded from**, **where this project publishes its own artifact**, and **which credentials are used**.

- **`settings.xml` + `<mirrors>`:** controls repository routing for Maven running on that machine/user. A mirror such as `mirrorOf=*` tells Maven to route requests for repositories through the specified Nexus group URL. This is why we put the Nexus IP and `/repository/maven-public/` URL in the mirror.
- **`pom.xml` + `<distributionManagement>`:** tells Maven where this project should publish its own release or snapshot artifact when you run `mvn deploy`. It does not replace a mirror and does not, by itself, provide credentials.
- **`settings.xml` + `<servers>`:** stores authentication for repository IDs. The `<server><id>` must exactly match the `<id>` used by the publishing repository in `distributionManagement` (or another repository configuration that requires authentication).
- **`pom.xml` dependencies/repositories:** describes project dependencies and may declare repositories, but it is not the right place for passwords or tokens. Putting credentials in `pom.xml` risks exposing them in GitHub.

For example, a Nexus **group** repository like `maven-public` is normally used to download/proxy dependencies. A **hosted release** repository and a **hosted snapshot** repository are used for uploads. Do not use the group URL as the upload target unless your Nexus repository is explicitly configured for that purpose.

### 7.3 Configure Maven mirror

On the **separate Maven client EC2**, the settings file was:

```text
/opt/apache-maven-3.10.0/conf/settings.xml
```

A mirror must be placed inside the existing `<mirrors>` element. Preserve other settings and avoid duplicate root tags.

```xml
<mirrors>
    <mirror>
        <id>nexus-maven-public</id>
        <mirrorOf>*</mirrorOf>
        <url>http://NEXUS_PUBLIC_IP:8081/repository/maven-public/</url>
    </mirror>
</mirrors>
```

Replace `NEXUS_PUBLIC_IP` with the current address. This is an example mirror block, not a full replacement for an existing `settings.xml`.

A mirror routes dependency requests through Nexus; it does not by itself configure artifact publishing.

### 7.4 Publish artifacts

For `mvn deploy`, the project's `pom.xml` needs valid `distributionManagement` pointing to hosted repositories that accept deployments. If authentication is enabled, credentials belong in Maven `settings.xml` under `<servers>`, using an ID matching the repository ID in `pom.xml`.

Example structure only—use the actual repository names and URLs in Nexus:

```xml
<distributionManagement>
    <repository>
        <id>nexus-releases</id>
        <url>http://NEXUS_PUBLIC_IP:8081/repository/RELEASE_REPOSITORY/</url>
    </repository>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>http://NEXUS_PUBLIC_IP:8081/repository/SNAPSHOT_REPOSITORY/</url>
    </snapshotRepository>
</distributionManagement>
```

Then add credentials to the Maven `settings.xml` `<servers>` section. The IDs below must match the IDs in `distributionManagement` exactly:

```xml
<servers>
    <server>
        <id>nexus-releases</id>
        <username>YOUR_NEXUS_USERNAME</username>
        <password>YOUR_NEXUS_PASSWORD</password>
    </server>
    <server>
        <id>nexus-snapshots</id>
        <username>YOUR_NEXUS_USERNAME</username>
        <password>YOUR_NEXUS_PASSWORD</password>
    </server>
</servers>
```

This is a structural example, not a full `settings.xml`. Keep secrets out of GitHub. For shared/production environments, prefer an appropriately protected credentials mechanism and HTTPS. The Maven mirror's `<id>` (`nexus-maven-public`) is a different purpose from the upload server IDs (`nexus-releases` and `nexus-snapshots`).

**How Maven selects release versus snapshot:** a version such as `1.0.0` is a release; a version ending in `-SNAPSHOT`, such as `1.0.1-SNAPSHOT`, is a snapshot. Maven selects the matching destination in `distributionManagement`. Ensure the target repositories exist in Nexus and allow deployment for the user.

Nexus troubleshooting:
- **`Unrecognised tag: mirror`:** `<mirror>` is outside `<mirrors>`.
- **XML mismatch:** check nesting and ensure there is one `<settings>` root.
- **Dependency resolution fails:** verify the mirror URL and Nexus status, then retry with `mvn -U validate`.
- **`mvn validate` succeeds but no artifact appears:** `validate` does not publish artifacts. Check `distributionManagement`, hosted repository, permissions, and `mvn deploy`.
- **Anonymous access:** read access does not necessarily allow publishing. Use authentication for writes.
- **Network security:** the lab used HTTP for practice. Prefer HTTPS for internet-exposed or production systems and restrict administrative access.

## 8. Tomcat Deployment

### 8.1 Check Tomcat

The practical used Tomcat `10.1.61` at `/opt/apache-tomcat-10.1.61`, listening on port `8080`.

```bash
sudo systemctl status tomcat
sudo ss -lntp | grep 8080
```

If the service has another name or is not managed by systemd, use the service/start method configured for that installation.

### 8.2 Application URLs

WAR:

```text
tomcat-web-application/target/tomcat-web-application-1.0.0.war
```

Home page:

```text
http://<TOMCAT_PUBLIC_IP>:8080/tomcat-web-application/
```

Servlet:

```text
http://<TOMCAT_PUBLIC_IP>:8080/tomcat-web-application/hello
```

The home page displayed `Welcome to My Tomcat Web Application!`; `/hello` responded `Hello from my Tomcat web application!`.

### 8.3 Tomcat Manager remote deployment

Relevant files:

```text
/opt/apache-tomcat-10.1.61/webapps/manager/META-INF/context.xml
/opt/apache-tomcat-10.1.61/conf/tomcat-users.xml
```

Tomcat Manager commonly restricts remote access with a `RemoteCIDRValve`. Allow only the current Jenkins server IP (or the required private network). Do not remove the restriction and expose Manager to everyone.

Jenkins credential used:
- ID: `tomcat-manager`
- Username used in the practical: `tomcat`
- Type: Username with password

Keep the real password in Jenkins Credentials. Configure only the roles required by the deployment plugin/API.

### 8.4 Jenkins post-build deployment

Configure these Post-build Actions.

**Archive artifacts**

```text
tomcat-web-application/target/*.war
```

**Deploy WAR/EAR to a container**

| Setting | Value |
|---|---|
| WAR/EAR files | `tomcat-web-application/target/*.war` |
| Context path | `tomcat-web-application` |
| Adapter shown in the practical | `Tomcat 9.x Remote` |
| Credentials | `tomcat-manager` |
| Tomcat URL | `http://<TOMCAT_PUBLIC_IP>:8080/` |

The plugin showed `Tomcat 9.x Remote` even though the server was Tomcat `10.1.61`; this setup worked in the practical. Check compatibility when using different versions.

After deployment, open the home page and `/hello` URL to confirm the application works.

## 9. Final Jenkins Job Configuration

| Setting | Value |
|---|---|
| Job type/name | Freestyle / `DevOps-Integration-Project` |
| Repository | `https://github.com/Ramana175/Maven.git` |
| Branch | `*/main` |
| Build step | Invoke top-level Maven targets |
| Maven installation | `Maven-3.10.0` |
| Goals | `clean package deploy org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar` |
| SonarQube credential | `SonarQube analysis token` |
| Archive pattern | `tomcat-web-application/target/*.war` |
| Deployment context | `tomcat-web-application` |
| Tomcat credential | `tomcat-manager` |

`deploy` publishes to Nexus only if Maven publishing is configured. Check the SonarQube Quality Gate separately from scanner success.

The practical's successful console output showed SonarScanner completion, `BUILD SUCCESS`, artifact archiving, and deployment of `tomcat-web-application-1.0.0.war`.

## 10. End-to-End Verification

- [ ] Latest source code and `pom.xml` are visible in GitHub.
- [ ] Jenkins checks out the correct repository and `main` branch.
- [ ] Jenkins selects Maven installation `Maven-3.10.0`.
- [ ] Maven creates the WAR in the correct module's `target` directory.
- [ ] SonarQube shows the latest analysis and Quality Gate status.
- [ ] Nexus is running and repository URLs match the Nexus UI.
- [ ] If publishing, `distributionManagement` and credentials are configured.
- [ ] Jenkins archives and deploys the WAR.
- [ ] Application home page loads and `/hello` responds.

## 11. Troubleshooting

| Problem | What to check |
|---|---|
| Maven not found | Jenkins Tools configuration and selected Maven installation |
| Files missing from Jenkins home | Inspect `/var/lib/jenkins/workspace/DevOps-Integration-Project/` |
| WAR not found | Correct module, `target` directory, and version in `pom.xml` |
| Git push rejected | `git status`, branch, remote commits, and merge state |
| SonarQube UI unavailable | Service, port `9000`, current IP, security group, logs |
| SonarQube authentication fails | Confirm token is valid, the correct Secret text credential is selected/bound, `SONAR_TOKEN` is actually available to the scanner, and the server URL is reachable; never print the token |
| Quality Gate not green | Inspect project conditions and findings in SonarQube |
| Nexus mirror XML error | Put `<mirror>` inside `<mirrors>` and check XML nesting |
| Maven dependency resolution fails | Check `settings.xml` mirror nesting/URL, Nexus status, network access, and Maven cached failures; retry with `mvn -U` after fixing the cause |
| `validate` succeeds but nothing uploaded | `validate` does not upload; configure `distributionManagement`, matching `<servers>` IDs/credentials, hosted repository permissions, and run `mvn deploy` |
| Tomcat timeout | Current IP, port `8080`, listener, security group |
| Tomcat Manager rejects Jenkins | IP allow-list, required role, `tomcat-manager` credential, and Manager endpoint reachability |
| `Permission denied` reading a file | Use `ls -l` and `namei -l <path>`; check ownership and permissions |
| Wrong application URL | Include `/tomcat-web-application/` in the URL |

Change one thing at a time, read the full error, and run the smallest useful test before changing additional configuration.

## 12. Command Cheat Sheet

### Git (PowerShell)

```powershell
git status
git remote -v
git branch
git add .
git commit -m "Describe the change"
git push
```

### Maven

```bash
mvn -version
mvn clean package
mvn -U validate
```

Use `mvn deploy` only after publishing is configured.

### Service and port checks

```bash
sudo systemctl status jenkins
sudo systemctl status sonarqube
sudo systemctl status nexus
sudo ss -lntp | grep 8080
sudo ss -lntp | grep 9000
sudo ss -lntp | grep 8081
```

### Logs

```bash
sudo journalctl -u jenkins -n 100 --no-pager
sudo journalctl -u sonarqube -n 100 --no-pager
sudo journalctl -u nexus -n 100 --no-pager
```

Tomcat logs are commonly under its `logs` directory.

## 13. Security and Cleanup

- Never commit passwords, tokens, private keys, or credential files to GitHub.
- Store SonarQube and Tomcat credentials in Jenkins Credentials.
- Restrict EC2 security groups to necessary IPs and ports.
- Restrict Tomcat Manager and Nexus administration/publishing access.
- Prefer HTTPS and suitable authentication for internet-exposed services.
- Avoid broad permissions such as `chmod -R 777`.
- Record current public IPs; they may change after stop/start.
- Stop or terminate lab resources when not needed. Check EC2 instances, EBS volumes, Elastic IPs, snapshots, and other resources for ongoing charges.

---

**Summary:** GitHub stores source code. Jenkins checks it out and runs Maven. SonarQube analyzes source quality. Nexus manages dependencies and artifacts when configured. Tomcat runs the deployed WAR.
