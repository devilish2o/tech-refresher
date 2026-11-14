# Technology Refresher

A practical, rolling refresher for an experienced DevOps / infrastructure engineer. This is not a beginner certification course: each session reconnects concepts to real operational work, modern practices, troubleshooting, and technical interviews.

## Learning rhythm

- **Daily — 30 minutes:** one focused lesson, a small hands-on task, and a short self-check.
- **Tuesday / Thursday — 18:00 deep dive:** longer labs when a topic benefits from building something end-to-end.
- **Progression:** lessons build on previous work. Record what you completed and what needs revisiting in `PROGRESS.md`.
- **Environment:** use disposable VMs, local containers, kind/minikube, or isolated test directories. Do not experiment on production services or systems.

## Coverage

Jenkins and CI/CD; Java WAR builds with Maven/Ant; Python automation; Terraform; Docker; Kubernetes; Control-M; Axway SecureTransport / managed file transfer; Linux; Windows and PowerShell; networking; security; MATLAB; Git; observability and troubleshooting.

## Lessons

- [Lesson 001 — Build a Java WAR with Maven](lessons/lesson-001-maven-war.md) — install/check the JDK and Maven, create a correctly structured project, build and inspect the WAR.
- [Lesson 002 — Jenkins Freestyle versus Declarative Pipeline](lessons/lesson-002-jenkins-freestyle-vs-pipeline.md) — map the Maven WAR build to both job models and place the Jenkinsfile correctly.
- [Lesson 003 — Diagnose a Linux systemd service safely](lessons/lesson-003-linux-systemd-diagnostics.md) — validate a unit offline and practise read-only service/journal diagnostics.

## Repository map

- [Learning path](curriculum/learning-path.md) — recommended sequence and outcomes.
- [Rotation](curriculum/rotation.md) — how subjects are revisited.
- [Daily lesson template](templates/daily-lesson.md) — consistent 30-minute structure.
- [Progress log](PROGRESS.md) — maintain your own completion notes.
- [Labs](labs/) — practical exercises and checkpoints.

## Ground rules

1. Prefer official documentation and current versions; check compatibility before using commands.
2. Understand the *why*, not just the syntax. Explain trade-offs and failure modes.
3. Keep secrets out of Git. Use dummy credentials and synthetic data.
4. Security exercises are defensive and authorised only.
5. For deployment exercises, include validation, rollback, and environment-specific configuration.

The daily lesson can select the next logical topic; this repository is the durable syllabus and lab notebook.
