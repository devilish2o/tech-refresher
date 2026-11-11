# Learning path

The sequence is modular rather than tied to a fixed number of weeks. The daily refresher should select one 30-minute unit at a time and adapt the next unit to the previous result.

## Track 1 — Delivery engineering

1. Git workflows, branching, tags, release hygiene.
2. Jenkins architecture, agents, credentials, plugins, build isolation.
3. Freestyle jobs versus Jenkinsfile pipelines; declarative pipeline structure.
4. Java application build: Maven or Ant, tests, WAR packaging, artifact versioning.
5. Deploy the same artifact through dev / test / production-like environments.
6. Configuration externalisation, approvals, health checks, rollback, audit trail.

**Outcome:** design and explain a reliable build-and-release pipeline, including operational controls.

## Track 2 — Automation and infrastructure as code

1. Python for reliable operational automation: arguments, logging, exceptions, config.
2. APIs, HTTP, JSON, pagination, retries, timeouts, idempotency.
3. Terraform providers, resources, variables, outputs, state and plans.
4. Modules, remote state, locking, drift, import and safe change review.
5. PowerShell for Windows administration and repeatable automation.

**Outcome:** create maintainable automation with predictable failure handling and safe execution.

## Track 3 — Containers and orchestration

1. Docker images, layers, build context, multi-stage builds.
2. Container networking, volumes, health checks, resource limits and security.
3. Kubernetes objects: Pods, Deployments, Services, ConfigMaps and Secrets.
4. Probes, requests/limits, rollouts, rollback, namespaces and RBAC.
5. Debugging scheduling, DNS, networking, storage and crash loops.

**Outcome:** package, deploy and troubleshoot a small service locally without risking live workloads.

## Track 4 — Enterprise scheduling and managed file transfer

1. Control-M: jobs, folders, calendars, dependencies, conditions and variables.
2. Reruns, cyclic jobs, SLA monitoring, alerts, recovery and operational handover.
3. Axway SecureTransport concepts: partners, users, protocols, endpoints, keys and certificates.
4. MFT workflow design: landing zones, routing, naming, encryption, acknowledgements.
5. Failure scenarios: duplicate delivery, partial transfer, retries, expiry, audit and reconciliation.

**Outcome:** reason about batch and file-transfer workflows as reliable, observable business processes. Use vendor documentation matching the installed product version.

## Track 5 — Systems, networks and security

1. Linux processes, systemd, filesystems, permissions, logs and performance.
2. Windows services, Event Viewer, PowerShell, scheduled tasks and networking.
3. TCP/IP, DNS, routing, NAT, TLS, proxies, firewalls and packet-level diagnosis.
4. Identity, least privilege, secrets, certificates, patching and vulnerability response.
5. Observability: logs, metrics, traces, alert quality and incident analysis.

**Outcome:** troubleshoot across the application-to-network-to-operating-system boundary.

## Track 6 — MATLAB and analytical computing

Refresh MATLAB syntax, arrays, tables, plotting, scripts/functions, data import and numerical workflows. Use small synthetic datasets and focus on reading, maintaining and validating existing code.

## Session completion

A topic is complete only when you can:
- explain the design and its trade-offs without relying on notes;
- perform a small practical task;
- diagnose at least one intentional failure;
- state how you would make the change safe in a regulated environment.
