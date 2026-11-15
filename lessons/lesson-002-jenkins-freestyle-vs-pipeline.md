# Lesson 002 — Jenkins Freestyle versus Declarative Pipeline

**Domain:** Jenkins / CI/CD  
**Duration:** 30 minutes, excluding one-time setup  
**Level:** Experienced engineer refresher  
**Environment:** A disposable Jenkins LTS controller and a build agent; Windows and Linux agents are covered  
**Version assumptions:** A supported Jenkins LTS release with the Pipeline and Pipeline: Declarative plugins. The build agent has JDK 21+ and Maven 3.9.x on PATH.

> This lesson continues from Lesson 001. You need the `war-refresher` Maven project and a successful local `mvn -B -ntp clean verify` build. If you have not finished Lesson 001, first create and verify that project. Use a disposable Jenkins instance only; do not change a production or shared controller.

## Objective

Run the same Maven WAR build in two Jenkins job models:

1. A **Freestyle project**, configured through the Jenkins web interface.
2. A **Declarative Pipeline**, defined in a file named `Jenkinsfile` and stored with the project source.

Compare maintainability, reviewability and operational controls. This lesson builds and archives the WAR; it does not deploy to an application server.

## 0. Install a disposable Jenkins controller on PEACEKEEPER

For this lesson, use a **standalone Jenkins LTS WAR**, not an APT-installed system service or Docker container. It runs in your user session, stores all Jenkins data under one lab directory, listens only on your own computer, and can be stopped and removed without changing system services.

This lab uses PEACEKEEPER as both:
- the **controller** (the Jenkins web application); and
- the **built-in node** (the machine that executes this small test build).

That single-machine arrangement is acceptable for a disposable learning lab. In production, keep build executors off the controller and use separate, managed agents.

### 0.1 Check prerequisites

Open a PEACEKEEPER terminal (Bash):

```bash
java -version
javac -version
mvn -version
git --version
command -v curl
command -v sha256sum
```

Use Java 21, Maven 3.9.x and Git. If Maven or Git is missing, install only the missing package:

```bash
sudo apt update
sudo apt install -y maven git curl
```

Do not reinstall Java if your existing OpenJDK 21 installation is working. Re-run the version checks after installing anything.

### 0.2 Download and verify Jenkins LTS

Create a dedicated directory. The WAR file and Jenkins configuration will remain separate from your Maven project:

```bash
mkdir -p ~/labs/jenkins-lab/home
cd ~/labs/jenkins-lab
```

Download Jenkins **2.580.2 LTS**:

```bash
curl -fL https://get.jenkins.io/war-stable/2.580.2/jenkins.war -o ~/labs/jenkins-lab/jenkins.war
```

Verify the download against the SHA-256 published by Jenkins:

```bash
echo 'd98032d2401c540a35613a82ed358921fcf3166d5e6eaaaf60a7941845be3251  jenkins.war' | sha256sum -c -
```

Expected output:

```text
jenkins.war: OK
```

If verification fails, do not run the file. Delete it and download it again.

### 0.3 Start Jenkins locally

First check that port 8085 is free:

```bash
if ss -ltnH 'sport = :8085' | grep -q .; then
  echo "Port 8085 is already in use; stop and investigate before continuing."
else
  echo "Port 8085 is available."
fi
```

If it is available, start Jenkins with this command:

```bash
JENKINS_HOME="$HOME/labs/jenkins-lab/home" \
  java -jar "$HOME/labs/jenkins-lab/jenkins.war" \
  --httpPort=8085 \
  --httpListenAddress=127.0.0.1
```

Keep this terminal open while using Jenkins. The process runs in the foreground. The loopback address means the web interface is available only from PEACEKEEPER itself; it is not exposed to your LAN or the internet.

Wait for Jenkins to finish starting, then open Firefox on PEACEKEEPER and visit:

`http://127.0.0.1:8085`

### 0.4 Complete the first-run wizard

1. On the **Unlock Jenkins** page, open a second terminal and run:

   ```bash
   cat ~/labs/jenkins-lab/home/secrets/initialAdminPassword
   ```

2. Copy the displayed password into the browser.
3. Select **Install suggested plugins**. This should install the common Pipeline and Git functionality needed for the exercises.
4. Wait for plugin installation to finish.
5. Create a local Jenkins administrator account. Use a unique lab password; do not reuse a system or online-account password.
6. Select **Save and Finish**, then **Start using Jenkins**.
7. Confirm the Jenkins dashboard appears.

If a required plugin is missing later, install it only in this disposable instance through **Manage Jenkins → Plugins**.

### 0.5 Allow this lab's built-in node to run the build

Jenkins normally has a built-in node associated with the controller. For this single-machine lab, we will deliberately let that node run one build at a time:

