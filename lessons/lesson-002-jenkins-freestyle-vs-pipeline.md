# Lesson 002 — Jenkins Freestyle versus Declarative Pipeline

**Domain:** Jenkins / CI/CD  
**Duration:** 30 minutes, excluding one-time setup  
**Level:** Experienced engineer refresher  
**Environment:** A disposable Jenkins LTS controller and a build agent; Windows and Linux agents are covered  
**Version assumptions:** A supported Jenkins LTS release with the Pipeline and Pipeline: Declarative plugins. The build agent has JDK 17+ and Maven 3.9.x on PATH.

> This lesson continues from Lesson 001. You need the `war-refresher` Maven project and a successful local `mvn -B -ntp clean verify` build. If you have not finished Lesson 001, first create and verify that project. Use a disposable Jenkins instance only; do not change a production or shared controller.

## Objective

Run the same Maven WAR build in two Jenkins job models:

1. A **Freestyle project**, configured through the Jenkins web interface.
2. A **Declarative Pipeline**, defined in a file named `Jenkinsfile` and stored with the project source.

Compare maintainability, reviewability and operational controls. This lesson builds and archives the WAR; it does not deploy to an application server.

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

You need access to a non-production Jenkins LTS controller. Required plugins:

- **Pipeline**
- **Pipeline: Declarative**

Check **Manage Jenkins → Plugins → Installed plugins**. Many Jenkins LTS installations include these plugins through the suggested plugin set. Install a missing plugin only on your disposable lab controller, using Jenkins' plugin manager.

Do not install or modify plugins on a production controller for this exercise.

### 1.3 Jenkins build agent

The controller schedules work; the **agent** is the machine/process that checks out your project and runs Maven. The agent selected for this job needs:

- JDK 17 or newer (`java` and `javac`);
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

If the controller and agent are separate machines, installing Java/Maven on the controller does not install them on the agent. Verify the agent itself. Maven's version output shows which Java runtime Maven is using.

Windows agents run Windows batch commands using Jenkins' `bat` step. Linux/Unix agents run shell commands using `sh`. The Jenkinsfile below detects the agent OS.

### 1.4 Git source repository

For the SCM-based Pipeline option and a realistic Freestyle checkout, the project must be committed to a Git repository that the test Jenkins controller can access.

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

On your disposable Jenkins controller:

1. From the dashboard, select **New Item**.
2. Name the job `war-refresher-freestyle`.
3. Select **Freestyle project**, then **OK**.
4. Under **Source Code Management**, select **Git** and enter the URL of your test repository containing the Maven project. If it is private, select a Jenkins credential; never paste a token into the URL.
5. Under **Build Steps**, choose the command type matching the agent:
   - Linux/Unix: **Execute shell** with `mvn -B -ntp clean verify`.
   - Windows: **Execute Windows batch command** with `mvn -B -ntp clean verify`.
6. Under **Post-build Actions**, add **Archive the artifacts** with pattern `target/*.war`.
7. Save and select **Build Now**.

Optional: if your project has automated tests that produce Surefire XML reports, configure the JUnit publisher with `**/target/surefire-reports/*.xml`. The sample project has no test classes, so do not expect reports unless you add tests.

If Jenkins is unavailable, write down this configuration mapping. Do not experiment on a production/shared controller.

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

- [Jenkins — Installing Jenkins](https://www.jenkins.io/doc/book/installing/)
- [Jenkins — Managing Plugins](https://www.jenkins.io/doc/book/managing/plugins/)
- [Jenkins — Pipeline](https://www.jenkins.io/doc/book/pipeline/)
- [Jenkins — Using a Jenkinsfile](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/)
- [Jenkins — Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
