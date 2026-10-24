# DevOps Integration Project: GitHub, Jenkins, Maven, SonarQube, Nexus, and Tomcat

A beginner-friendly, hands-on guide to building and deploying a Java web application using a simple DevOps workflow.

This README is written as a practical reference: follow the sections in order, understand what each tool does, and use the troubleshooting notes when something goes wrong.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [How the Workflow Works](#2-how-the-workflow-works)
3. [Tools and Servers Used](#3-tools-and-servers-used)
4. [Step 1 — Push the Project to GitHub](#4-step-1--push-the-project-to-github)
5. [Step 2 — Configure Jenkins and Maven](#5-step-2--configure-jenkins-and-maven)
6. [Step 3 — Integrate SonarQube](#6-step-3--integrate-sonarqube)
7. [Step 4 — Integrate Nexus Repository Manager](#7-step-4--integrate-nexus-repository-manager)
8. [Step 5 — Deploy the WAR to Tomcat](#8-step-5--deploy-the-war-to-tomcat)
9. [Final Jenkins Job Configuration](#9-final-jenkins-job-configuration)
10. [End-to-End Verification](#10-end-to-end-verification)
11. [Troubleshooting Notes from the Practicals](#11-troubleshooting-notes-from-the-practicals)
12. [Important Commands Cheat Sheet](#12-important-commands-cheat-sheet)
13. [Security and Cleanup Notes](#13-security-and-cleanup-notes)

---

## 1. Project Overview

The goal is to take a Java web application from source code to a running application using a straightforward DevOps workflow.

We do the work in this fixed order:

1. **GitHub** — store and version-control the source code.
2. **Jenkins + Maven** — retrieve the code and build the application.
3. **SonarQube** — analyze source-code quality.
4. **Nexus Repository Manager** — store and retrieve build artifacts and dependencies.
5. **Apache Tomcat** — deploy and run the web application.

The application used in the practical is a Maven-based Java web application packaged as a **WAR** file. Its context path is `tomcat-web-application`.

### What success looks like

At the end:

- The project is stored in GitHub.
- Jenkins checks out the repository and runs the Maven build.
- SonarQube analysis completes successfully.
- Maven can use Nexus as a repository manager.
- Jenkins archives the WAR and deploys it to Tomcat.
- The application opens in a browser.

---

## 2. How the Workflow Works

```text
Developer
   |
   | git add, git commit, git push
   v
GitHub repository
   |
   | Jenkins checks out the source
   v
Jenkins + Maven
   |
   | compile, test (if configured), package WAR
   | run SonarQube analysis
   | use Nexus for dependencies/artifact publishing
   v
WAR artifact
   |
   | Jenkins post-build deployment
   v
Apache Tomcat
   |
   v
Application available in a browser
```

**Simple explanation:** GitHub stores the code. Jenkins automates the work. Maven builds the Java project. SonarQube checks code quality. Nexus manages artifacts and dependencies. Tomcat runs the deployed web application.

> The exact order of Maven goals matters. A successful `mvn deploy` publishes an artifact only when the project and Maven settings are configured for a deployment repository. A successful SonarQube scan does not automatically mean a Quality Gate has passed; check the result in SonarQube.

---

## 3. Tools and Servers Used

The practical used separate Ubuntu EC2 servers for the tools. Public IP addresses can change when an EC2 instance is stopped and started, so use the current address shown in your own AWS console.

| Component | Purpose | Practical details |
|---|---|---|
| GitHub | Source-code repository | `Ramana175/Maven` |
| Jenkins | Automation/build server | Freestyle job: `DevOps-Integration-Project` |
| Maven | Build and package Java code | Maven `3.10.0` configured in Jenkins |
| SonarQube | Source-code analysis | Community Build, web port `9000` |
| Nexus | Artifact/dependency repository manager | Nexus Repository `3.97.1-02`, web port `8081` |
| Tomcat | Java web application server | Tomcat `10.1.61`, port `8080` |

Example repository URL used in the practical:

```text
https://github.com/Ramana175/Maven.git
```

Example application URL used for final verification:

```text
http://<TOMCAT_PUBLIC_IP>:8080/tomcat-web-application/
```

Do not copy an old public IP blindly. Replace placeholders with the current addresses of your instances.

---

## 4. Step 1 — Push the Project to GitHub

The project source code is stored in the GitHub repository:

```text
https://github.com/Ramana175/Maven
```

From your project folder on your computer, add the changed files, commit them, and push them to GitHub:

```powershell
git add .
git commit -m "Update DevOps project"
git push
```

Check the repository in your browser and confirm that the latest files are visible. Jenkins will use this repository to fetch the source code.

If Git says there is nothing to commit, there may be no new changes. If `git push` is rejected because the remote contains newer commits, check `git status` and resolve the branch/merge issue before pushing. Avoid adding extra Git setup commands when the repository is already configured.

---

## 5. Step 2 — Configure Jenkins and Maven
Installing Jenkins does not automatically configure Maven. We configured Maven through the Jenkins web UI rather than placing the project files on the server by hand.

The practical used Maven `3.10.0`, managed by Jenkins and installed at:

```text
/var/lib/jenkins/tools/hudson.tasks.Maven_MavenInstallation/Maven-3.10.0
```

**Important:** `/opt/apache-maven-3.10.0` was the installation path on our separate Maven client EC2 server, not on the Jenkins server. We checked the Jenkins server's `/opt` directory and found it empty.

In Jenkins:
### 5.1 Jenkins Service Management

Jenkins runs as a system service on the Ubuntu EC2 instance. We can manage it using `systemctl`.

**Connect to the Jenkins EC2 server**

First, connect to your Jenkins server using SSH.

**Check Jenkins status**

```bash
sudo systemctl status jenkins
```

This shows whether Jenkins is running, stopped, or has encountered an error.

**Start Jenkins**

Use this command when Jenkins is stopped:

```bash
sudo systemctl start jenkins
```

**Stop Jenkins**

Use this command when you need to stop Jenkins temporarily:

```bash
sudo systemctl stop jenkins
```

**Restart Jenkins**

Use this after changing Jenkins-related configuration when a restart is required:

```bash
sudo systemctl restart jenkins
```

**Enable Jenkins at system startup**

This configures Jenkins to start automatically when the EC2 instance boots:

```bash
sudo systemctl enable jenkins
```

**Disable automatic startup**

```bash
sudo systemctl disable jenkins
```

This disables automatic startup at boot; it does not necessarily stop Jenkins immediately.

**Check Jenkins service logs**

To view recent logs:

```bash
sudo journalctl -u jenkins -n 50 --no-pager
```

To follow logs live while troubleshooting:

```bash
sudo journalctl -u jenkins -f
```

Press `Ctrl+C` to stop following the logs.

**Check whether Jenkins is listening on port 8080**

```bash
sudo ss -lntp | grep ':8080'
```

Jenkins commonly uses port `8080` in this practical.

**Access Jenkins in your browser**

```text
http://<JENKINS_PUBLIC_IP>:8080
```

Replace `<JENKINS_PUBLIC_IP>` with the current public IP address of your Jenkins EC2 instance. Make sure the EC2 security group permits inbound TCP traffic on port `8080` from your intended client IP.

**Important:** Restarting Jenkins can interrupt running builds. Check the job activity before restarting the service.

1. Open **Manage Jenkins → Tools** (or **Global Tool Configuration**, depending on the version).
2. Find **Maven installations** and add or select the Maven installation.
3. Name it `Maven-3.10.0`.
4. Save the configuration and make sure the job selects this same name.

To verify Maven on the Jenkins server, run:

```bash
sudo -u jenkins /var/lib/jenkins/tools/hudson.tasks.Maven_MavenInstallation/Maven-3.10.0/bin/mvn -version
```

The verified output showed:

- **Maven:** `3.10.0`
- **Java:** `21.0.12.1`
- **Java runtime:** `/usr/lib/jvm/java-21-openjdk-amd64`

### 5.3 Configure the Freestyle job

The job name is:

```text
DevOps-Integration-Project
```

In **Source Code Management → Git**:

- **Repository URL:** `https://github.com/Ramana175/Maven.git`
- **Branch specifier:** `*/main`

In **Build Steps**, select **Invoke top-level Maven targets**:

- **Maven Version:** `Maven-3.10.0`
- The final practical used these goals:

```text
clean package deploy org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar
```

The goals are run by Maven from the checked-out project. `deploy` requires the project's Maven publishing configuration to be set up for Nexus; the SonarQube goal requires a working SonarQube connection and token.

### 5.4 Check the build

Click **Build Now**, then open the build's **Console Output**. The build should fetch the GitHub repository, run Maven, and create the WAR under the relevant module's `target` directory, for example:

```text
tomcat-web-application/target/tomcat-web-application-1.0.0.war
```

The exact WAR version comes from `pom.xml`.

In this practical, the console output showed `BUILD SUCCESS`, archived the artifacts, and deployed the WAR to Tomcat.

### 5.1 Important: Jenkins and Maven were configured through the Jenkins UI

In this practical, we did not copy the project files manually onto the Jenkins server. Jenkins fetched the source code from GitHub when the job ran.

That is why running `ls` in the Jenkins server's home directory may not show the application source code. Jenkins keeps its job configuration and build workspace in its own directories.

Common Jenkins locations on Ubuntu are:

```text
/var/lib/jenkins/
    config.xml                 # Main Jenkins system configuration
    jobs/
        DevOps-Integration-Project/
            config.xml         # Configuration for this Jenkins job
            builds/            # Build records and console output
            workspace/         # Checked-out project files during builds
```

The exact contents and paths can vary. The application files are normally checked out into the job's workspace when the job runs; they are not necessarily placed in the Linux user's home directory.

You can inspect the job workspace on the Jenkins server with:

```bash
sudo ls -la /var/lib/jenkins/workspace/DevOps-Integration-Project/
```

Do not edit Jenkins `config.xml` files manually for this practical. We configured the job using the Jenkins web UI, which is safer and easier to understand.

### 5.2 Configure Maven in Jenkins

Installing Jenkins does not automatically configure Maven. We configured Maven through the Jenkins web UI rather than placing the project files on the server by hand.

The practical used Maven `3.10.0`, installed at:

```text
/opt/apache-maven-3.10.0
```

In Jenkins:

1. Open **Manage Jenkins → Tools** (or **Global Tool Configuration**, depending on the version).
2. Find **Maven installations** and add/select the Maven installation.
3. Name it `Maven-3.10.0`.
4. Save the configuration and make sure the job selects this same name.

If you have terminal access to the server, you can check the Maven installation with:

```bash
/opt/apache-maven-3.10.0/bin/mvn -version
```

### 5.3 Configure the Freestyle job

The job name is:

```text
DevOps-Integration-Project
```

In **Source Code Management → Git**:

- **Repository URL:** `https://github.com/Ramana175/Maven.git`
- **Branch specifier:** `*/main`

In **Build Steps**, select **Invoke top-level Maven targets**:

- **Maven Version:** `Maven-3.10.0`
- The final practical used these goals:

  ```text
  clean package deploy org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar
  ```

The goals are run by Maven from the checked-out project. `deploy` needs the project's Maven publishing configuration to be set up for Nexus; the SonarQube goal needs a working SonarQube connection and token.

### 5.4 Check the build

Click **Build Now**, then open the build's **Console Output**. The build should fetch the GitHub repository, run Maven, and create the WAR under the relevant module's `target` directory, for example:

```text
tomcat-web-application/target/tomcat-web-application-1.0.0.war
```

The exact WAR version comes from `pom.xml`.

### Common Jenkins/Maven issues

- **No application files when running `ls` in the Jenkins home directory:** check the job workspace under `/var/lib/jenkins/workspace/DevOps-Integration-Project/`. Jenkins checks out the repository into the workspace when the job runs.
- **Maven is not found:** check the Maven installation in Jenkins Tools and confirm the job selects `Maven-3.10.0`.
- **WAR not found:** check the correct module's `target` directory and the version in `pom.xml`.
- **Java/compiler error:** check the Java version used by Maven and the compiler settings in `pom.xml`. The practical used Java 21.

---

## 6. Step 3 — Integrate SonarQube

### 6.1 What SonarQube does

SonarQube analyzes source code for maintainability issues, bugs, vulnerabilities, security hotspots, and other quality findings. It complements compilation: code can compile successfully and still have issues reported by static analysis.

The practical used a SonarQube Community Build on an Ubuntu EC2 server, accessible on port `9000`.

### 6.2 Check SonarQube is running

On the SonarQube server:

```bash
sudo systemctl status sonarqube
```

If the service is configured under a different name, use that service name. Check the web UI at:

```text
http://<SONARQUBE_PUBLIC_IP>:9000
```

Use the current public IP and make sure the EC2 security group allows port `9000` only from the intended sources.

### 6.3 Configure the SonarQube URL in `pom.xml`

In our practical, we also updated the project's `pom.xml` with the SonarQube server URL so Maven analysis knows which SonarQube server to contact. The URL must be the current address of your SonarQube server.

For example, the property can be configured like this:

```xml
<properties>
  <sonar.host.url>http://<SONARQUBE_PUBLIC_IP>:9000</sonar.host.url>
</properties>
```

If your `pom.xml` already has a `<properties>` section, add the property inside that existing section rather than creating a duplicate. Replace the placeholder with the current SonarQube address.

The URL is configuration, not a secret. The SonarQube token is a secret and should stay in Jenkins Credentials (in this practical, the credential was named `SonarQube analysis token`). Do not commit the token to GitHub.

### 6.4 Configure the Jenkins connection

In Jenkins:

1. Open **Manage Jenkins → System**.
2. Find the SonarQube servers section.
3. Add the SonarQube server URL.
4. Select the Jenkins credential containing the SonarQube token.
5. Save the configuration.

The practical used a Jenkins credential named `SonarQube analysis token`, bound to the environment variable `SONAR_TOKEN`. Keep the token in Jenkins Credentials; do not commit it to GitHub or place it in the README.

### 6.5 Run the analysis

The integrated Maven build used the Sonar Maven plugin goal:

```text
org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar
```

The final job's console output showed that the SonarScanner engine completed successfully and Maven reported `BUILD SUCCESS`.

Open the project in SonarQube and check the latest analysis. A completed analysis and a passed Quality Gate are different things; read the actual Quality Gate status shown in the UI.

### SonarQube troubleshooting and lessons

**The service is running but the page does not open**

Check the service status and port listener on the server:

```bash
sudo systemctl status sonarqube
sudo ss -lntp | grep 9000
```

Also check the EC2 security group and the address/port used in the browser.

**Analysis cannot authenticate**

Check that the token is valid, the Jenkins credential is selected correctly, and the environment variable or server connection is configured as expected. Do not paste tokens into console output, source code, or GitHub.

**Default administrator credentials warning**

Change the default administrator password when prompted. Do not leave a publicly accessible SonarQube instance using default credentials.

**PostgreSQL or database setup is incomplete**

SonarQube's database configuration must match the installed database service and credentials. Check the SonarQube service logs and configuration instead of repeatedly reinstalling the application. Back up data before replacing or deleting a database.

**Analysis completed but the Quality Gate is not green**

Open the project in SonarQube and inspect the Quality Gate conditions and findings. A successful scanner run only means the analysis was processed; it does not guarantee that all quality conditions passed.

---

## 7. Step 4 — Integrate Nexus Repository Manager

### 7.1 What Nexus does

Nexus Repository Manager stores and serves build artifacts and can proxy external dependency repositories. This gives a team a central place to retrieve approved dependencies and publish versioned build artifacts.

In the practical, Nexus Repository `3.97.1-02` was installed on Ubuntu EC2 and used port `8081`.

### 7.2 Check Nexus is running

On the Nexus server:

```bash
sudo systemctl status nexus
sudo ss -lntp | grep 8081
```

Open:

```text
http://<NEXUS_PUBLIC_IP>:8081
```

The practical's Nexus installation used:

```text
/opt/nexus
/opt/sonatype-work/nexus3
```

Do not assume these paths are the same on every installation.

### 7.3 Understand the repositories

The UI showed repositories including Maven Central, Maven release, Maven snapshot, `maven-public`, and `maven-central`.

The working group repository URL used in the Maven client setup was:

```text
http://<NEXUS_PUBLIC_IP>:8081/repository/maven-public/
```

Use the repository name that actually appears in your Nexus UI. A repository name in a guide is not proof that a repository with that name exists on your server.

### 7.4 Configure Maven to use Nexus

The practical edited Maven's settings file at:

```text
/opt/apache-maven-3.10.0/conf/settings.xml
```

A Maven mirror entry belongs **inside the existing `<mirrors>` element**, not after `</mirrors>` or outside the `<settings>` root element.

Example structure:

```xml
<settings>
  <mirrors>
    <mirror>
      <id>nexus-maven-public</id>
      <mirrorOf>*</mirrorOf>
      <url>http://<NEXUS_PUBLIC_IP>:8081/repository/maven-public/</url>
    </mirror>
  </mirrors>
</settings>
```

This is a structural example, not a complete replacement for an existing `settings.xml`. Preserve the other settings already present in your file and place the `<mirror>` element inside the existing `<mirrors>` block. Replace the placeholder with the current Nexus address.

**Important:** A mirror redirects Maven repository requests through Nexus; it does not by itself configure where your own project is published. Publishing with `mvn deploy` also requires a `<distributionManagement>` section in the project's `pom.xml` and matching credentials in Maven settings when authentication is enabled.

### 7.5 Configure artifact publishing

For a project to publish its release artifact to Nexus, configure the appropriate repository URL in the project's `pom.xml`, for example:

```xml
<distributionManagement>
  <repository>
    <id>nexus-releases</id>
    <url>http://<NEXUS_PUBLIC_IP>:8081/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <url>http://<NEXUS_PUBLIC_IP>:8081/repository/maven-snapshots/</url>
  </snapshotRepository>
</distributionManagement>
```

Use the exact repository names and URLs from your Nexus UI. This example assumes those hosted repositories exist and are configured for publishing.

If authentication is required, add matching credentials to `settings.xml` under `<servers>` with an `<id>` that matches the repository ID in `pom.xml`. Keep usernames and passwords private. Never commit credentials to GitHub.

### Nexus troubleshooting and lessons from the practical

**`Unrecognised tag: 'mirror'`**

The `<mirror>` element was placed outside `<mirrors>`. Move it inside the `<mirrors>` element.

**Mismatched `</settings>` or other XML errors**

Check the nesting of XML elements and ensure there is only one root `<settings>` element. Do not add a second closing tag or replace the full file unnecessarily.

**Maven reports an artifact was cached from `maven-default-http-blocker`**

Maven may have cached a failed resolution attempt. First verify the mirror URL and repository status. Then retry with:

```bash
mvn -U validate
```

If the failure persists, inspect the full error and local Maven cache for the affected artifact. Do not delete the entire local repository as a first step.

**`mvn validate` succeeds, but `mvn deploy` does not publish the artifact**

`validate` does not upload the project artifact. Check that the project has a valid `distributionManagement` section, that the repository is a hosted repository accepting deployments, that the URL is correct, and that credentials match the configured repository ID.

**Anonymous access**

Anonymous access can permit reading repositories, but it should not be used as a substitute for proper deployment permissions. Use authentication for publishing and do not expose administrative credentials.

**HTTP versus HTTPS**

The lab used HTTP URLs for the internal practice setup. For production or internet-exposed environments, prefer HTTPS and restrict access with network controls. Do not expose Nexus administration or write access broadly to the internet.

---

## 8. Step 5 — Deploy the WAR to Tomcat

### 8.1 What Tomcat does

Apache Tomcat is a Java web application server that runs servlet-based web applications. Maven packages the application into a WAR file; Tomcat deploys that WAR and serves the application over HTTP.

The practical used Tomcat `10.1.61` at:

```text
/opt/apache-tomcat-10.1.61
```

Tomcat listened on port `8080`.

### 8.2 Check Tomcat is running

On the Tomcat server:

```bash
sudo systemctl status tomcat
```

If the service was installed under a different name, use that service name. Confirm that the process is listening on port `8080`:

```bash
sudo ss -lntp | grep 8080
```

If the service is not managed by systemd in your installation, check it using the method used to start Tomcat.

### 8.3 Confirm the application can run

The WAR built by the practical was:

```text
tomcat-web-application/target/tomcat-web-application-1.0.0.war
```

The deployment context path was:

```text
tomcat-web-application
```

The application URL therefore follows this pattern:

```text
http://<TOMCAT_PUBLIC_IP>:8080/tomcat-web-application/
```

The application home page displayed:

```text
Welcome to My Tomcat Web Application!
```

The application also included a `hello` servlet, accessible at:

```text
http://<TOMCAT_PUBLIC_IP>:8080/tomcat-web-application/hello
```

Use the actual Tomcat IP currently assigned to your EC2 instance.

### 8.4 Configure Tomcat Manager for remote deployment

Jenkins uses the Tomcat Manager interface to deploy the WAR remotely. The practical configured the Manager application and allowed the Jenkins server's public IP in the Manager application's access-control configuration.

Relevant files in the practical were:

```text
/opt/apache-tomcat-10.1.61/webapps/manager/META-INF/context.xml
/opt/apache-tomcat-10.1.61/conf/tomcat-users.xml
```

The Manager access configuration uses a `RemoteCIDRValve` by default to restrict which client IPs may access it. Add only the required, current Jenkins public IP to the allow-list, preserving the existing configuration. Do not remove the restriction and expose Manager to everyone.

The Tomcat user needs the permissions required by the deployment method. The practical used a Jenkins credential with username `tomcat`, stored in Jenkins as:

- **Kind:** Username with password
- **ID:** `tomcat-manager`
- **Description:** `Tomcat Manager credentials`

Store the real password in Jenkins Credentials, not in this README or the Git repository.

### 8.5 Configure the Jenkins post-build deployment

In the Jenkins job, under **Post-build Actions**, configure:

**Archive the artifacts**

```text
tomcat-web-application/target/*.war
```

**Deploy war/ear to a container**

- **WAR/EAR files:** `tomcat-web-application/target/*.war`
- **Context path:** `tomcat-web-application`
- **Container:** `Tomcat 9.x Remote` (the remote deployment adapter available in the installed Jenkins plugin)
- **Credentials:** `tomcat-manager`
- **Tomcat URL:** `http://<TOMCAT_PUBLIC_IP>:8080/`

The lab's plugin configuration showed `Tomcat 9.x Remote` even though the server itself was Tomcat `10.1.61`. The configured adapter successfully redeployed the WAR in this practical. The label describes the Jenkins plugin's adapter; verify compatibility when using a different plugin or server version.

### 8.6 Verify the deployment

After the Jenkins build succeeds, open:

```text
http://<TOMCAT_PUBLIC_IP>:8080/tomcat-web-application/
```

Confirm that the home page loads. Then test:

```text
http://<TOMCAT_PUBLIC_IP>:8080/tomcat-web-application/hello
```

Confirm that the servlet responds.

### Tomcat troubleshooting and lessons from the practical

**Browser times out even though Tomcat is running**

Check that Tomcat is listening on the expected port and that the EC2 security group allows inbound TCP `8080` from your intended client IP. Also verify that the public IP is current.

**Tomcat Manager blocks remote access**

The Manager application may only allow localhost by default. Review the `RemoteCIDRValve` configuration and allow only the required Jenkins client IP. A changed public IP may require updating the allow-list.

**Jenkins deployment fails with an authentication error**

Confirm that the Jenkins job selects the correct credential ID (`tomcat-manager`) and that the username/password match the configured Tomcat Manager user.

**Jenkins deployment fails with a forbidden response**

Check the Tomcat Manager user's required role and the Manager access-control rules. The roles needed depend on the deployment API used by the plugin. Do not solve this by granting broad access to all users.

**`Permission denied` while reading an application file**

Check file and directory ownership/permissions on the server. Use `ls -l` and `namei -l <path>` to identify which directory blocks access. Avoid using `chmod -R 777`; grant only the access needed by the Tomcat service account.

**Application opens at the wrong URL**

The context path is `tomcat-web-application`, so include it in the URL. A root URL such as `http://<TOMCAT_PUBLIC_IP>:8080/` is not the same as the deployed application URL.

**WAR redeployment succeeds but the browser still shows old content**

Hard-refresh the browser and confirm the latest Jenkins build deployed the expected WAR. Check the build number, artifact path, and Tomcat logs before changing configuration.

---

## 9. Final Jenkins Job Configuration

The final working job was a **Freestyle project** named:

```text
DevOps-Integration-Project
```

### Source Code Management

| Setting | Value |
|---|---|
| SCM | Git |
| Repository | `https://github.com/Ramana175/Maven.git` |
| Branch | `*/main` |

### Build step

| Setting | Value |
|---|---|
| Build step | Invoke top-level Maven targets |
| Maven installation | `Maven-3.10.0` |
| Goals used in the final practical | `clean package deploy org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar` |

The build also used the Jenkins credential for the SonarQube analysis token. Nexus publishing depends on the Maven repository configuration described earlier.

### Post-build actions

| Action | Configuration |
|---|---|
| Archive artifacts | `tomcat-web-application/target/*.war` |
| Deploy WAR/EAR | `tomcat-web-application/target/*.war` |
| Context path | `tomcat-web-application` |
| Tomcat adapter | `Tomcat 9.x Remote` |
| Jenkins credential | `tomcat-manager` |
| Tomcat URL | `http://<TOMCAT_PUBLIC_IP>:8080/` |

### Successful final build

The final Jenkins console output reported:

```text
[INFO] SonarScanner Engine completed successfully
[INFO] BUILD SUCCESS
Archiving artifacts
[DeployPublisher][INFO] Attempting to deploy 1 war file(s)
[DeployPublisher][INFO] Deploying ...tomcat-web-application-1.0.0.war
Finished: SUCCESS
```

The deployment log also showed the WAR being redeployed. The browser check then confirmed that the application page opened.

---

## 10. End-to-End Verification

Use this checklist when repeating the practical.

- [ ] The latest code and `pom.xml` are visible in GitHub.
- [ ] Jenkins can check out the repository.
- [ ] Jenkins has a configured Maven installation.
- [ ] The Maven build creates the WAR in the expected `target` directory.
- [ ] SonarQube shows the latest analysis for the project.
- [ ] Nexus is running and Maven repository URLs are correct.
- [ ] If publishing to Nexus, the Maven `distributionManagement` and credentials are configured.
- [ ] Jenkins archives the WAR.
- [ ] Jenkins deploys the WAR to Tomcat.
- [ ] The application home page opens.
- [ ] The `/hello` servlet responds.

A successful Jenkins build and deployment are good signs, but always verify the application in a browser and inspect the SonarQube result separately.

---

## 11. Troubleshooting Notes from the Practicals

These are the main lessons from the errors encountered during the labs.

| Problem | Cause or lesson | What to check |
|---|---|---|
| Jenkins installed, but Maven build not ready | Jenkins and Maven are separate tools; Maven must be installed or configured in Jenkins. | Jenkins Tools and the Maven installation name selected by the job. |
| Maven executable not found | Maven may not be in the shell's `PATH`. | Run Maven by its full path and check the Jenkins tool configuration. |
| WAR file not found | The project may be inside a subdirectory or the file name/version may differ. | The correct module's `target` directory and `pom.xml`. |
| Git push rejected | The remote has commits not present locally, or branch names differ. | `git status`, `git branch`, and `git remote -v`; merge carefully. |
| Git merge still in progress | Conflicts were resolved but the merge commit was not completed. | `git status`, stage resolved files, then run `git commit`. |
| SonarQube UI unavailable | Service, listener, security group, IP, or database configuration may be wrong. | Service status, logs, port `9000`, security-group rules. |
| SonarQube default password warning | The default administrator credentials have not been changed. | Change the password and store credentials securely. |
| Nexus `Unrecognised tag: mirror` | The mirror element was placed outside the `<mirrors>` block. | XML structure in `settings.xml`. |
| Nexus XML mismatch | Incorrect opening/closing tags or duplicate root elements. | Preserve a single `<settings>` root and validate nesting. |
| Maven dependency resolution keeps failing | Wrong mirror URL, repository issue, or cached failed resolution may be involved. | Full Maven error, repository URL, Nexus status; retry with `mvn -U`. |
| `mvn validate` works but Nexus has no uploaded artifact | Validation does not publish artifacts. | `distributionManagement`, hosted repository, credentials, and `mvn deploy`. |
| Tomcat page times out | Wrong/current IP, port listener, or security-group rule. | Tomcat service, port `8080`, and network rules. |
| Tomcat Manager denies Jenkins | Manager IP restriction, credentials, or required role is incorrect. | Manager `context.xml`, `tomcat-users.xml`, Jenkins credential selection. |
| `Permission denied` for a deployed file | File or parent-directory permissions prevent access. | `ls -l`, `namei -l`, ownership, and minimum required permissions. |
| Deployment works but application URL fails | Wrong context path or wrong WAR artifact. | Use `/tomcat-web-application/` and confirm the deployed WAR. |

**General troubleshooting habit:** change one thing at a time, read the complete error message, confirm the relevant service and URL, and re-run the smallest useful test before changing more configuration.

---

## 12. Important Commands Cheat Sheet

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

Use `mvn deploy` only after the deployment repository and credentials are configured. Run the Sonar Maven goal only after the SonarQube connection/token is configured.

### Service and port checks on Ubuntu

```bash
sudo systemctl status nexus
sudo systemctl status sonarqube
sudo ss -lntp | grep 8081
sudo ss -lntp | grep 9000
sudo ss -lntp | grep 8080
```

Use the actual service names configured on each server.

### Tomcat application checks

```text
http://<TOMCAT_PUBLIC_IP>:8080/tomcat-web-application/
http://<TOMCAT_PUBLIC_IP>:8080/tomcat-web-application/hello
```

### Log review

```bash
sudo journalctl -u nexus --no-pager -n 100
sudo journalctl -u sonarqube --no-pager -n 100
```

Use the matching service name and log path for your installation. Tomcat logs are commonly located under its `logs` directory.

---

## 13. Security and Cleanup Notes

- Never commit passwords, API tokens, private keys, or credential files to GitHub.
- Store SonarQube and Tomcat credentials in Jenkins Credentials.
- Allow only required source IPs and ports in EC2 security groups.
- Restrict Tomcat Manager and Nexus administration access.
- Use HTTPS and suitable authentication for internet-exposed services.
- Avoid broad permissions such as `chmod -R 777`.
- Stop or terminate lab EC2 instances when you are finished if they are not needed. Check whether the instance, EBS volumes, elastic IPs, snapshots, and other resources continue to incur charges.
- Keep a note of current public IP addresses; they can change when instances are stopped and restarted.

---

## Final Summary

This practical demonstrates a simple end-to-end DevOps workflow:

**GitHub → Jenkins + Maven → SonarQube → Nexus → Tomcat**

GitHub stores the source code. Jenkins fetches it into the job workspace and runs the Maven build configured in the Jenkins UI. SonarQube analyzes the code using the server URL configured in `pom.xml` and a token stored in Jenkins Credentials. Nexus manages dependencies and published artifacts, and Tomcat runs the deployed WAR.

Use this README as a repeatable lab guide. Follow the sections in order, verify each tool before moving to the next, and use the troubleshooting table to understand errors instead of changing multiple settings at once.
