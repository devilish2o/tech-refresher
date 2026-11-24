# Learning path

The sequence is modular rather than tied to a fixed number of weeks. The daily refresher should select one 30-minute unit at a time and adapt the next unit to the previous result.

AWS receives increased emphasis throughout the learning path. Integrate AWS with Terraform, networking, security, containers, CI/CD and observability rather than treating it as a standalone certification track.

## Track 1 — AWS infrastructure and operations (high priority)

1. AWS account structure, regions, Availability Zones and shared responsibility.
2. IAM users, roles, policies, trust relationships, STS and least privilege.
3. VPC design: CIDR, public/private subnets, route tables, internet/NAT gateways, security groups and NACLs.
4. EC2 operations: instance lifecycle, AMIs, EBS, metadata, Systems Manager and safe access.
5. S3 security, encryption, versioning, lifecycle and access controls.
6. RDS fundamentals, backups, Multi-AZ and connectivity.
7. Elastic Load Balancing, target groups, health checks and Auto Scaling.
8. CloudWatch metrics, logs, alarms and dashboards; CloudTrail audit investigation.
9. ECR/ECS/EKS and container delivery.
10. AWS cost visibility, tagging, budgets and cleanup.

**Outcome:** design, secure, troubleshoot and explain common AWS infrastructure and operational workflows. Prefer free/local simulations or read-only exercises where practical. Identify cost-bearing resources before any creation and obtain explicit approval before creating resources that may incur charges.

## Track 2 — Delivery engineering

1. Git workflows, branching, tags, release hygiene.
2. Jenkins architecture, agents, credentials, plugins, build isolation.
3. Freestyle jobs versus Jenkinsfile pipelines; declarative pipeline structure.
4. Java application build: Maven or Ant, tests, WAR packaging, artifact versioning.
5. Deploy the same artifact through dev / test / production-like environments.
6. Configuration externalisation, approvals, health checks, rollback, audit trail.
7. AWS CI/CD integration: IAM roles, artifact storage, ECR and deployment orchestration.

**Outcome:** design and explain a reliable build-and-release pipeline, including operational controls.

## Track 3 — Automation and infrastructure as code

1. Python for reliable operational automation: arguments, logging, exceptions, config.
2. APIs, HTTP, JSON, pagination, retries, timeouts, idempotency.
3. Terraform providers, resources, variables, outputs, state and plans.
4. AWS provider workflows: VPC, IAM, EC2, S3 and safe plan review.
5. Modules, remote state, locking, drift, import and safe change review.
6. PowerShell for Windows administration and repeatable automation.

**Outcome:** create maintainable automation with predictable failure handling and safe execution.

## Track 4 — Containers and orchestration

1. Docker images, layers, build context, multi-stage builds.
2. Container networking, volumes, health checks, resource limits and security.
3. Kubernetes objects: Pods, Deployments, Services, ConfigMaps and Secrets.
4. Probes, requests/limits, rollouts, rollback, namespaces and RBAC.
5. EKS architecture, IAM integration, networking, node groups and workload identity.
6. Debugging scheduling, DNS, networking, storage and crash loops.

**Outcome:** package, deploy and troubleshoot a small service locally and understand its AWS deployment counterpart.

## Track 5 — Systems, networks and security

1. Linux processes, systemd, filesystems, permissions, logs and performance.
2. Windows services, Event Viewer, PowerShell, scheduled tasks and networking.
3. TCP/IP, DNS, routing, NAT, TLS, proxies, firewalls and packet-level diagnosis.
4. AWS network boundaries, VPC endpoints, security groups, NACLs and flow logs.
5. Identity, least privilege, secrets, certificates, patching and vulnerability response.
6. Observability: logs, metrics, traces, alert quality and incident analysis, including CloudWatch.

**Outcome:** troubleshoot across the application-to-cloud-network-to-operating-system boundary.

## Track 6 — MATLAB and analytical computing

Refresh MATLAB syntax, arrays, tables, plotting, scripts/functions, data import and numerical workflows. Use small synthetic datasets and focus on reading, maintaining and validating existing code.

## Paused vendor-specific products

Control-M and Axway SecureTransport are not installed in the current practice environment. Do not schedule further hands-on tutorials for these products unless the user confirms access or explicitly asks for a conceptual/simulated lesson.

## Session completion

A topic is complete only when you can:
- explain the design and its trade-offs without relying on notes;
- perform a small practical task;
- diagnose at least one intentional failure;
- state how you would make the change safe in a regulated environment.