1. Open **Manage Jenkins → Nodes**.
2. Select **Built-In Node**.
3. Select **Configure**.
4. Set **Number of executors** to `1`.
5. Save.

The Jenkins process was started from your PEACEKEEPER terminal as your normal user, so the built-in node can use that user's installed Java, Maven and Git. Verify those commands in Section 1.3. Do not run Jenkins with `sudo`.

**Important:** this controller/built-in-node arrangement is only for the local disposable lab. Do not copy this executor configuration to a production Jenkins controller.

### 0.6 Stop and remove the disposable instance

When finished, return to the terminal running Jenkins and press **Ctrl+C**. This stops the Jenkins process; it does not uninstall Java, Maven or Git.

To remove the Jenkins controller configuration, jobs, users and installed plugins after you no longer need them:

```bash
rm -rf ~/labs/jenkins-lab
```

That command permanently deletes only the dedicated lab directory. Do not run it if you want to keep your Jenkins jobs for the next lesson.

## 1. Prerequisites

### 1.1 Maven project from Lesson 001

You need these files from Lesson 001:

- `pom.xml`
- `src/main/java/com/example/HealthServlet.java`
- `src/main/webapp/index.html`

Example project folder:
- Windows: `C:\DevOps\war-refresher`
- Linux: `~/labs/war-refresher`

The directory containing `pom.xml` is the **project root**. You will create the Jenkinsfile in this exact directory, beside `pom.xml`.

### 1.2 Disposable Jenkins controller

For this lesson, use the local disposable controller you started in Section 0. Its suggested plugin installation should include the Pipeline functionality. Required plugins:

- **Pipeline**
- **Pipeline: Declarative**

Check **Manage Jenkins → Plugins → Installed plugins**. Many Jenkins LTS installations include these plugins through the suggested plugin set. Install a missing plugin only on your disposable lab controller, using Jenkins' plugin manager.

Do not install or modify plugins on a production controller for this exercise.

### 1.3 Jenkins build agent

The **agent** is the machine/process that checks out your project and runs Maven. In this PEACEKEEPER lab, the built-in node on the same machine as the controller is the agent. It needs:

- JDK 21 or newer (`java` and `javac`);
- Maven 3.9.x (`mvn`) on PATH;
- Git (`git`) if Jenkins will check out the source from a Git repository;
- network access to Maven repositories for first-time dependency downloads.

Verify the tools on the actual agent:

```text
java -version
javac -version
mvn -version
git --version
```

For this local lab, verify the commands in the same PEACEKEEPER environment from which you started Jenkins. In a distributed Jenkins setup, install and verify Java/Maven on each actual agent; installing them on the controller does not install them on an agent. Maven's version output shows which Java runtime Maven is using.

Windows agents run Windows batch commands using Jenkins' `bat` step. Linux/Unix agents run shell commands using `sh`. The Jenkinsfile below detects the agent OS.

### 1.4 Git source repository

For the Pipeline-as-Code option and a realistic SCM checkout, the project must be committed to a Git repository that the test Jenkins controller can access. The first Freestyle exercise deliberately uses a local workspace copy so you can run it before configuring a Git remote.

If you do not have a Git remote ready, you can still create and review the Jenkinsfile locally. You can paste its contents into a Pipeline job for a quick test, but that stores the script in Jenkins job configuration rather than loading the file from source control.

Do not put passwords, access tokens or private keys in the Jenkinsfile or repository. Use Jenkins Credentials if a later exercise needs authentication. No credentials are required here if the repository is public or already accessible.

## 2. Exactly where the Jenkinsfile goes

The **project root** is the top-level project directory containing `pom.xml`. Create the Jenkinsfile in that directory:

```text
C:\DevOps\war-refresher\             <- project root
├── pom.xml
├── Jenkinsfile                         <- CREATE THIS FILE HERE
└── src/
    └── main/
        ├── java/com/example/HealthServlet.java
        └── webapp/index.html
```

The filename is exactly `Jenkinsfile`, with no extension. It is not `Jenkinsfile.txt` and it does not belong inside `src`, `target` or `.m2`.

### 2.1 Navigate to the project root

**Windows PowerShell:**
```powershell
Set-Location C:\DevOps\war-refresher
Get-Location
Get-ChildItem
Test-Path .\pom.xml
```

**Linux:**
```bash
cd ~/labs/war-refresher
pwd
ls -la
test -f pom.xml && echo "pom.xml found"
```

You should see `pom.xml`. If not, stop and navigate to the directory where you created the Lesson 001 project.

### 2.2 Create the file

Use VS Code, Notepad++, Vim or another text editor.

**Windows PowerShell:**
```powershell
New-Item -ItemType File -Path .\Jenkinsfile
notepad .\Jenkinsfile
```

