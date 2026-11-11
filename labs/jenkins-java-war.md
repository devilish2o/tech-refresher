# Lab: Jenkins pipeline for a Java WAR

## Goal

Build, test, archive and promote one immutable WAR artifact through isolated environments. Compare a Freestyle job with a Jenkinsfile pipeline.

## Suggested setup

A disposable Jenkins instance and a small sample Java web application. Use Maven or Ant according to the project. Do not connect this lab to production credentials, deployment targets or repositories.

## Stages

1. **Checkout:** pin the branch or tag; record the commit SHA.
2. **Build:** run the Maven/Ant build reproducibly.
3. **Test:** execute unit tests and publish test results.
4. **Package:** create a versioned WAR and calculate its checksum.
5. **Archive:** retain the exact artifact; do not rebuild separately for each environment.
6. **Deploy to dev:** inject environment-specific configuration at deploy time.
7. **Promote:** require an explicit approval before test / production-like deployment.
8. **Verify:** health check, smoke test and deployment record.
9. **Rollback:** redeploy the previous known-good artifact.

## Exercises

- Create the equivalent Freestyle job and declarative Jenkinsfile; compare maintainability and reviewability.
- Add build parameters and validate them rather than interpolating arbitrary shell input.
- Demonstrate that the same artifact checksum is used in each environment.
- Simulate a failed health check and implement a rollback path.
- Identify where credentials belong and ensure they never appear in console logs.

## Self-check

Can you explain agent isolation, workspace cleanup, artifact retention, credential scope, approvals, immutable artifacts and rollback? What happens if a deployment succeeds but the health check fails?

## Documentation

- [Jenkins Pipeline](https://www.jenkins.io/doc/book/pipeline/)
- [Jenkinsfile](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/)
- [Maven Getting Started](https://maven.apache.org/guides/getting-started/)
- [Apache Ant Manual](https://ant.apache.org/manual/)
