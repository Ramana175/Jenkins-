# DevOps Integration Guide: detailed lab notes (GitHub, Jenkins, Maven, SonarQube, Nexus, Tomcat)

A practical guide to building, analyzing, publishing, and deploying a Java web application using DevOps tools.

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
    - [11.1 Case study: Tomcat Manager 403 / 401](#111-case-study-tomcat-manager-403--401-25-26-oct-2026)
12. [Command Cheat Sheet](#12-command-cheat-sheet)
13. [Security and Cleanup](#13-security-and-cleanup)

---

## 1. Project Overview

The goal of this project is to take a Java web application from source code to a running application.

| Tool | Purpose |
|---|---|
| GitHub | Store and version-control source code |
| Jenkins | Automate checkout, build, analysis, and deployment |
| Maven | Compile, test, package, and publish artifacts |
| SonarQube | Analyze source-code quality |
| Nexus Repository Manager | Proxy dependencies and store artifacts |
| Apache Tomcat | Run the deployed Java web application |

The web module used in this practical is:

`tomcat-web-application`

The WAR artifact used in this project is:

`tomcat-web-application-1.0.0.war`

### Project objectives

1. Store Java source code in GitHub.
2. Configure Jenkins to retrieve the source code.
3. Use Maven to build and package the application.
4. Analyze source code with SonarQube.
5. Use Nexus for dependency downloads and artifact storage.
6. Deploy the WAR file to Tomcat.
7. Verify that the deployed application works in a browser.

---

## 2. Workflow and Lab Environment

### 2.1 DevOps workflow

```text
Developer
    |
    v
  GitHub
    |
    v
  Jenkins
    |
    v
  Maven Build
    |
    +-----------> Nexus
    |              |
    |              +-- Download dependencies
    |              +-- Store published artifacts
    |
    +-----------> SonarQube
    |              |
    |              +-- Analyze source code
    |
    v
  WAR Artifact
    |
    v
  Tomcat Deployment
    |
    v
  Running Web Application
```

### 2.2 Lab environment

| Component | Practical details |
|---|---|
| GitHub repository | `https://github.com/Ramana175/Maven` |
| Jenkins job | `DevOps-Integration-Project` |
| Jenkins Maven tool name | `Maven-3.10.0` |
| SonarQube | Community Build, port `9000` |
| Nexus | Version `3.97.1-02`, port `8081` |
| Tomcat | Version `10.1.61`, port `8080` |

**Important:** EC2 public IP addresses can change after an instance is stopped and started. Always check the current IP in the AWS EC2 console before using it.

### 2.3 What happens during a build?

1. Jenkins checks out source code from GitHub.
2. Maven reads `pom.xml`.
3. Maven downloads required dependencies, potentially through Nexus.
4. Maven compiles and packages the application.
5. SonarQube analyzes the source code.
6. Maven can publish the artifact to Nexus if publishing is configured.
7. Jenkins deploys the WAR to Tomcat.
8. The application is tested through its browser URL.

The precise order depends on the Jenkins job configuration.

---

## 3. Why Each Tool Is Connected

Each tool solves a different problem in the delivery process.

### 3.1 GitHub — Source Code Management

GitHub stores the application's source code and version history.

Repository:

`https://github.com/Ramana175/Maven.git`

**Why connect GitHub to Jenkins?**

Jenkins can retrieve the selected branch automatically instead of requiring someone to copy source files manually to the build server.

### 3.2 Jenkins — Automation Controller

Jenkins coordinates the build and deployment workflow.

It can:

- Retrieve code from GitHub.
- Execute Maven commands.
- Use securely stored credentials.
- Trigger SonarQube analysis.
- Publish artifacts through Maven.
- Deploy the WAR to Tomcat.
- Show build logs and failures.

### 3.3 Maven — Build and Dependency Management

Maven reads `pom.xml` to understand the project, dependencies, plugins, and build configuration.

It can:

- Download dependencies.
- Compile Java code.
- Run configured tests.
- Package the web application as a WAR.
- Publish artifacts to Nexus.
- Run the SonarQube Maven scanner.

### 3.4 SonarQube — Code Quality

SonarQube analyzes source code and reports findings such as:

- Bugs
- Vulnerabilities
- Code smells
- Security hotspots
- Quality Gate status

SonarQube analyzes code; it does not replace Maven, Nexus, or Tomcat.

### 3.5 Nexus — Repository Management

Nexus provides repositories for software artifacts and dependencies.

In this project it can:

1. Proxy or group dependency repositories for Maven downloads.
2. Store release and snapshot artifacts uploaded by Maven.

A repository used to download dependencies is not necessarily the same repository used to upload release artifacts.

### 3.6 Tomcat — Application Runtime

Tomcat runs the deployed Java web application packaged as a WAR.

The lab installation is:

`/opt/apache-tomcat-10.1.61`

The HTTP port is `8080`.

### 3.7 Complete integration summary

| Connection | Purpose |
|---|---|
| GitHub → Jenkins | Retrieve source code |
| Jenkins → Maven | Automate the build |
| Maven → Nexus | Download dependencies and publish artifacts |
| Jenkins/Maven → SonarQube | Analyze source code |
| Jenkins → Tomcat | Deploy the WAR |
| Browser → Tomcat | Test the application |

**Remember:** A successful build, successful SonarQube analysis, successful Nexus upload, and successful Tomcat deployment are separate outcomes. Verify each independently.

### 3.8 A small story: why we configure SonarQube, Nexus, and Tomcat

#### The story: the School Project

Ramana makes a school project (the Java code). Before it is shown to everyone, it goes through three places.

**1. SonarQube is the teacher who checks the homework.**
The delivery boy (Maven) must know **where the teacher sits**. That is the SonarQube address (`sonar.host.url`). He also needs a **permission slip** that proves he is allowed to submit homework. That is the SonarQube **token**.

- If the address is wrong, the homework goes to the wrong room and nobody checks it.
- If the slip is shown in public (GitHub), a stranger can use it, so the slip stays in a locked box (Jenkins Credentials).

**2. Nexus is the store room.**
The delivery boy does two jobs here.

- **Take things:** he needs library books (dependencies). He goes to the **front desk** (`<mirrors>` in `settings.xml`).
- **Keep things:** when the project is ready, he puts it on a **shelf** (`<distributionManagement>` in `pom.xml`). Final work goes on the releases shelf, drafts go on the snapshots shelf.
- To put something on a shelf he shows an **ID card** (`<servers>` in `settings.xml`). The name on the ID card must match the name of the shelf.

**3. Tomcat is the exhibition hall.**
The finished project (the WAR file) is shown here to the public. Jenkins is the **courier** who carries the project to the hall.

- The hall's **gate guard** checks the courier's **address** (the `RemoteCIDRValve` allow-list in `context.xml`).
- Then the **room guard** checks the **ID card and role** (`tomcat-users.xml`). Jenkins uses the `manager-script` role, and you use `manager-gui` in the browser.

#### Why do we need to update these?

Because in a cloud lab, **addresses and logins change**.

- An EC2 server gets a **new public IP** after stop/start, so the SonarQube, Nexus, and Tomcat addresses must be updated.
- Your own **mobile IP changes**, so the Tomcat guard blocks you until you add the new one.
- A new **password or role** is needed when you replace a weak one.

#### What happens if we do not update

| Tool | What we configure | Why | If we do not update |
|---|---|---|---|
| SonarQube | `sonar.host.url` in `<properties>`, token in Jenkins | Maven must know where SonarQube is and be allowed in | The scan goes to `localhost` or is rejected, so no analysis appears |
| Nexus (download) | `<mirrors>` in `settings.xml` | Maven must know where to download libraries | Downloads fail or go to the internet instead of Nexus |
| Nexus (upload) | `<distributionManagement>` in `pom.xml` | Maven must know which shelf to use | `mvn deploy` has nowhere to upload |
| Nexus (login) | `<servers>` in `settings.xml` | Only logged-in users may upload | `401 Unauthorized` on upload |
| Tomcat (IP) | `allow` list in `context.xml` | Only trusted IPs may reach the Manager | `403` before the login box appears |
| Tomcat (login) | `tomcat-users.xml` and Jenkins credential `tomcat-manager` | Jenkins and you need the right roles | `401` or `403` on deploy or login |

#### Remember

1. Addresses (IPs) change. Check them first when something stops working.
2. Passwords and tokens stay in Jenkins Credentials or on the server, never on GitHub.
3. Restart the tool after changing its config file (Tomcat needs this).
4. One thing at a time: change one setting, test, then change the next.

---

## 4. GitHub

### 4.1 Repository

```text
https://github.com/Ramana175/Maven.git
```

The branch configured in the Jenkins job is:

`main`

### 4.2 Clone the repository

Run these commands on a machine with Git installed:

```bash
git --version
git clone https://github.com/Ramana175/Maven.git
cd Maven
ls -la
git branch
git remote -v
```

Confirm that the source code and `pom.xml` are present.

### 4.3 Push updated files from Windows PowerShell

Open PowerShell in your local project directory:

```powershell
git status
git add .
git commit -m "Update DevOps project"
git push
```

If there is nothing to commit, there may be no new changes.

If `git push` is rejected because the remote contains commits that are not local, inspect `git status`, branches, and remote history before resolving the merge. Avoid force-pushing without understanding what would be overwritten.

### 4.4 Configure GitHub in Jenkins

1. Open Jenkins.
2. Open `DevOps-Integration-Project`.
3. Select **Configure**.
4. Find **Source Code Management**.
5. Select **Git**.
6. Enter:

   `https://github.com/Ramana175/Maven.git`

7. Set the branch specifier to:

   `*/main`

8. Configure credentials if the repository is private.
9. Save the job.

For a public repository, Jenkins may be able to clone without credentials. For a private repository, use a suitable GitHub credential stored in Jenkins.

Never commit a personal access token to GitHub or put it directly into the repository URL.

---

## 5. Jenkins and Maven

### 5.1 Check the Jenkins service

Run on the Jenkins Ubuntu EC2 server:

```bash
sudo systemctl status jenkins
```

Start Jenkins if it is stopped:

```bash
sudo systemctl start jenkins
```

Enable it at boot:

```bash
sudo systemctl enable jenkins
```

Restart Jenkins when necessary:

```bash
sudo systemctl restart jenkins
```

Check the port:

```bash
sudo ss -lntp | grep ':8080'
```

View recent logs:

```bash
sudo journalctl -u jenkins -n 100 --no-pager
```

Follow logs:

```bash
sudo journalctl -u jenkins -f
```

Press `Ctrl+C` to stop following the logs.

### 5.2 Open Jenkins

```text
http://<JENKINS_PUBLIC_IP>:8080
```

Replace `<JENKINS_PUBLIC_IP>` with the current Jenkins EC2 public IP.

Allow inbound TCP port `8080` only from intended client IPs in the EC2 security group wherever possible.

### 5.3 Configure Maven in Jenkins

In Jenkins:

1. Open **Manage Jenkins**.
2. Open **Tools** or **Global Tool Configuration**, depending on the version.
3. Find **Maven installations**.
4. Configure or select the Maven installation.
5. Use the name:

   `Maven-3.10.0`

6. Save the configuration.

The Maven installation used by the Jenkins job was located at:

```text
/var/lib/jenkins/tools/hudson.tasks.Maven_MavenInstallation/Maven-3.10.0
```

Verify Maven:

```bash
sudo -u jenkins /var/lib/jenkins/tools/hudson.tasks.Maven_MavenInstallation/Maven-3.10.0/bin/mvn -version
```

The practical reported Maven `3.10.0` and Java `21.0.12.1`.

**Important path distinction:** `/opt/apache-maven-3.10.0` belongs to the separate Maven client EC2 lab. Jenkins uses the Jenkins-managed Maven installation shown above.

### 5.4 Jenkins workspace

The workspace used in the practical was:

```text
/var/lib/jenkins/workspace/DevOps-Integration-Project/
```

Inspect it:

```bash
sudo ls -la /var/lib/jenkins/workspace/DevOps-Integration-Project/
```

Jenkins checks out the repository into its workspace. Source files do not have to be stored in a Linux user's home directory.

### 5.5 Configure the Freestyle job

Job name:

`DevOps-Integration-Project`

Under **Source Code Management → Git**:

- Repository URL: `https://github.com/Ramana175/Maven.git`
- Branch specifier: `*/main`

Under **Build Steps**, choose **Invoke top-level Maven targets**:

- Maven Version: `Maven-3.10.0`
- Goals:

```text
clean package deploy org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar
```

Meaning:

- `clean` removes previous build output.
- `package` compiles and packages the application.
- `deploy` publishes the artifact to a Maven repository if publishing is configured.
- The Sonar Maven goal submits code analysis to SonarQube.

Save the job, select **Build Now**, and inspect **Console Output**.

For clearer diagnostics, teams often separate build/publishing, SonarQube analysis, and Tomcat deployment into different stages.

### 5.6 Understand the Maven project structure

A typical Maven web application has a structure similar to:

```text
tomcat-web-application/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/
    │   └── webapp/
    └── test/
        └── java/
```

| Item | Purpose |
|---|---|
| `pom.xml` | Project metadata, dependencies, plugins, build configuration |
| `src/main/java/` | Main Java source code |
| `src/main/webapp/` | Web application files |
| `src/test/java/` | Test source code |
| `target/` | Generated build output, including the WAR |

### 5.7 Important Maven commands

Run these from the directory containing `pom.xml`.

Check project configuration:

```bash
mvn validate
```

Clean previous build output:

```bash
mvn clean
```

Compile:

```bash
mvn compile
```

Run tests:

```bash
mvn test
```

Package:

```bash
mvn package
```

Clean and package:

```bash
mvn clean package
```

Publish the artifact to a configured Maven repository:

```bash
mvn deploy
```

**Important:** `mvn deploy` does not deploy the application to Tomcat. It publishes an artifact to a Maven repository such as a configured Nexus hosted repository. Tomcat deployment is a separate operation.

---

## 6. SonarQube

### 6.1 Check SonarQube

On the SonarQube server:

```bash
sudo systemctl status sonarqube
sudo ss -lntp | grep 9000
```

URL:

```text
http://<SONARQUBE_PUBLIC_IP>:9000
```

Use the current IP and restrict inbound TCP port `9000` to intended sources.

### 6.2 Configure the SonarQube URL

Add the URL inside the existing `<properties>` section of the project's `pom.xml`. Do not create a duplicate `<properties>` block.

```xml
<properties>
    <sonar.host.url>http://SONARQUBE_PUBLIC_IP:9000</sonar.host.url>
</properties>
```

Replace `SONARQUBE_PUBLIC_IP` with the current address.

The URL is configuration; the token is a secret. Do not put the token in `pom.xml` or GitHub.

### 6.3 Create a SonarQube token

1. Sign in to SonarQube.
2. Open the user/account menu.
3. Find **My Account → Security**; labels can vary by version.
4. Generate a token for analysis.
5. Give it a clear name.
6. Copy it immediately and store it securely.
7. Do not commit the token to GitHub or paste it into source code.

Use an appropriate token type and expiry if the installed SonarQube version provides those options.

### 6.4 Store the token in Jenkins Credentials

1. Open Jenkins.
2. Go to **Manage Jenkins → Credentials**.
3. Select the appropriate credentials store, such as **System → Global credentials (unrestricted)**.
4. Select **Add Credentials**.
5. Choose **Kind: Secret text**.
6. Paste the SonarQube token into **Secret**.
7. Use a stable ID such as:

   `sonarqube-token`

8. Use a description such as:

   `SonarQube analysis token`

9. Save the credential.

The practical referred to this credential by the description `SonarQube analysis token`. The actual credential ID must match the ID selected by the job or integration. A description is not necessarily the credential ID.

### 6.5 Configure the SonarQube server in Jenkins

If the SonarQube Scanner for Jenkins plugin is installed:

1. Open **Manage Jenkins → System**.
2. Find **SonarQube servers**.
3. Add a server configuration, for example `SonarQube`.
4. Enter the reachable server URL:

   `http://<SONARQUBE_PUBLIC_IP>:9000`

5. Select the token credential stored in Jenkins.
6. Save.

### 6.6 Pass the token securely to the scanner

Creating a Jenkins credential does not automatically make it available as an environment variable in every build step.

One supported pattern is to use the Jenkins **Credentials Binding** functionality to bind the Secret text credential to:

`SONAR_TOKEN`

Then run the scanner without printing the token.

For example, in a shell step with the credential securely bound as `SONAR_TOKEN`:

```bash
mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar \
  -Dsonar.host.url="$SONAR_HOST_URL" \
  -Dsonar.token="$SONAR_TOKEN"
```

Configure `SONAR_HOST_URL` as a normal environment variable or replace it with the reachable SonarQube URL. Ensure `SONAR_TOKEN` is bound by Jenkins before this command runs.

If the installed Jenkins plugin manages the scanner/server connection, use that plugin's configured server and credential mechanism. Do not assume that simply creating a credential makes `SONAR_TOKEN` available.

Never echo the token or store it in source control.

### 6.7 Run the SonarQube Maven goal

The goal used in the practical was:

```text
org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar
```

After the scan:

1. Open SonarQube.
2. Open the project.
3. Check the latest analysis.
4. Review bugs, vulnerabilities, code smells, and security hotspots.
5. Check the Quality Gate status.

A successful scanner run does not automatically mean the Quality Gate passed.

---

## 7. Nexus Repository Manager

### 7.1 Check Nexus

The practical used Nexus Repository `3.97.1-02`.

Installation directory:

```text
/opt/nexus
```

Data directory:

```text
/opt/sonatype-work/nexus3
```

Check the service:

```bash
sudo systemctl status nexus
```

Check port `8081`:

```bash
sudo ss -lntp | grep 8081
```

Open:

```text
http://<NEXUS_PUBLIC_IP>:8081
```

Use the current public IP and the repository names shown in the Nexus UI.

The group repository used by the Maven client was:

```text
http://<NEXUS_PUBLIC_IP>:8081/repository/maven-public/
```

The Nexus UI showed repositories including Maven Central, release, snapshot, `maven-public`, and `maven-central`.

Do not assume a repository named `public` exists if the UI shows `maven-public`.

### 7.2 Understand `settings.xml`, `pom.xml`, and Nexus

These settings have different responsibilities.

| Configuration | Purpose |
|---|---|
| `settings.xml` → `<mirrors>` | Routes dependency requests through a repository such as Nexus |
| `pom.xml` → `<distributionManagement>` | Specifies where the project publishes release and snapshot artifacts |
| `settings.xml` → `<servers>` | Supplies credentials for repository IDs that require authentication |
| `pom.xml` → dependencies | Declares the libraries the project needs |

Putting the Nexus IP or username/password into `pom.xml` alone does not reliably configure dependency routing and publishing.

A Nexus **group** repository such as `maven-public` is normally used to download or proxy dependencies. A **hosted release** repository and a **hosted snapshot** repository are normally used for uploads.

### 7.3 Configure the Maven mirror

On the **separate Maven client EC2**, the settings file was:

```text
/opt/apache-maven-3.10.0/conf/settings.xml
```

Open it with Vim:

```bash
sudo vim /opt/apache-maven-3.10.0/conf/settings.xml
```

A mirror must be inside the existing `<mirrors>` element.

Example:

```xml
<mirrors>
    <mirror>
        <id>nexus-maven-public</id>
        <mirrorOf>*</mirrorOf>
        <url>http://NEXUS_PUBLIC_IP:8081/repository/maven-public/</url>
    </mirror>
</mirrors>
```

Replace `NEXUS_PUBLIC_IP` with the current address.

This is an example mirror block, not a complete replacement for an existing `settings.xml`. Preserve other settings and avoid duplicate root elements.

The meaning of the configuration:

- `<id>` identifies the mirror.
- `<mirrorOf>*</mirrorOf>` routes requests for all repositories through this mirror.
- `<url>` points to the Nexus group repository.

A mirror controls dependency download routing. It does not, by itself, configure artifact publishing.

### 7.4 Configure artifact publishing in `pom.xml`

To use `mvn deploy`, configure `distributionManagement` in the project's `pom.xml`.

Example structure only:

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

Replace the placeholder IP and repository names with the actual hosted repository URLs shown in Nexus.

Do not use the `maven-public` group URL as the upload target unless the repository is specifically configured for deployments.

### 7.5 Configure Nexus credentials in `settings.xml`

If the hosted repositories require authentication, add matching server entries to the `<servers>` section of the Maven settings file.

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

The `<server><id>` values must match the repository IDs in `distributionManagement` exactly.

This is a structural example, not a complete `settings.xml`. Preserve existing settings and keep secrets out of GitHub.

For shared or production environments, use an appropriately protected credentials mechanism and HTTPS.

The mirror ID `nexus-maven-public` and upload IDs `nexus-releases` / `nexus-snapshots` have different purposes.

### 7.6 Release versus snapshot

A version such as:

`1.0.0`

is a release version.

A version such as:

`1.0.1-SNAPSHOT`

is a snapshot version.

Maven selects the matching destination from `distributionManagement`. Ensure the relevant hosted repositories exist in Nexus and the deployment user has permission to publish.

### 7.7 Verify Nexus configuration

First check that the Nexus service is running:

```bash
sudo systemctl status nexus
```

Check port `8081`:

```bash
sudo ss -lntp | grep 8081
```

From a Maven client that can reach Nexus, run:

```bash
mvn -U validate
```

A successful `validate` checks basic project configuration; it does **not** prove that the artifact was uploaded.

To publish, run:

```bash
mvn deploy
```

only after the repository destination, credentials, and permissions are configured.

### 7.8 Nexus troubleshooting

- **`Unrecognised tag: mirror`:** the `<mirror>` element is outside `<mirrors>`.
- **XML mismatch:** check XML nesting and make sure there is only one `<settings>` root element.
- **Dependency resolution fails:** verify the mirror URL, Nexus service, network access, and Maven logs.
- **`mvn validate` succeeds but nothing appears in Nexus:** `validate` does not upload artifacts. Check `distributionManagement` and run `mvn deploy`.
- **Authentication failure:** check the credentials and matching repository ID.
- **Anonymous downloads work but uploads fail:** anonymous read access does not necessarily permit publishing. Use an authorized account for writes.
- **Connection timeout:** check the Nexus EC2 public IP, network route, security group, and whether port `8081` is listening.
- **Maven warning about `credentialScope` / `repositoryOrigins`:** this is a warning, not an error. See [11.1](#111-case-study-tomcat-manager-403--401-25-26-oct-2026).

The lab used HTTP for practice. Prefer HTTPS for internet-exposed or production environments and restrict administrative access.

---

## 8. Tomcat Deployment

### 8.1 Tomcat installation

The practical used Tomcat `10.1.61` at:

```text
/opt/apache-tomcat-10.1.61
```

The default HTTP port was `8080`.

### 8.2 Check the installed Tomcat version

```bash
sudo /opt/apache-tomcat-10.1.61/bin/catalina.sh version
```

This confirms the installed Tomcat version and Java runtime.

### 8.3 Check whether port 8080 is listening

```bash
sudo ss -lntp | grep 8080
```

If a `LISTEN` entry appears, something is listening on port `8080`.

If there is no output, check the process:

```bash
ps -ef | grep '[o]rg.apache.catalina.startup.Bootstrap'
```

### 8.4 Start Tomcat

The practical used a manual installation, and there was no configured `tomcat.service` systemd unit. Therefore, `Unit tomcat.service could not be found` did not by itself mean Tomcat was broken.

Start Tomcat:

```bash
sudo /opt/apache-tomcat-10.1.61/bin/startup.sh
```

Then check the port:

```bash
sudo ss -lntp | grep 8080
```

Test the local HTTP endpoint:

```bash
curl -I --connect-timeout 5 http://localhost:8080/
```

An HTTP response confirms that an HTTP server responded. Check the actual status and application URL to verify the expected page.

### 8.5 Check Tomcat logs

```bash
sudo tail -n 100 /opt/apache-tomcat-10.1.61/logs/catalina.out
```

List log files:

```bash
ls -lah /opt/apache-tomcat-10.1.61/logs/
```

If startup fails, inspect the logs before making further configuration changes.

Use `systemctl status <actual-service-name>` only if you have configured a systemd unit.

### 8.6 Application URLs

WAR file:

```text
tomcat-web-application/target/tomcat-web-application-1.0.0.war
```

Application home page:

```text
http://<TOMCAT_PUBLIC_IP>:8080/tomcat-web-application/
```

Servlet endpoint:

```text
http://<TOMCAT_PUBLIC_IP>:8080/tomcat-web-application/hello
```

Replace `<TOMCAT_PUBLIC_IP>` with the current Tomcat EC2 public IP.

The practical home page displayed:

`Welcome to My Tomcat Web Application!`

The `/hello` endpoint responded:

`Hello from my Tomcat web application!`

### 8.7 Tomcat Manager configuration

Relevant files:

```text
/opt/apache-tomcat-10.1.61/webapps/manager/META-INF/context.xml
/opt/apache-tomcat-10.1.61/conf/tomcat-users.xml
```

Tomcat Manager commonly restricts remote access through a `RemoteCIDRValve`.

Do not remove the restriction or expose Manager to everyone. Allow only the verified client IPs or the required private network.

The Jenkins credential used in this practical was:

- Credential ID: `tomcat-manager`
- Username: `tomcat`
- Type: Username with password

Keep the real password in Jenkins Credentials.

For Tomcat Manager's text API, the `manager-script` role is commonly required. The browser GUI generally uses `manager-gui`. Grant only the role needed for the deployment mechanism. Use separate users for the two roles (see [11.1](#111-case-study-tomcat-manager-403--401-25-26-oct-2026)).

Changing an IP address alone does not require changing the username or password.

### 8.8 Configure the Tomcat Manager IP allow-list

Use this procedure if your computer or Jenkins can no longer access Tomcat Manager after an EC2 stop/start, or Jenkins deployment is rejected.

EC2 public IP addresses may change after stop/start unless an Elastic IP is used. Check current addresses in the EC2 console.

**Step 1 — Get your computer's current public IP**

In Windows PowerShell:

```powershell
(Invoke-RestMethod -Uri "https://checkip.amazonaws.com").Trim()
```

Use this IP only if you need to access Tomcat Manager from your computer.

For IPv4, `/32` means one exact address.

**Step 2 — Identify Jenkins' actual client IP**

Check the current Jenkins EC2 public IP and network configuration.

If Jenkins uses a NAT gateway or another egress route, its outbound IP may differ from the instance's public IP. The allow-list must match the address Tomcat actually sees as the client.

Where practical, prefer private networking and security-group references.

**Step 3 — Back up the Manager context file**

```bash
sudo cp /opt/apache-tomcat-10.1.61/webapps/manager/META-INF/context.xml \
  /opt/apache-tomcat-10.1.61/webapps/manager/META-INF/context.xml.bak
```

**Step 4 — Edit the file using Vim**

```bash
sudo vim /opt/apache-tomcat-10.1.61/webapps/manager/META-INF/context.xml
```

Find the `RemoteCIDRValve` and its `allow` attribute.

Example format only:

```xml
<Valve className="org.apache.catalina.valves.RemoteCIDRValve"
       allow="127.0.0.0/8,::1,REPLACE_WITH_YOUR_PUBLIC_IP/32,REPLACE_WITH_JENKINS_EGRESS_IP/32" />
```

Replace the placeholders with current, verified client addresses. Retain required localhost entries from the existing configuration.

Do not paste the placeholders literally. Do not add the Tomcat server's own public IP just because it changed; the allow-list checks the connecting client.

Do not use `0.0.0.0/0` for convenience.

Keep only one `allow=` attribute in the `Valve`. A duplicate attribute is invalid XML.

**Step 5 — Save in Vim**

Press `Esc`, type:

```text
:wq
```

Then press `Enter`.

**Step 6 — Verify the saved configuration**

```bash
sudo grep -n -A 2 'RemoteCIDRValve' /opt/apache-tomcat-10.1.61/webapps/manager/META-INF/context.xml
sudo grep -n 'allow=' /opt/apache-tomcat-10.1.61/webapps/manager/META-INF/context.xml
```

**Step 7 — Check `tomcat-users.xml` if needed**

```bash
sudo vim /opt/apache-tomcat-10.1.61/conf/tomcat-users.xml
```

Confirm that the deployment user has the role required by the deployment mechanism.

Changing the allow-list does not require changing the username or password.

Never paste the password into GitHub or chat.

**Step 8 — Restart Tomcat if required**

Only restart when configuration changes need it or a restart is otherwise necessary:

```bash
sudo /opt/apache-tomcat-10.1.61/bin/shutdown.sh
sudo /opt/apache-tomcat-10.1.61/bin/startup.sh
sudo ss -lntp | grep 8080
curl -I --connect-timeout 5 http://localhost:8080/
```

If shutdown reports that Tomcat is not running, verify the process and logs before taking further action. A restart can briefly interrupt the application.

### 8.9 Configure Jenkins post-build deployment

In the Jenkins job, configure the appropriate post-build actions.

**Archive artifacts**

```text
tomcat-web-application/target/*.war
```

**Deploy WAR/EAR to a container**

| Setting | Value |
|---|---|
| WAR/EAR files | `tomcat-web-application/target/*.war` |
| Context path | `tomcat-web-application` |
| Adapter used in the practical | `Tomcat 9.x Remote` |
| Credentials | `tomcat-manager` |
| Tomcat URL | `http://<TOMCAT_PUBLIC_IP>:8080/` |

The practical used `Tomcat 9.x Remote` even though the server was Tomcat `10.1.61`. Check compatibility when using other plugin and server versions.

After deployment, open the home page and `/hello` endpoint to verify the application.

### 8.10 Tomcat deployment troubleshooting

**Connection timeout**

Check:

- Current Tomcat public IP.
- Whether Tomcat is listening on port `8080`.
- Tomcat EC2 security-group inbound TCP `8080` rule.
- Network ACLs and routing.
- Whether Jenkins can reach the Tomcat endpoint.

Restrict the rule to Jenkins' actual source wherever possible.

**HTTP 403 from Manager**

Check the `RemoteCIDRValve` allow-list and the client IP Tomcat sees.

**HTTP 401 or authentication failure**

Check the Jenkins credential ID, username/password, and required Tomcat role.

**Tomcat opens, but the application returns 404**

Check that the WAR deployed successfully and use the correct context path:

`/tomcat-web-application/`

**Tomcat does not start**

Inspect:

```text
/opt/apache-tomcat-10.1.61/logs/catalina.out
```

Also inspect other files in the `logs` directory.

**Permission denied reading deployed files**

Inspect ownership and permissions:

```bash
ls -l /opt/apache-tomcat-10.1.61/webapps/
namei -l /opt/apache-tomcat-10.1.61/webapps/tomcat-web-application/index.html
```

Do not use `chmod -R 777` as a general fix.

The Tomcat `context.xml` allow-list, `tomcat-users.xml` roles, Jenkins credentials, and EC2 security-group rules solve different problems. Diagnose the error before changing configuration.

---

## 9. Final Jenkins Job Configuration

| Setting | Value |
|---|---|
| Job type | Freestyle |
| Job name | `DevOps-Integration-Project` |
| Repository | `https://github.com/Ramana175/Maven.git` |
| Branch | `*/main` |
| Maven build step | Invoke top-level Maven targets |
| Maven installation | `Maven-3.10.0` |
| Maven goals used | `clean package deploy org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar` |
| SonarQube credential description | `SonarQube analysis token` |
| WAR archive pattern | `tomcat-web-application/target/*.war` |
| Deployment context | `tomcat-web-application` |
| Tomcat credential ID | `tomcat-manager` |

`deploy` publishes to Nexus only if Maven publishing is configured. Check the SonarQube Quality Gate separately from scanner success.

The practical's successful console output showed SonarScanner completion, `BUILD SUCCESS`, artifact archiving, and deployment of `tomcat-web-application-1.0.0.war`.

### Important networking note

A build can compile and package successfully but fail while connecting to Nexus or Tomcat.

For example, a connection timeout to:

`http://<NEXUS_PUBLIC_IP>:8081/repository/maven-releases/`

does not automatically mean the Maven project is wrong. Check the current IP, service status, listening port, routing, and EC2 security-group rules first.

---

## 10. End-to-End Verification

Use this checklist after configuring the complete project.

- [ ] Latest source code and `pom.xml` are visible in GitHub.
- [ ] Jenkins checks out the correct repository and `main` branch.
- [ ] Jenkins selects Maven installation `Maven-3.10.0`.
- [ ] Maven compiles the application successfully.
- [ ] Maven creates the WAR in the correct module's `target` directory.
- [ ] SonarQube shows the latest analysis.
- [ ] SonarQube Quality Gate status has been checked.
- [ ] Nexus service is running.
- [ ] Nexus repository names and URLs match the Nexus UI.
- [ ] Maven mirror configuration is valid.
- [ ] If publishing, `distributionManagement` and credentials are configured.
- [ ] If publishing, the artifact appears in the intended hosted repository.
- [ ] Jenkins archives the WAR.
- [ ] Jenkins deploys the WAR to Tomcat.
- [ ] Tomcat is listening on port `8080`.
- [ ] Application home page loads in the browser.
- [ ] `/hello` returns the expected response.

---

## 11. Troubleshooting

| Problem | What to check |
|---|---|
| Maven not found | Jenkins Tools configuration and selected Maven installation |
| Files missing from Jenkins workspace | Inspect `/var/lib/jenkins/workspace/DevOps-Integration-Project/` |
| WAR not found | Correct module, `target` directory, and version in `pom.xml` |
| Git push rejected | `git status`, branch, remote commits, and merge state |
| Jenkins dashboard unavailable | Service status, port `8080`, current IP, and security group |
| SonarQube UI unavailable | Service, port `9000`, current IP, security group, and logs |
| SonarQube authentication fails | Valid token, correct Secret text credential, token binding, scanner arguments, and reachable server URL |
| Quality Gate not green | Inspect Quality Gate conditions and findings in SonarQube |
| Nexus mirror XML error | Put `<mirror>` inside `<mirrors>` and check XML nesting |
| Maven dependency resolution fails | Check `settings.xml`, mirror URL, Nexus status, network access, and Maven logs |
| Maven uses a cached failed download | After correcting the root cause, retry with `mvn -U` |
| `validate` succeeds but nothing was uploaded | `validate` does not publish artifacts; configure `distributionManagement` and run `mvn deploy` |
| Nexus authentication fails | Check repository ID matching, credentials, and write permissions |
| Nexus connection timeout | Check current IP, service status, listening port, routing, and security groups |
| Tomcat connection timeout | Current IP, port `8080`, listener, routing, and security group |
| Tomcat Manager returns 403 | Check the Manager IP allow-list and actual client IP |
| Tomcat Manager returns 401 | Check Jenkins credential, username/password, and required role |
| Tomcat application returns 404 | Confirm deployment succeeded and use the correct context path |
| Tomcat startup fails | Inspect `catalina.out` and the Tomcat logs directory |
| Permission denied reading a file | Use `ls -l` and `namei -l <path>`; check ownership and permissions |
| `Unit tomcat.service not found` | Tomcat was installed manually. Use `startup.sh` and `shutdown.sh` from the Tomcat `bin` folder |
| Manager returns 403 and the access log shows a username | The user has the wrong role. Use `manager-gui` for the browser |
| Manager returns 403 and the access log shows `-` | The IP check failed. Update the `allow` list with the current IP |
| Browser keeps failing after the user was changed | Cached login. Use a new incognito window |
| `shutdown.sh: command not found` | Wrong directory. `cd` into `/opt/apache-tomcat-10.1.61/bin` |
| Maven warning about `credentialScope` / `repositoryOrigins` | Warning only. Optionally declare the Nexus origin in `settings.xml` |

Change one thing at a time. Read the full error output and run the smallest useful test before changing additional configuration.

### 11.1 Case study: Tomcat Manager 403 / 401 (25-26 Oct 2026)

**Setup:** Tomcat `10.1.61` installed manually at `/opt/apache-tomcat-10.1.61` on EC2, Manager opened from a browser and used by Jenkins ("Deploy WAR/EAR to a container").

**Result:** The Manager browser page was fixed, and the Jenkins pipeline finished with `BUILD SUCCESS` and `Finished: SUCCESS`. The WAR was undeployed and redeployed by Jenkins.

#### The two checks behind Tomcat Manager

Tomcat Manager has two separate "doors". Both must allow the request.

| Door | What it checks | Where it is configured | Failure shows as |
|---|---|---|---|
| 1. IP check | Is the client IP in the allow-list? | `webapps/manager/META-INF/context.xml` (`RemoteCIDRValve`) | `403`, user column is `-` |
| 2. User and role | Does the logged-in user have the right role? | `conf/tomcat-users.xml` | `401` (bad login) or `403` with a username shown |

Roles:

| Role | Used for |
|---|---|
| `manager-gui` | Browser page `/manager/html` |
| `manager-script` | Programs such as Jenkins (`/manager/text`) |

Use separate users for the two roles. Do not give one user both roles.

#### Day 1 (25 Oct): browser 403 and 401

| # | Problem | Cause | Fix |
|---|---|---|---|
| 1 | `Unit tomcat.service not found` | Tomcat was installed manually, no systemd unit exists | Use `bin/shutdown.sh` and `bin/startup.sh` from `/opt/apache-tomcat-10.1.61/bin` |
| 2 | `403` on `/manager/html` after the IP was added | The browser logged in as user `tomcat`, which had only `manager-script` | Use a separate user with `manager-gui` for the browser |
| 3 | Browser never asked for a new login | The browser cached the old credentials | Close all incognito windows and open a new one |
| 4 | `403` when using `curl` on the server | `curl` ran on the EC2 server, so Tomcat saw the server's own IP, which is not in the allow-list | Test from the laptop, not from the server |
| 5 | `ssh` run inside the EC2 session failed with `Permission denied (publickey)` | The command ran on the server, with a placeholder key path | Start SSH tunnels from the laptop (for example MobaXterm) |
| 6 | `syntax error near unexpected token` | XML lines were pasted into the terminal | XML belongs inside the file, not in the shell |
| 7 | `401` after editing `tomcat-users.xml` | Two users had the same name, a placeholder (`yourusername`) was used literally, and Tomcat was not restarted | One user per name, replace placeholders, restart Tomcat |
| 8 | Weak passwords | Guessable passwords on a public port | Use long random passwords, for example `openssl rand -hex 12` |

#### Day 2 (26 Oct): IP change and config typo

| # | Problem | Cause | Fix |
|---|---|---|---|
| 9 | `sudo: ./shutdown.sh: command not found` | Wrong directory: `./` runs a file from the current folder | `cd /opt/apache-tomcat-10.1.61/bin` first, or use the full path |
| 10 | `403` with `-` in the user column, from a new IP | The mobile public IP changed, so the valve blocked it | Find the new IP and update the `allow` list |
| 11 | Two `allow=` lines in `context.xml` | A paste mistake duplicated the attribute, which is invalid XML | Keep one `allow=` line only |
| 12 | Log showed `Stopping service` | The log was read before `startup.sh` finished | Run `ps -ef \| grep -i [t]omcat`, then check `catalina.out` for `Server startup in [...] milliseconds` |

#### How to find the current public IP

On the laptop, in the same browser or network used for the Manager:

```powershell
(Invoke-RestMethod -Uri "https://checkip.amazonaws.com").Trim()
```

Or open `https://checkip.amazonaws.com`. Do not use the Wi-Fi settings address (`192.168.x.x`), because it is private.

You can also read it from Tomcat: the first field of a line in the access log is the client IP.

```bash
sudo tail -n 3 /opt/apache-tomcat-10.1.61/logs/localhost_access_log.<YYYY-MM-DD>.txt
```

#### Correct configuration (placeholders only)

`conf/tomcat-users.xml`:

```xml
<role rolename="manager-script"/>
<role rolename="manager-gui"/>
<user username="JENKINS_USER" password="STRONG_PASSWORD_1" roles="manager-script"/>
<user username="BROWSER_USER" password="STRONG_PASSWORD_2" roles="manager-gui"/>
```

`webapps/manager/META-INF/context.xml`:

```xml
<Valve className="org.apache.catalina.valves.RemoteCIDRValve"
       allow="127.0.0.0/8,YOUR_CURRENT_IP/32,JENKINS_IP/32" />
```

Keep the Jenkins IP in the list, or Jenkins cannot deploy. If the Jenkins password changes, update the Jenkins credential (`tomcat-manager`) as well.

Restart Tomcat after every edit to either file:

```bash
cd /opt/apache-tomcat-10.1.61/bin
sudo ./shutdown.sh && sleep 3 && sudo ./startup.sh
tail -n 3 /opt/apache-tomcat-10.1.61/logs/catalina.out
```

#### How to read the access log

| Log line shows | Meaning | Action |
|---|---|---|
| `IP - user ... 200` | Success | None |
| `IP - - ... 401` | The IP is allowed, but the login was rejected | Retype the credentials, check the file, restart |
| `IP - user ... 403` | The IP is allowed, but the role is wrong | Use a `manager-gui` user for the browser |
| `IP - - ... 403` | The IP check blocked the request | Update the `allow` list with the current IP |
| Server's own IP `... 403` | The test ran on the server | Test from the laptop |
| No new line | The request did not reach Tomcat | Check the IP and the EC2 security group for port `8080` |

#### Maven warning seen in the Jenkins build

```text
[WARNING] Using credentials of server 'nexus' for repository http://<NEXUS_IP>:8081/...
although no repository or mirror with this id is declared ...
```

This is a warning, not an error. Maven matched the `nexus` credentials by server ID only, not by address. The build still finished with `BUILD SUCCESS` and the upload to Nexus worked. To remove the warning, declare the Nexus address in `<repositoryOrigins>` for that server in `settings.xml` (check the Maven settings reference for your version). Do not set `maven.repository.credentialScope=strict` until the origin is declared, or the upload will be refused.

#### Lessons learned

1. Test from the client machine, not from the server.
2. Keep separate users for `manager-gui` and `manager-script`.
3. Restart Tomcat after changing `tomcat-users.xml` or `context.xml`.
4. Mobile and home IPs change. Re-check the IP before editing the allow-list.
5. Read the access log first: the user column and the status code show which door blocked the request.
6. Public ports on Tomcat (`8080`), Nexus (`8081`) and SonarQube (`9000`) attract scanners. Restrict them in the security group to your IP and the Jenkins IP.
7. Never paste passwords into chats, screenshots or GitHub.

---

## 12. Command Cheat Sheet

### 12.1 Git — Windows PowerShell

```powershell
git status
git remote -v
git branch
git add .
git commit -m "Describe the change"
git push
```

### 12.2 Maven

```bash
mvn -version
mvn validate
mvn clean package
mvn -U validate
```

Use `mvn deploy` only after publishing configuration is correct.

### 12.3 Jenkins service

```bash
sudo systemctl status jenkins
sudo systemctl start jenkins
sudo systemctl restart jenkins
sudo systemctl enable jenkins
sudo journalctl -u jenkins -n 100 --no-pager
```

### 12.4 SonarQube service

```bash
sudo systemctl status sonarqube
sudo ss -lntp | grep 9000
sudo journalctl -u sonarqube -n 100 --no-pager
```

### 12.5 Nexus service

```bash
sudo systemctl status nexus
sudo ss -lntp | grep 8081
sudo journalctl -u nexus -n 100 --no-pager
```

### 12.6 Tomcat

```bash
sudo /opt/apache-tomcat-10.1.61/bin/catalina.sh version
sudo ss -lntp | grep 8080
ps -ef | grep '[o]rg.apache.catalina.startup.Bootstrap'
sudo tail -n 100 /opt/apache-tomcat-10.1.61/logs/catalina.out
curl -I --connect-timeout 5 http://localhost:8080/
```

Restart Tomcat (manual installation):

```bash
cd /opt/apache-tomcat-10.1.61/bin
sudo ./shutdown.sh && sleep 3 && sudo ./startup.sh
```

---

## 13. Security and Cleanup

- Never commit passwords, tokens, private keys, or credential files to GitHub.
- Store SonarQube and Tomcat credentials in Jenkins Credentials.
- Do not print secrets in Jenkins console output.
- Restrict EC2 security groups to the required sources and ports.
- Restrict Tomcat Manager and Nexus administration/publishing access.
- Prefer HTTPS and appropriate authentication for internet-exposed services.
- Avoid broad permissions such as `chmod -R 777`.
- Use current public IP addresses; they may change after stop/start.
- Prefer private networking or stable, controlled addresses where appropriate.
- Use strong random passwords for Tomcat Manager users, and do not run Tomcat as root where avoidable.
- Stop or terminate lab resources when they are not needed.
- Check EC2 instances, EBS volumes, Elastic IPs, snapshots, and other resources for ongoing charges.

---

## Final Summary

- **GitHub** stores the source code.
- **Jenkins** automates the workflow.
- **Maven** builds and packages the application.
- **SonarQube** analyzes source-code quality.
- **Nexus** manages dependencies and published artifacts.
- **Tomcat** runs the deployed WAR.

The complete workflow is successful only when source retrieval, build, code analysis, artifact publishing (if configured), and application deployment have each been verified.