Paste the complete script from Section 5 into Notepad and save. If Windows asks about an extension, keep the filename exactly `Jenkinsfile`. Verify it:

```powershell
Get-ChildItem -Force Jenkinsfile*
```

It must show `Jenkinsfile`, not `Jenkinsfile.txt`.

**Linux:**
```bash
vim Jenkinsfile
```

Paste the script, save and quit with `:wq`. You may use another editor, but create the file in the current directory and name it exactly `Jenkinsfile`.

## 3. Compare the job models

| Freestyle project | Declarative Pipeline |
|---|---|
| Build steps and post-build actions configured in Jenkins UI | Workflow defined in a Pipeline script |
| Quick for simple jobs | Better for multi-stage workflows |
| UI changes can be harder to review as code | Jenkinsfile can be versioned and code-reviewed |
| Complex flows often rely on job chaining/plugins | Stages, conditions, parallel work and approvals are first-class Pipeline concepts |

Freestyle remains useful for straightforward or legacy jobs. Pipeline-as-Code is generally easier to review and maintain as delivery workflows grow.

## 4. Exercise A — configure a Freestyle job

Use the local Jenkins dashboard at `http://127.0.0.1:8085`. Because your Maven project is currently a local directory and may not yet have a Git remote, this first run uses a simple local-workspace copy. It lets you practise the Freestyle configuration without pretending that a Git checkout has happened.

1. Select **New Item**.
2. Enter `war-refresher-freestyle`.
3. Select **Freestyle project**, then **OK**.
4. Under **Source Code Management**, select **None** for this first local run.
5. Under **Build Steps**, select **Execute shell** and paste this complete script:

   ```bash
   set -eu
   PROJECT="$HOME/labs/war-refresher"

   test -f "$PROJECT/pom.xml"
   mkdir -p "$WORKSPACE"
   cp "$PROJECT/pom.xml" "$WORKSPACE/"
   rm -rf "$WORKSPACE/src"
   cp -a "$PROJECT/src" "$WORKSPACE/"

   cd "$WORKSPACE"
   java -version
   mvn -version
   mvn -B -ntp clean verify
   ```

   This copies only the Maven project definition and source tree into the Jenkins job's own workspace. Maven builds there, not in your original Lesson 001 directory.
6. Under **Post-build Actions**, add **Archive the artifacts** with pattern `target/*.war`.
7. Save, then select **Build Now**.
8. Open the build number and select **Console Output**. Confirm Maven reports `BUILD SUCCESS`.
9. Return to the build page and confirm that `war-refresher-1.0.0.war` is listed under **Build Artifacts**.

If your Lesson 001 project is in a different directory, change only the `PROJECT=` line to its actual path. If the build fails, check that the source directory contains `pom.xml` and `src/`, and that Maven is available to the user running Jenkins.

**Why Source Code Management is set to None here:** this is a first local exercise to learn the Freestyle job UI. It is not yet a source-controlled CI build. Once the project is pushed to a Git repository, you can change SCM to **Git**, enter that repository URL and use the normal checkout workflow. Never put a Git token or password in the repository URL.

Optional: if your project has automated tests that produce Surefire XML reports, configure the JUnit publisher with `**/target/surefire-reports/*.xml`. The sample project has no test classes, so do not expect reports unless you add tests.

Do not experiment on a production/shared controller.

## 5. Exercise B — create the Declarative Jenkinsfile

In the file `Jenkinsfile` at the project root (beside `pom.xml`), paste this complete script:

```groovy
pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    stages {
        stage('Check toolchain') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'java -version'
                        sh 'mvn -version'
                    } else {
                        bat 'java -version'
                        bat 'mvn -version'
                    }
                }
            }
        }

        stage('Build and verify WAR') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'mvn -B -ntp clean verify'
                    } else {
                        bat 'mvn -B -ntp clean verify'
                    }
                }
            }
        }

        stage('Archive WAR') {
            steps {
                archiveArtifacts(
                    artifacts: 'target/*.war',
                    fingerprint: true
                )
            }
        }
    }

    post {
        always {
            echo "Build finished with result: ${currentBuild.currentResult}"
        }
        failure {
            echo 'Build failed. Inspect the console log and Maven output.'
        }
    }
}
```

What the main sections do:

- `pipeline { }`: declares a Declarative Pipeline.
- `agent any`: lets Jenkins choose any eligible agent. In a larger estate, use a label for an agent with the correct JDK and Maven.
- `options`: prevents overlapping builds of this job and adds timestamps to logs.
- `stages`: groups work into named stages shown in Jenkins.
- `steps`: commands and actions within a stage.
- `isUnix()`: selects the right command step for the agent OS.
- `archiveArtifacts`: retains the WAR as a Jenkins build artifact.
- `fingerprint: true`: records artifact fingerprint data to help trace artifact use between Jenkins jobs.
- `post`: runs actions after the pipeline completes; here it reports the result and emits a failure message.

