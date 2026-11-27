# Progress log

Update this file after each session. Keep entries brief and record concrete evidence, not just a topic name.

## Current status

- Start date: 2026-11-12
- Daily target: 30 minutes
- Lessons 001–008: completed (user-reported; see table below).
- Lesson 009: not completed; Control-M is not installed. Hands-on lessons paused.
- Lesson 010: not completed; Axway SecureTransport is not installed. Hands-on lessons paused.
- Lessons 011–012: completed (user-reported, 2026-11-24).
- Lesson 013: delivered (2026-11-24); completion checkpoint pending.
- Lesson 014: delivered (2026-11-25); completion checkpoint pending. AWS supplementary lesson: IAM policy evaluation; supports AWS course Lesson 017.
- Lesson 015: delivered (2026-11-26); completion checkpoint pending. AWS supplementary lesson: CloudTrail triage; supports AWS course Lesson 022.
- AWS foundations course: accepted by user; Lessons 016–023 planned, not yet delivered. User already has an AWS account. Start at Lesson 016 (account access, console, CLI and cost guardrails); do not ask them to create a new account.

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

## Delivered and planned lessons

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
| 011 | Networking/TLS diagnostics | Completed (user-reported, 2026-11-24) |
| 012 | MATLAB telemetry anomaly detection | Completed (user-reported, 2026-11-24) |
| 013 | Apache Ant Java WAR packaging | Delivered (2026-11-24); completion pending |
| 014 | AWS IAM policy evaluation and least-privilege S3 access | Delivered (2026-11-25); completion pending; supplementary to AWS course Lesson 017 |
| 015 | AWS CloudTrail S3 data-event triage and AccessDenied diagnosis | Delivered (2026-11-26); completion pending; supplementary to AWS course Lesson 022 |
| 016 | AWS account access, console, CLI and cost guardrails | Planned — next AWS course lesson |
| 017 | AWS IAM administration and permissions | Planned — builds on Lesson 014 |
| 018 | AWS VPC networking from first principles | Planned |
| 019 | AWS EC2 Linux compute and lifecycle | Planned |
| 020 | AWS storage: S3, EBS and EFS | Planned |
| 021 | AWS application delivery: Route 53, load balancing and scaling | Planned |
| 022 | AWS monitoring, auditing and incident troubleshooting | Planned — builds on Lesson 015 |
| 023 | AWS Terraform and end-to-end operations | Planned |

## Topics to revisit

- [ ] AWS foundations course Lessons 016–023
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

- Continue the accepted AWS foundations sequence at Lesson 016. Do not restart at account registration: the user already has an AWS account.
- Give AWS increased weight across the rotation, pairing lessons with Terraform, IAM/security, networking, observability and CI/CD where useful.
- Every AWS command-based lesson must include aligned PowerShell and Linux Bash commands, prerequisites, expected output and platform-specific differences.
- Prefer free/local simulations or read-only inspection. Identify cost-bearing AWS resources and cleanup steps. Do not create paid resources without explicit user approval.
- Do not schedule new hands-on Control-M or Axway SecureTransport tutorials while the products are unavailable. A conceptual/simulated lesson requires the user's explicit request.
- Every daily lesson remains independently achievable in 30 minutes. Longer labs are optional and must not be created automatically from a daily lesson.
