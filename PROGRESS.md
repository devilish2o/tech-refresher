# Progress log

Update this file after each session. Keep entries brief and record concrete evidence, not just a topic name.

## Current status

- Start date: 2026-11-12
- Daily target: 30 minutes
- Lesson 001: completed (user-reported).
- Lesson 002: completed (user-reported, 2026-11-15).
- Lesson 003: completed (user-reported, 2026-11-15).
- Lesson 004: completed (user-reported).
- Lesson 005: completed (user-reported, 2026-11-18).
- Lesson 006: completed (user-reported, 2026-11-18).
- Lesson 007: completed (user-reported, 2026-11-24).
- Lesson 008: completed (user-reported, 2026-11-20).
- Lesson 009: not completed; Control-M is not installed. Do not schedule another hands-on Control-M tutorial unless the user confirms an environment or explicitly requests a conceptual simulation.
- Lesson 010: not completed; Axway SecureTransport is not installed. Do not schedule another hands-on SecureTransport tutorial unless the user confirms an environment or explicitly requests a conceptual simulation.
- Lesson 011: completed (user-reported, 2026-11-24).
- Lesson 012: completed (user-reported, 2026-11-24).
- Lesson 013: delivered (2026-11-24); completion checkpoint pending.

## Completed sessions

| Date | Topic | Practical result | Confidence (1–5) | Next step |
|---|---|---|---:|---|
| 2026-11-14 | Lesson 001 — Maven WAR packaging | User reported lesson completed | — | Continue curriculum |
| 2026-11-15 | Lesson 002 — Jenkins Freestyle versus Declarative Pipeline | User reported lesson completed | — | Continue curriculum |
| 2026-11-15 | Lesson 003 — Linux systemd service diagnostics | User reported lesson completed | — | Continue curriculum |
| 2026-11-16 | Lesson 004 — Python HTTP health checker | User reported lesson completed | — | Continue curriculum |
| 2026-11-18 | Lesson 005 — Terraform saved plans and local state safety | User reported lesson completed | — | Continue curriculum |
| 2026-11-18 | Lesson 006 — Docker multi-stage Java image and runtime hardening | User confirmed restricted container run and Maven-absence check; lesson completed | — | Continue curriculum |
| 2026-11-24 | Lesson 007 — Kubernetes readiness probes, rolling updates and rollback | User reported lesson completed | — | Continue curriculum |
| 2026-11-20 | Lesson 008 — Windows service and Event Log incident triage | User reported lesson completed; unrelated SCM events were identified and ruled out for the DNS Client service | — | Continue curriculum |
| 2026-11-24 | Lesson 011 — Networking/TLS diagnostics | User reported lesson completed | — | Continue curriculum |
| 2026-11-24 | Lesson 012 — MATLAB telemetry anomaly detection | User reported lesson completed | — | Continue curriculum |

## Unavailable software — do not schedule hands-on lessons

| Lesson | Product | Status | Future handling |
|---:|---|---|---|
| 009 | Control-M | Not completed; product not installed | Pause hands-on tutorials. Resume only if user confirms access or requests a conceptual simulation. |
| 010 | Axway SecureTransport | Not completed; product not installed | Pause hands-on tutorials. Resume only if user confirms access or requests a conceptual simulation. |

## Delivered lessons

| Lesson | Domain / topic | Status |
|---:|---|---|
| 001 | Maven WAR packaging | Completed (user-reported) |
| 002 | Jenkins Freestyle versus Declarative Pipeline | Completed (user-reported, 2026-11-15) |
| 003 | Linux systemd service diagnostics | Completed (user-reported, 2026-11-15) |
| 004 | Python HTTP health checker | Completed (user-reported) |
| 005 | Terraform saved plans and local state safety | Completed (user-reported, 2026-11-18) |
| 006 | Docker multi-stage Java image and runtime hardening | Completed (user-reported, 2026-11-18) |
| 007 | Kubernetes readiness probes, rolling updates and rollback | Completed (user-reported, 2026-11-24) |
| 008 | Windows PowerShell service and Event Log incident triage | Completed (user-reported, 2026-11-20) |
| 009 | Control-M dependency-driven batch flow | Not completed — product unavailable; hands-on tutorials paused |
| 010 | Axway SecureTransport transfer triage | Not completed — product unavailable; hands-on tutorials paused |
| 011 | Networking/TLS: DNS, TCP, SNI, certificate-chain and hostname verification with OpenSSL/curl | Completed (user-reported, 2026-11-24) |
| 012 | MATLAB: CSV telemetry validation, timetable construction and causal moving-median anomaly detection | Completed (user-reported, 2026-11-24) |
| 013 | Apache Ant: Java 11 compilation, smoke test, WAR packaging, archive-layout verification and SHA-256 integrity check | Delivered (2026-11-24); completion pending |

## Topics to revisit

- [ ] AWS architecture and operations: IAM, EC2, VPC, S3, RDS, ELB/ALB, Auto Scaling, CloudWatch, CloudTrail, Systems Manager, ECR and EKS
- [ ] Jenkins pipelines and shared libraries
- [ ] Java WAR build and environment promotion
- [ ] Python automation and API resilience
- [ ] Terraform state, modules, drift and AWS provider workflows
- [ ] Docker image and runtime security
- [ ] Kubernetes operations and troubleshooting
- [ ] Linux and Windows diagnostics
- [ ] Networking, TLS and defensive security
- [ ] MATLAB refresh
- [ ] Git, observability and incident response

## Scheduling guidance

- Give AWS increased weight across the rotation, not just as a one-off topic. Pair AWS labs with Terraform, IAM/security, networking, observability and CI/CD where useful.
- Prefer free/local simulations or read-only inspection. Identify cost-bearing AWS resources and cleanup steps; do not create paid resources without explicit user approval.
- Do not schedule new hands-on Control-M or Axway SecureTransport tutorials while the products are unavailable. A conceptual/simulated lesson requires the user's explicit request.
- Every daily lesson remains independently achievable in 30 minutes. Longer labs are optional and must not be created automatically from a daily lesson.
