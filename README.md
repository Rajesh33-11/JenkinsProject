# Jenkins CI/CD Pipeline: JSON Config Update + Git Push + Maven Build + SonarQube + Nexus

A parameterized Jenkins declarative pipeline that:

1. Validates user inputs
2. Checks out the repository from GitHub
3. Updates one or more environment JSON files (`node/dev.json`, `node/prod.json`, `node/stage.json`, `node/uat.json`) using `jq`
4. Shows the Git diff for review
5. Commits and pushes the changes back to GitHub
6. Builds the project with Maven
7. Runs SonarQube static code analysis
8. Deploys the artifact to a Nexus Repository
9. Verifies and archives the build artifact in Jenkins

---

## Table of Contents

1. [Architecture](#1-architecture)
2. [Prerequisites](#2-prerequisites)
3. [Repository Structure](#3-repository-structure)
4. [Step 1: Install Java and Basic Tools](#4-step-1-install-java-and-basic-tools)
5. [Step 2: Install and Configure Jenkins](#5-step-2-install-and-configure-jenkins)
6. [Step 3: Install and Configure SonarQube](#6-step-3-install-and-configure-sonarqube)
7. [Step 4: Install and Configure Nexus](#7-step-4-install-and-configure-nexus)
8. [Step 5: Configure the Maven Project (pom.xml)](#8-step-5-configure-the-maven-project-pomxml)
9. [Step 6: Configure Jenkins (Tools, Credentials, SonarQube Server)](#9-step-6-configure-jenkins-tools-credentials-sonarqube-server)
10. [Step 7: Create the Jenkins Pipeline Job](#10-step-7-create-the-jenkins-pipeline-job)
11. [Pipeline Parameters](#11-pipeline-parameters)
12. [Stage-by-Stage Explanation](#12-stage-by-stage-explanation)
13. [Sample JSON File](#13-sample-json-file)
14. [How to Run the Pipeline](#14-how-to-run-the-pipeline)
15. [Verifying the Results](#15-verifying-the-results)
16. [Troubleshooting](#16-troubleshooting)
17. [Known Limitations and Recommended Improvements](#17-known-limitations-and-recommended-improvements)
18. [Security Best Practices](#18-security-best-practices)

---

## 1. Architecture

```
                 +-----------------+
   User  ----->  |  Jenkins (8080) |
 (parameters)    +--------+--------+
                          |
     +--------------------+----------------------+
     |                    |                      |
     v                    v                      v
+---------+        +--------------+        +--------------+
| GitHub  |        | SonarQube    |        | Nexus        |
| (repo)  |        | (9000)       |        | (8081)       |
+---------+        +--------------+        +--------------+
 JSON update        Code quality            Artifact storage
 + push             analysis                (maven-releases1)
```

**Flow:**

```
Validate Inputs -> Checkout -> Update JSON -> Review Changes -> Commit & Push
      -> Maven Build -> SonarQube Analysis -> Nexus Upload -> Check Artifact -> Archive Artifact
```

---

## 2. Prerequisites

| Component | Version / Requirement | Purpose |
|---|---|---|
| OS | Ubuntu 22.04 / 24.04 (or any Linux) | Host for Jenkins, SonarQube, Nexus |
| Java (JDK) | 17 | Jenkins, Maven, SonarQube |
| Jenkins | LTS (2.440+) | CI/CD server |
| Maven | 3.8+ | Build tool |
| Git | 2.x | Source control |
| jq | 1.6+ | JSON editing |
| SonarQube | Community Edition 10.x | Code quality |
| Nexus Repository | 3.x | Artifact repository |
| GitHub account | Personal Access Token (PAT) | Git push from Jenkins |

**Recommended hardware (minimum):**

| Server | CPU | RAM | Disk |
|---|---|---|---|
| Jenkins | 2 vCPU | 4 GB | 30 GB |
| SonarQube | 2 vCPU | 4 GB (8 GB better) | 20 GB |
| Nexus | 2 vCPU | 4 GB | 50 GB |

**Network ports to open (Security Group / firewall):**

| Port | Service |
|---|---|
| 22 | SSH |
| 8080 | Jenkins |
| 9000 | SonarQube |
| 8081 | Nexus |

---

## 3. Repository Structure

```
flipkart/
|-- Jenkinsfile          # This pipeline
|-- pom.xml              # Maven project file
|-- README.md            # This documentation
|-- src/                 # Application source code
`-- node/
    |-- dev.json
    |-- stage.json
    |-- uat.json
    `-- prod.json
```

> The files in `node/` **must already exist** in the repository. The pipeline fails if a selected file is missing.

---

## 4. Step 1: Install Java and Basic Tools

Run on the Jenkins server (and on the SonarQube/Nexus servers wherever Java is needed).

```bash
sudo apt update && sudo apt upgrade -y

# Java 17
sudo apt install -y openjdk-17-jdk

# Git, jq, curl, unzip
sudo apt install -y git jq curl unzip wget

# Verify
java -version
git --version
jq --version
```

> **Why jq?** The `Update JSON` stage uses `jq` to edit JSON files. Jenkins must be able to run `jq` on the agent, so install it on every agent that runs this pipeline.

---

## 5. Step 2: Install and Configure Jenkins

### 5.1 Install Jenkins (Ubuntu / Debian)

```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre
java -version

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins

sudo apt update
sudo apt install -y jenkins

sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

### 5.2 First-time Setup

1. Open `http://<JENKINS_SERVER_IP>:8080`
2. Get the initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Paste it, choose **Install suggested plugins**.
4. Create the admin user and confirm the Jenkins URL.

### 5.3 Install Required Plugins

Go to **Manage Jenkins -> Plugins -> Available plugins** and install:

| Plugin | Used For |
|---|---|
| Pipeline | Declarative pipeline support |
| Git | `git` step |
| Credentials Binding | `withCredentials` |
| Workspace Cleanup | `cleanWs()` |
| SonarQube Scanner | `withSonarQubeEnv` |
| Maven Integration | Maven tool |
| Pipeline: Stage View (optional) | Visualisation |

Restart Jenkins after installation if prompted.

---

## 6. Step 3: Install and Configure SonarQube

### 6.1 Prepare the Server (required kernel settings)

```bash
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072

# Make permanent
echo "vm.max_map_count=524288" | sudo tee -a /etc/sysctl.conf
echo "fs.file-max=131072"      | sudo tee -a /etc/sysctl.conf
```

### 6.2 Install using Docker (easiest)

```bash
sudo apt install -y docker.io
sudo systemctl enable --now docker

sudo docker volume create sonarqube_data
sudo docker volume create sonarqube_extensions
sudo docker volume create sonarqube_logs

sudo docker run -d --name sonarqube --restart unless-stopped \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  -v sonarqube_logs:/opt/sonarqube/logs \
  sonarqube:community
```

Wait 1-2 minutes, then open `http://<SONARQUBE_IP>:9000`.

### 6.3 Initial Login

- Username: `admin`
- Password: `admin` (you will be forced to change it)

### 6.4 Generate a Token for Jenkins

1. SonarQube -> top-right user icon -> **My Account -> Security**
2. Under **Generate Tokens**: Name = `jenkins-token`, Type = *User Token*, Expiry as needed
3. Click **Generate** and **copy the token immediately** (it is shown only once).

### 6.5 (Optional) Create the Project Manually

Project key and name used by this pipeline: `flipkart`. SonarQube creates the project automatically on first analysis if the token has permission.

---

## 7. Step 4: Install and Configure Nexus

### 7.1 Install using Docker

```bash
sudo apt install -y docker.io
sudo systemctl enable --now docker

sudo docker volume create nexus-data

sudo docker run -d --name nexus --restart unless-stopped \
  -p 8081:8081 \
  -v nexus-data:/nexus-data \
  sonatype/nexus3
```

Startup takes 2-3 minutes. Open `http://<NEXUS_IP>:8081`.

### 7.2 Get the Initial Admin Password

```bash
sudo docker exec nexus cat /nexus-data/admin.password
```

Login as `admin`, follow the wizard, and set a new password.

### 7.3 Create the Hosted Maven Repository

1. Gear icon -> **Repository -> Repositories -> Create repository**
2. Choose **maven2 (hosted)**
3. Settings:
   - **Name:** `maven-releases1` (must match the name used in the pipeline/pom.xml)
   - **Version policy:** `Release`
   - **Layout policy:** `Strict`
   - **Deployment policy:** `Allow redeploy` (recommended for CI, see note below)
4. Click **Create repository**

> **Important:** With the default policy **Disable redeploy**, running the pipeline twice with the same version in `pom.xml` fails with `400 Bad Request` / `Cannot update component version`. Either bump the version on every build or set the policy to *Allow redeploy*.

### 7.4 Create a Deployment User (recommended)

1. **Security -> Roles -> Create role** -> Nexus role -> ID `maven-deployer`
   - Privileges: `nx-repository-view-maven2-maven-releases1-*`
2. **Security -> Users -> Create local user** -> ID `jenkins-deployer`, assign role `maven-deployer`

---

## 8. Step 5: Configure the Maven Project (pom.xml)

### 8.1 Add `distributionManagement`

The `<id>` **must be exactly `nexus-releases`**, as it matches the `<server><id>` generated by the pipeline's temporary `settings.xml`.

```xml
<distributionManagement>
    <repository>
        <id>nexus-releases</id>
        <url>http://<NEXUS_IP>:8081/repository/maven-releases1/</url>
    </repository>
</distributionManagement>
```

### 8.2 Set Packaging and Version

```xml
<groupId>com.example</groupId>
<artifactId>flipkart</artifactId>
<version>1.0.0</version>
<packaging>jar</packaging>   <!-- or war -->
```

> The version **must not end in `-SNAPSHOT`** because the target repository is a *Release* repository.

### 8.3 (Optional) Sonar Properties

```xml
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <sonar.projectKey>flipkart</sonar.projectKey>
    <sonar.projectName>flipkart</sonar.projectName>
</properties>
```

---

## 9. Step 6: Configure Jenkins (Tools, Credentials, SonarQube Server)

### 9.1 Configure JDK and Maven Tools

**Manage Jenkins -> Tools**

| Section | Name (exact) | Setting |
|---|---|---|
| JDK installations | `JDK17` | Uncheck *Install automatically*; JAVA_HOME = `/usr/lib/jvm/java-17-openjdk-amd64` |
| Maven installations | `Maven3` | Check *Install automatically*, choose Maven 3.9.x |

> Names are case-sensitive and must match the `tools { jdk 'JDK17'  maven 'Maven3' }` block in the Jenkinsfile.

### 9.2 Add Credentials

**Manage Jenkins -> Credentials -> System -> Global credentials -> Add Credentials**

| ID (exact) | Kind | Username | Password / Secret | Used In |
|---|---|---|---|---|
| `github-creds` | Username with password | GitHub username | GitHub **Personal Access Token** | Checkout, Commit & Push |
| `nexus-creds` | Username with password | `jenkins-deployer` (or admin) | Nexus password | Nexus Upload |
| `sonar-token` | Secret text | n/a | SonarQube token from step 6.4 | Sonar server config |

**Creating the GitHub PAT:**

1. GitHub -> **Settings -> Developer settings -> Personal access tokens -> Tokens (classic)** (or fine-grained)
2. Scope: `repo` (classic) or *Contents: Read and write* (fine-grained)
3. Copy the token and paste it into the `github-creds` password field.

### 9.3 Configure the SonarQube Server in Jenkins

**Manage Jenkins -> System -> SonarQube servers**

1. Check **Environment variables -> Enable injection of SonarQube server configuration**
2. Click **Add SonarQube**
   - **Name:** `sonarqube` (must match `withSonarQubeEnv('sonarqube')`)
   - **Server URL:** `http://<SONARQUBE_IP>:9000`
   - **Server authentication token:** select `sonar-token`
3. Save.

---

## 10. Step 7: Create the Jenkins Pipeline Job

### Option A: Pipeline script from SCM (recommended)

1. **New Item** -> name: `flipkart-pipeline` -> **Pipeline** -> OK
2. **Pipeline** section:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/Rajesh33-11/flipkart.git`
   - Credentials: `github-creds`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
3. Save.

### Option B: Paste the script directly

Choose **Pipeline script**, paste the whole Jenkinsfile, and save.

> **First run note:** For a parameterized pipeline defined in the Jenkinsfile, the first run only registers the parameters. After it completes (or fails), the menu changes to **Build with Parameters**.

---

## 11. Pipeline Parameters

### 11.1 File Selection (select at least one)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `DEV_JSON` | Boolean | false | Update `node/dev.json` |
| `STAGE_JSON` | Boolean | false | Update `node/stage.json` |
| `UAT_JSON` | Boolean | false | Update `node/uat.json` |
| `PROD_JSON` | Boolean | false | Update `node/prod.json` |

### 11.2 Value Parameters (all are mandatory)

| Parameter | JSON Path | Example |
|---|---|---|
| `P_ENVIRONMENT` | `environment` | `dev` |
| `P_NODE_NAME` | `nodeName` | `worker-node-01` |
| `P_NODE_TYPE` | `nodeType` | `worker` |
| `P_REGION` | `region` | `ap-south-1` |
| `P_AZ` | `availabilityZone` | `ap-south-1a` |
| `P_INSTANCE_TYPE` | `instanceType` | `t3.medium` |
| `P_OS` | `os` | `ubuntu-22.04` |
| `P_K8S_ROLE` | `kubernetes.role` | `worker` |
| `P_K8S_VERSION` | `kubernetes.version` | `1.29` |
| `P_CPU` | `resources.cpu` | `2` (numbers only) |
| `P_MEMORY` | `resources.memory` | `4Gi` |
| `P_DISK` | `resources.disk` | `50Gi` |
| `P_LABEL_ENV` | `labels.environment` | `dev` |
| `P_LABEL_TEAM` | `labels.team` | `platform` |

---

## 12. Stage-by-Stage Explanation

### Helper: `abortBuild(msg)`
Sets the build result to `ABORTED` and throws an error, so validation problems are shown as *Aborted* (grey) rather than *Failed* (red).

### Pipeline-level blocks

| Block | Purpose |
|---|---|
| `agent any` | Runs on any available Jenkins node |
| `tools { jdk 'JDK17'; maven 'Maven3' }` | Puts Java 17 and Maven on the `PATH` |
| `parameters { ... }` | Defines the "Build with Parameters" form |
| `environment { REPO, BRANCH }` | Repo URL (without `https://`) and the branch to work on |

### Stage 1: Validate Inputs
- Collects the selected JSON files into `env.FILES` (space-separated).
- Aborts if **no file** is selected.
- Aborts if **any** value parameter is empty or equals `no-change`.
- Aborts if `P_CPU` is not numeric (`[0-9]+`).
- Sets the build display name and description (e.g. `#12 node/dev.json`).

### Stage 2: Checkout
- `cleanWs()` wipes the workspace so there are no stale files.
- `git(...)` clones the `main` branch using the `github-creds` credential.

### Stage 3: Update JSON
- Writes a `jq` filter (`filter.jq`) that reads each `P_*` variable from the environment (`$ENV`).
- For every selected file: checks that the file exists, applies the filter to a temp file, replaces the original, and prints the result.
- Empty values are ignored by the filter (`setstr` leaves the field untouched), though Stage 1 already guarantees all values are supplied.
- `filter.jq` is deleted at the end so it is never committed.

### Stage 4: Review Changes
- `git diff --quiet` returns non-zero if something changed -> `env.HAS_CHANGES = 'true'`.
- Prints `git diff --stat` and the full diff in the console log for audit.

### Stage 5: Commit & Push (only when `HAS_CHANGES == 'true'`)
- Uses `withCredentials` to expose `GIT_USER` and `GIT_TOKEN` (masked in logs).
- Configures the committer identity (`Jenkins`).
- `git add $FILES` stages only the selected JSON files.
- Commit message format: `[skip ci] Jenkins: updated <files> (build #<n>)`.
- `git pull --rebase` fetches remote updates first to avoid push rejection.
- `git push HEAD:main` pushes the commit.

> `[skip ci]` only stops the *next* build if your SCM trigger plugin honours it. Pipelines triggered by webhooks should have an explicit guard (see section 17).

### Stage 6: Maven Build
- Prints Java and Maven versions.
- Verifies `pom.xml` exists.
- Runs `mvn -B clean package` (compile, unit tests, create JAR/WAR in `target/`).

### Stage 7: SonarQube Analysis
- `withSonarQubeEnv('sonarqube')` injects `SONAR_HOST_URL` and `SONAR_AUTH_TOKEN`.
- Runs the Sonar Maven plugin with `sonar.projectKey=flipkart`.
- Results appear at `http://<SONARQUBE_IP>:9000/dashboard?id=flipkart`.

### Stage 8: Nexus Upload
- Reads `nexus-creds` and generates a temporary `settings.xml` with server id `nexus-releases`.
- Runs `mvn -B deploy -s settings.xml`, which uploads to the repository defined in `distributionManagement`.
- Deletes `settings.xml` after deployment.

### Stage 9: Check Artifact
- Lists the `target/` folder and finds the generated `.jar` / `.war`.

### Stage 10: Archive Artifact
- Stores `target/*.jar` and `target/*.war` in Jenkins with fingerprints for tracking. Fails if nothing is found (`allowEmptyArchive: false`).

### Post Actions

| Block | Triggered When | Action |
|---|---|---|
| `success` | All stages pass | Prints success summary |
| `failure` | Any stage fails | Prints a troubleshooting checklist |
| `aborted` | Validation abort / manual cancel | Prints an abort message |

---

## 13. Sample JSON File

Example `node/dev.json` structure expected by the `jq` filter:

```json
{
  "environment": "dev",
  "nodeName": "worker-node-01",
  "nodeType": "worker",
  "region": "ap-south-1",
  "availabilityZone": "ap-south-1a",
  "instanceType": "t3.medium",
  "os": "ubuntu-22.04",
  "kubernetes": {
    "role": "worker",
    "version": "1.29"
  },
  "resources": {
    "cpu": "2",
    "memory": "4Gi",
    "disk": "50Gi"
  },
  "labels": {
    "environment": "dev",
    "team": "platform"
  }
}
```

> **Note:** `jq`'s `setpath` writes all values as **strings**. `resources.cpu` will become `"2"` (string), not `2` (number). If your consumers expect a number, change the filter line to:
> `| (if p("P_CPU") != "" then setpath(["resources","cpu"]; (p("P_CPU") | tonumber)) else . end)`

---

## 14. How to Run the Pipeline

1. Open the job -> **Build with Parameters**
2. Tick one or more JSON files (e.g. `DEV_JSON`)
3. Fill in **all** `P_*` fields
4. Click **Build**
5. Open **Console Output** to follow the progress

**Example inputs:**

```
DEV_JSON        = true
P_ENVIRONMENT   = dev
P_NODE_NAME     = worker-node-01
P_NODE_TYPE     = worker
P_REGION        = ap-south-1
P_AZ            = ap-south-1a
P_INSTANCE_TYPE = t3.medium
P_OS            = ubuntu-22.04
P_K8S_ROLE      = worker
P_K8S_VERSION   = 1.29
P_CPU           = 2
P_MEMORY        = 4Gi
P_DISK          = 50Gi
P_LABEL_ENV     = dev
P_LABEL_TEAM    = platform
```

---

## 15. Verifying the Results

| Check | Where |
|---|---|
| JSON changes committed | GitHub -> repo -> `node/` -> latest commit by "Jenkins" |
| Build artifact | Jenkins -> Build page -> **Build Artifacts** |
| Code quality report | `http://<SONARQUBE_IP>:9000/dashboard?id=flipkart` |
| Deployed artifact | Nexus -> **Browse -> maven-releases1** -> `com/example/flipkart/<version>/` |

Manual verification from a terminal:

```bash
# Check Nexus
curl -u <user>:<password> \
  "http://<NEXUS_IP>:8081/service/rest/v1/components?repository=maven-releases1"

# Check Sonar
curl -u <sonar-token>: "http://<SONARQUBE_IP>:9000/api/system/status"
```

---

## 16. Troubleshooting

| Problem | Likely Cause | Solution |
|---|---|---|
| `ABORTED: Select at least one JSON file` | No checkbox ticked | Tick at least one `*_JSON` box |
| `ABORTED: Missing values` | Empty field or `no-change` | Fill every `P_*` field |
| `ABORTED: P_CPU must contain numbers only` | Value like `2 cores` | Use digits only, e.g. `2` |
| `jq: command not found` | jq not installed | `sudo apt install -y jq` on the agent |
| `File not found: node/xxx.json` | File absent in repo | Create/commit the file first |
| `Invalid tool 'JDK17'` / `Maven3` | Tool name mismatch | Check **Manage Jenkins -> Tools** names |
| `Authentication failed` on checkout/push | Wrong or expired PAT | Regenerate PAT; update `github-creds` |
| `remote: Permission denied` / `403` | PAT lacks `repo` scope or branch protection | Add scope; allow Jenkins to push to `main` |
| `non-fast-forward` rejected | Remote changed during build | Stage already does `pull --rebase`; re-run |
| `pom.xml not found` | Wrong repo/branch or path | Place `pom.xml` in repo root |
| `Unable to find SonarQube server 'sonarqube'` | Server name mismatch | Name it exactly `sonarqube` |
| `Not authorized` (Sonar) | Bad/expired token | Regenerate token; update `sonar-token` |
| Sonar container exits | `vm.max_map_count` too low | Apply sysctl settings in section 6.1 |
| Nexus `401 Unauthorized` | Wrong `nexus-creds` | Verify username/password |
| Nexus `400 ... redeploy` | Same version already exists | Bump version or enable *Allow redeploy* |
| Nexus `Return code is: 405` | Wrong URL in `distributionManagement` | Use `.../repository/maven-releases1/` |
| Deploy fails with 401 despite correct creds | `<id>` mismatch | Use `nexus-releases` in both pom.xml and settings |
| `No artifacts found to archive` | Build produced no jar/war | Check `<packaging>` and Maven output |
| Connection timeout to Sonar/Nexus | Firewall / Security Group | Open ports 9000 / 8081 from Jenkins |

---

## 17. Known Limitations and Recommended Improvements

These are observations from reviewing the current pipeline:

1. **Push happens before the build is validated.** If Maven, Sonar or Nexus fails, the JSON change is already on `main`. Consider building/testing first, or pushing to a feature branch and opening a pull request.
2. **No approval gate for PROD.** Add an `input` step when `PROD_JSON` is true:
   ```groovy
   stage('Approval') {
       when { expression { params.PROD_JSON } }
       steps { input message: 'Approve PROD change?', ok: 'Deploy' }
   }
   ```
3. **SonarQube Quality Gate is not enforced.** The stage passes even if the code fails the gate. Add:
   ```groovy
   stage('Quality Gate') {
       steps { timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true } }
   }
   ```
   (Requires a SonarQube webhook to `http://<JENKINS_IP>:8080/sonarqube-webhook/`.)
4. **Infinite build loop risk.** If a webhook/SCM trigger is added, the pipeline's own push could re-trigger it. Add a guard for commits starting with `[skip ci]` (e.g. the *SCM Skip* plugin).
5. **Hard-coded values.** The Nexus URL and repo name are echoed as static text in the Nexus stage. Move the IP, repo name and project key into `environment {}` or Jenkins global properties.
6. **`settings.xml` cleanup.** If `mvn deploy` fails, `rm -f settings.xml` is skipped and the file (containing the password) stays in the workspace. Use a trap:
   ```bash
   trap 'rm -f settings.xml' EXIT
   ```
   or use the Config File Provider plugin.
7. **`cpu` is stored as a string** in JSON (see section 13).
8. **Validation vs filter logic is inconsistent.** Validation forces *every* field to be provided, so the filter's "skip empty values" logic never triggers. If partial updates are desired, relax validation to require only some fields.
9. **No dependency caching.** `cleanWs()` removes everything each run; consider a shared `~/.m2` cache on the agent to speed up builds.
10. **Unit test reports are not published.** Add `junit 'target/surefire-reports/*.xml'` in a `post` block.
11. **HTTP (not HTTPS)** is used for Nexus and Sonar. Put both behind a reverse proxy with TLS for production.

---

## 18. Security Best Practices

- Never hard-code passwords/tokens in the Jenkinsfile. Always use Jenkins Credentials.
- Use a **fine-grained PAT** limited to this single repository.
- Use a dedicated Nexus user with deploy-only rights instead of `admin`.
- Rotate PATs, Sonar tokens and Nexus passwords regularly.
- Restrict Jenkins, Sonar and Nexus ports to trusted IP ranges.
- Enable **Matrix-based security** in Jenkins and limit who can run the PROD option.
- Enable branch protection on `main` and require pull requests for production config changes where possible.
- Keep Jenkins and all plugins up to date.

---

## Quick Setup Checklist

- [ ] Java 17, Git, jq installed
- [ ] Jenkins installed, plugins added
- [ ] JDK17 and Maven3 tools configured in Jenkins
- [ ] `github-creds`, `nexus-creds`, `sonar-token` credentials created
- [ ] SonarQube running, token generated, server `sonarqube` configured in Jenkins
- [ ] Nexus running, repository `maven-releases1` created
- [ ] `pom.xml` contains `distributionManagement` with id `nexus-releases`
- [ ] `node/*.json` files exist in the repo
- [ ] `Jenkinsfile` committed to the repo root
- [ ] Pipeline job created -> **Build with Parameters** -> success

---

**Maintainer:** Rajesh
**Repository:** https://github.com/Rajesh33-11/flipkart
