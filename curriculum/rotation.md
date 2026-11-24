# Subject rotation

Use this as a guide, not a rigid timetable. Prefer the next unfinished prerequisite or the topic most useful to current work. **AWS should appear regularly and receive more weight** than a single slot in a long cycle.

| Session | Main focus | Practical emphasis |
|---|---|---|
| 1 | AWS IAM / security | roles, policy evaluation, least privilege and audit |
| 2 | Python / automation | automate a repeatable operational task |
| 3 | Jenkins / CI/CD | build, test and archive an artifact |
| 4 | AWS networking | VPC, subnets, routes, security groups and DNS |
| 5 | Terraform / AWS | plan and review an isolated infrastructure change |
| 6 | Docker / ECR | build, inspect and publish only in a safe test context |
| 7 | Linux / Windows | collect diagnostics and automate a task |
| 8 | Kubernetes / EKS | deploy and troubleshoot a local workload; compare EKS architecture |
| 9 | AWS operations | EC2, S3, RDS, load balancing, Systems Manager or cost controls |
| 10 | Java build / WAR | Maven or Ant, tests and immutable artifact promotion |
| 11 | Networking / observability | TLS, CloudWatch, CloudTrail and incident correlation |
| 12 | MATLAB / review / Git | refresh a skill and consolidate gaps |

Repeat the cycle at a higher level of difficulty. Insert security, incident response and architecture reviews where they reinforce a module.

## Paused subjects

Control-M and Axway SecureTransport are not installed in the current practice environment. Do not schedule new hands-on tutorials for either product unless the user confirms an appropriate environment or explicitly requests a conceptual/simulated lesson.

## AWS safety

- Prefer local simulations, read-only AWS CLI/console inspection or eligible free-tier resources.
- State the region and assumptions.
- Identify resources that may incur charges before creation.
- Never create paid resources without explicit user approval.
- Include least-privilege IAM, validation and cleanup in any cloud lab.

## Evening deep dives

Do not create additional schedules from within a daily lesson. Longer evening labs are separate and optional, at most twice per week, and should extend a topic only when the user asks for them.