The `sh` and `bat` steps are not interchangeable: `sh` is for Unix-like agents; `bat` is for Windows agents.

## 6. Run and validate the Pipeline

There are two ways to give Jenkins the pipeline script.

### Option 1 — quick test by pasting the script

1. In Jenkins, select **New Item**.
2. Name it `war-refresher-pipeline-test`.
3. Select **Pipeline**, then **OK**.
4. Find the **Pipeline** section.
5. Set **Definition** to **Pipeline script**.
6. Paste the full Jenkinsfile content into the script box.
7. Save and select **Build Now**.

This is useful for a quick lab. The script is stored in Jenkins job configuration, not loaded from your Git repository.

### Option 2 — Pipeline as Code, loading Jenkinsfile from Git

1. Commit and push the Maven project, including `Jenkinsfile`, to a Git repository accessible to the test controller.
2. Create a new Jenkins item and select **Pipeline**.
3. In the **Pipeline** section, set **Definition** to **Pipeline script from SCM**.
4. Set **SCM** to **Git**.
5. Enter the test repository URL and select a Jenkins credential only if needed.
6. Set **Branch Specifier** to the branch containing your commit, e.g. `*/main`.
7. Leave **Script Path** as `Jenkinsfile` because the file is at the repository root.
8. Save and select **Build Now**.

If the file were in a subdirectory, Script Path would need that relative path. For this exercise, keep it at the repository root beside `pom.xml`.

### Expected result

The build should show three stages:

1. Check toolchain
2. Build and verify WAR
3. Archive WAR

The console output should include Maven's `BUILD SUCCESS`. The Jenkins build page should show an archived artifact similar to `war-refresher-1.0.0.war`.

Troubleshooting:
- **Toolchain stage fails:** check Java and Maven on the agent, not just the controller.
- **Checkout fails:** check repository URL, branch and Jenkins credential selection.
- **No artifacts match:** confirm Maven succeeded and the workspace contains `target/*.war`.
- **Wrong command step:** use `bat` on Windows and `sh` on Linux/Unix.

If Jenkins is unavailable, save the Jenkinsfile and review the configuration. Do not claim a runtime build succeeded unless you ran it.

## 7. Declarative versus Scripted Pipeline

Both are Jenkins Pipeline syntaxes, but they are different:

- **Declarative Pipeline** uses the structured `pipeline { agent; stages; steps; post }` form used in this lesson.
- **Scripted Pipeline** is Groovy-based and commonly uses `node { ... }` with Groovy control flow. It is more flexible but gives the author more responsibility.

Do not put a Declarative top-level `pipeline { }` block inside a Scripted `node { }` script. Choose the syntax intentionally.

## 8. Production-minded review

A Jenkinsfile in source control gives reviewers a versioned view of the build logic. Artifact archiving and fingerprinting improve traceability. A real release pipeline would also need deliberate environment promotion, approvals, credentials and rollback design; none of those actions happen in this lesson.

Build the WAR once, test it, retain it and promote that same artifact. Avoid rebuilding separately for each environment just to change environment-specific values.

## 9. Self-check

1. Where exactly is the Jenkinsfile in this exercise? Which file is beside it?
2. What is the difference between **Pipeline script** and **Pipeline script from SCM**?
3. What do `agent`, `stages`, `steps` and `post` do?
4. Why does the pipeline select between `sh` and `bat`?
5. What does artifact fingerprinting help you investigate?
6. When might a simple Freestyle job still be reasonable?

## Completion checkpoint

Report: Freestyle configuration mapped (yes/no); Jenkinsfile created in project root (yes/no); disposable Jenkins controller/agent available (yes/no); Pipeline executed (yes/no/not available); build result and WAR archived (if executed); one meaningful difference between the two job models; confidence from 1–5.

Do not mark the lesson complete until you have attempted the exercise. If Jenkins is unavailable, record that fact rather than claiming it ran.

## Official documentation

- [Jenkins — WAR file installation (run, unlock and configure)](https://www.jenkins.io/doc/book/installing/war-file/)
- [Jenkins — Download and deploy (LTS releases and checksums)](https://www.jenkins.io/download/)
- [Jenkins — Java support policy](https://www.jenkins.io/doc/book/platform-information/support-policy-java/)
- [Jenkins — Installing Jenkins](https://www.jenkins.io/doc/book/installing/)
- [Jenkins — Managing Plugins](https://www.jenkins.io/doc/book/managing/plugins/)
- [Jenkins — Pipeline](https://www.jenkins.io/doc/book/pipeline/)
- [Jenkins — Using a Jenkinsfile](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/)
- [Jenkins — Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
