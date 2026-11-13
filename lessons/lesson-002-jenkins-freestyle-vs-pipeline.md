# Lesson 002 — Jenkins Freestyle versus Declarative Pipeline

**Domain:** Jenkins / CI/CD  
**Duration:** 30 minutes  
**Assumptions:** Jenkins LTS with the Pipeline and Declarative Pipeline plugins. The build agent has a compatible JDK and Maven available. Use a disposable Jenkins instance only.

## Objective

Take the Maven WAR build from Lesson 001 and express the same CI task in two Jenkins job models:

1. A Freestyle project configured through the Jenkins UI.
2. A Declarative Pipeline, preferably stored as a `Jenkinsfile` in source control.

Compare maintainability, reviewability, repeatability and operational controls. This lesson does not deploy to an application server.

## 30-minute plan

- **0–5 min — Compare the models.** Identify where each job's build logic and configuration live.
- **5–12 min — Design the Freestyle equivalent.** Map checkout, Maven build, test reports and artifact archiving to UI configuration.
- **12–22 min — Write the Jenkinsfile.** Create the Declarative Pipeline below and adapt the shell step for the agent OS.
- **22–27 min — Validate.** Run on a disposable Jenkins instance if available; otherwise perform a structured syntax and configuration review.
- **27–30 min — Review trade-offs.** Decide which model you would standardise for a multi-environment delivery process and why.

## 1. Freestyle job blueprint

Create a disposable job named `war-refresher-freestyle`.

| Jenkins configuration area | Setting |
|---|---|
| Source Code Management | Git; point to a test repository containing the Maven WAR project |
| Build step | Execute shell: `mvn -B -ntp clean verify` |
| Windows agent alternative | Execute Windows batch command: `mvn -B -ntp clean verify` |
| Post-build action | Archive artifacts: `target/*.war` |
| Optional test reporting | Publish JUnit results: `**/target/surefire-reports/*.xml` if the project has tests producing Surefire XML reports |

Do not configure deployment credentials or production targets. If you do not have a disposable Jenkins instance, write down this configuration mapping rather than changing an existing controller.

## 2. Declarative Pipeline

Create a file named `Jenkinsfile` at the root of the Maven project:

```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Build and test') {
            steps {
                // Linux / Unix agent
                sh 'mvn -B -ntp clean verify'
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
            echo 'Inspect the console log and test/build reports.'
        }
    }
}
```

For a Windows agent, replace the `sh` step with:

```groovy
bat 'mvn -B -ntp clean verify'
```

The `sh` and `bat` steps are agent-OS-specific; do not assume a Windows agent can run `sh` or a Linux agent can run `bat`.

### Validate it

On a disposable Jenkins instance:

1. Create a **Pipeline** job.
2. Either paste the script into the Pipeline definition for a quick test, or use **Pipeline script from SCM** to load the committed `Jenkinsfile`.
3. Run the job and inspect the stage view, console output and archived WAR.
4. If using SCM, make a harmless change to the Jenkinsfile and confirm the change is visible in version control and code review.

Jenkins also exposes a Declarative Pipeline linter in the controller's Pipeline development tools. Prefer that controller's linter because available steps and plugin versions are installation-specific.

If no disposable Jenkins is available, save the Jenkinsfile and complete the mapping/review exercise. Do not install or modify Jenkins on a live host just for this lesson.

## 3. What to distinguish

| Freestyle | Declarative Pipeline |
|---|---|
| Build steps and post-build actions are configured in the UI | Workflow is expressed in a Jenkinsfile |
| Simple sequential jobs are quick to configure | Stages and workflow logic are explicit and reviewable |
| Configuration changes can be harder to review as code | Jenkinsfile changes can be versioned and code-reviewed |
| Complex orchestration often requires chaining jobs or plugins | Pipeline has first-class stages, conditions, parallelism and input/approval steps |

Declarative Pipeline is not the same as Scripted Pipeline:

- **Declarative:** uses the constrained `pipeline { agent; stages; steps; post }` structure.
- **Scripted:** uses Groovy-based control flow, commonly inside `node { ... }`; it is more flexible but places more responsibility on the author.

Both are Jenkins Pipeline syntaxes. This lesson uses Declarative because it is the clearer default for a conventional build/test workflow. Do not mix Declarative-only blocks such as the top-level `pipeline { }` into a Scripted `node { }` script.

## Expected outcome

- A written Freestyle configuration map.
- A syntactically coherent Declarative `Jenkinsfile`.
- A successful build and archived WAR if run on a disposable Jenkins instance with the required tools and project available.
- No deployment or changes to production systems.

## Self-check

1. Where is the Freestyle build configuration stored and reviewed compared with a Jenkinsfile?
2. What is the role of `agent`, `stages`, `steps` and `post` in Declarative Pipeline?
3. Why is `sh` not interchangeable with `bat`?
4. What does artifact fingerprinting help you trace?
5. Which model would you choose for a multi-branch application with approvals and why?

## Production / interview relevance

For regulated delivery, a version-controlled Jenkinsfile provides a reviewable record of how a build is performed. A repeatable pipeline also makes it easier to enforce consistent tests, artifact retention and approval gates. A Freestyle job remains useful for straightforward legacy automation, but complex release orchestration is usually easier to reason about as Pipeline-as-Code.

## Completion checkpoint

Report:

- Freestyle configuration mapped: yes/no.
- Jenkinsfile created: yes/no.
- Pipeline executed on a disposable Jenkins: yes/no/not available.
- Build result and archived WAR: if executed.
- One meaningful difference you identified.
- Confidence: 1–5.

Do not mark this lesson complete until the exercise has actually been attempted.
