# Technology Refresher

A practical, rolling refresher for an experienced DevOps / infrastructure engineer. This is not a beginner certification course: each session reconnects concepts to real operational work, modern practices, troubleshooting, and technical interviews.

## Learning rhythm

- **Daily — 30 minutes:** one focused lesson, a small hands-on task, and a short self-check.
- **Progression:** lessons build on previous work. Record what you completed and what needs revisiting in `PROGRESS.md`.
- **Environment:** use disposable VMs, local containers, kind/minikube, AWS Free Tier where eligible, or isolated test directories. Avoid unexpected cloud charges; never experiment on production services or systems.

## Coverage and priorities

**AWS receives increased emphasis** and is a high-priority recurring subject alongside Terraform, Kubernetes, Docker and CI/CD. Cover AWS practically across IAM, EC2, VPC/subnets/route tables, security groups and NACLs, S3, RDS, Elastic Load Balancing, Auto Scaling, CloudWatch, CloudTrail, Systems Manager, ECR and EKS. Prefer local simulations or read-only console/CLI inspection when a lab could incur charges. Any resource-creation exercise must identify possible costs, use disposable resources, and include cleanup.

Core rotation: Jenkins and CI/CD; Java WAR builds with Maven/Ant; Python automation; Terraform; Docker; Kubernetes; Linux; Windows and PowerShell; networking; defensive security; MATLAB; Git; observability and troubleshooting.

**Paused for hands-on lessons:** Control-M and Axway SecureTransport, because neither product is installed in the current practice environment. Do not schedule further hands-on tutorials for these products unless the user later confirms an appropriate environment or explicitly requests a conceptual simulation.

## Lessons

- [Lesson 001 — Build a Java WAR with Maven](lessons/lesson-001-maven-war.md) — install/check the JDK and Maven, create a correctly structured project, build and inspect the WAR.
- [Lesson 002 — Jenkins Freestyle versus Declarative Pipeline](lessons/lesson-002-jenkins-freestyle-vs-pipeline.md) — map the Maven WAR build to both job models and place the Jenkinsfile correctly.
- [Lesson 003 — Diagnose a Linux systemd service safely](lessons/lesson-003-linux-systemd-diagnostics.md) — validate a unit offline and practise read-only service/journal diagnostics.
- [Lesson 004 — Build a resilient Python HTTP health checker](lessons/lesson-004-python-http-healthcheck.md) — practise bounded retries, timeouts, structured output and automation-friendly exit codes.
- [Lesson 005 — Terraform plan and state safety](lessons/lesson-005-terraform-plan-state.md) — use the built-in `terraform_data` resource to review a saved plan, apply it, inspect local state, recognise replacement and clean up without cloud credentials.
- [Lesson 006 — Docker image hardening](lessons/lesson-006-docker-image-hardening.md) — build a multi-stage Java image and apply safer runtime defaults.
- [Lesson 007 — Kubernetes readiness, rolling updates and rollback](lessons/lesson-007-kubernetes-readiness-rollout-rollback.md) — operate a disposable kind cluster and verify a controlled rollout and rollback.
- [Lesson 008 — Windows service and Event Log incident triage](lessons/lesson-008-windows-service-eventlog-triage.md) — inspect service state with CIM and query recent Service Control Manager events using PowerShell.
- [Lesson 009 — Control-M dependency-driven batch flow](lessons/lesson-009-controlm-flow-dependencies.md) — historical delivery; hands-on practice paused because Control-M is unavailable locally.
- [Lesson 010 — Axway SecureTransport transfer triage](lessons/lesson-010-securetransport-transfer-triage.md) — historical delivery; hands-on practice paused because SecureTransport is unavailable locally.
- [Lesson 011 — Layered TLS diagnostics with OpenSSL and curl](lessons/lesson-011-tls-diagnostics-openssl-curl.md) — separate DNS, TCP, SNI, certificate-chain/hostname verification and HTTP response.
- [Lesson 012 — MATLAB telemetry anomaly detection](lessons/lesson-012-matlab-telemetry-anomaly-detection.md) — validate synthetic telemetry, build a timetable and detect spikes with a causal moving-median baseline.
- [Lesson 013 — Apache Ant Java WAR packaging](lessons/lesson-013-ant-java-war-packaging.md) — compile Java 11 code, smoke-test, package a WAR, verify its layout and generate/check a SHA-256 digest; includes an expanded Maven-versus-Ant explanation.
- [Lesson 014 — AWS IAM policy evaluation and least-privilege S3 access](lessons/lesson-014-aws-iam-policy-evaluation.md) — draft a prefix-scoped read-only policy, validate JSON locally and simulate Allow/Deny outcomes without creating AWS resources.
- [Lesson 015 — AWS CloudTrail incident triage](lessons/lesson-015-aws-cloudtrail-incident-triage.md) — parse synthetic S3 data events, correlate assumed-role failures and distinguish Event history from data-event logging.

## Repository map

- [AWS Infrastructure Engineering — 8-part course (Lessons 016–023)](lessons/aws-infrastructure-course-8-part-roadmap.md) — account access through Terraform operations; Lessons 014–015 remain supplementary AWS groundwork.
- [Learning path](curriculum/learning-path.md) — recommended sequence and outcomes, with AWS prioritised.
- [Rotation](curriculum/rotation.md) — how subjects are revisited.
- [Daily lesson template](templates/daily-lesson.md) — consistent 30-minute structure.
- [Progress log](PROGRESS.md) — maintain your own completion notes.
- [Labs](labs/) — practical exercises and checkpoints.


## AWS lesson command parity

Every AWS lesson that includes command-line work must provide **both Windows PowerShell and Linux shell (Bash) equivalents**, either side by side or in clearly labelled subsections. Keep the operations and expected results equivalent. State prerequisites (AWS CLI, Python, jq, etc.), shell/version assumptions and platform-specific differences. If a task is console-only, say so rather than inventing a CLI equivalent. Reuse the same synthetic inputs/placeholders where practical. Keep cloud actions read-only or disposable, cost-aware and explicitly approved.

## Ground rules

1. Prefer official documentation and current versions; check compatibility before using commands.
2. Understand the *why*, not just the syntax. Explain trade-offs and failure modes.
3. Keep secrets out of Git. Use dummy credentials and synthetic data.
4. Security exercises are defensive and authorised only.
5. For deployment exercises, include validation, rollback, and environment-specific configuration.
6. For AWS, call out cost-bearing resources, least-privilege IAM, region assumptions and cleanup. Never create paid resources without explicit user approval.
7. Do not deliver hands-on Control-M or Axway SecureTransport lessons while the required products are unavailable.

The daily lesson can select the next logical topic; this repository is the durable syllabus and lab notebook.
