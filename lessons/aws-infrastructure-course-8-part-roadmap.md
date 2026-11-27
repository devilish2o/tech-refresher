# AWS Infrastructure Engineering — 8-part practical course

**Course mapping:** Eight consecutive daily refresher lessons, numbered **016–023** in the existing course. Existing AWS lessons 014 (IAM policy evaluation) and 015 (CloudTrail triage) remain supplementary groundwork; they are not renumbered or marked complete.

**Audience:** Senior DevOps / infrastructure engineer who already has an AWS account but is new to AWS administration.  
**Format:** Eight focused tutorials, approximately 30 minutes each.  
**Command parity:** Every applicable CLI exercise includes Windows PowerShell and Linux Bash. AWS CLI syntax is generally shared; shell syntax, paths, environment variables, JSON tools and installation differ.

## Outcome

Build, secure, operate, monitor, troubleshoot and tear down a small AWS environment. Understand account access, IAM, VPC networking, EC2 compute, storage, application delivery, monitoring and infrastructure as code.

## Safety and cost rules

- Use a dedicated learning account where possible; never experiment in an employer or production account.
- Secure the root user with MFA; do not use root for everyday administration or create root access keys.
- Prefer IAM Identity Center and temporary credentials. Never paste credentials, private keys or tokens into chat, scripts or Git.
- Set a budget and billing alerts before provisioning. Alerts are not hard spending caps.
- Use the smallest suitable resources, record the Region, tag resources and clean up after each lab.
- NAT Gateways, load balancers, public IPv4 addresses, EBS volumes, snapshots, data transfer and some logging features can incur charges. Check current pricing first.
- Prefer Systems Manager Session Manager where supported. Do not expose SSH/RDP broadly.
- No paid resource will be created without the user's explicit approval. Use read-only inspection, local simulation or disposable labs with clear teardown where appropriate.

## Course sequence

### Lesson 016 — AWS account access, console, CLI and cost guardrails
**Objective:** Become comfortable navigating and safely administering the existing AWS account.

Topics: root user versus IAM Identity Center identity, MFA and permission sets; console navigation, account and Region selectors, service search and CloudShell; AWS account, Region, Availability Zone and global versus regional services; AWS CLI v2 installation/checks on Windows and Linux; approved sign-in and STS identity verification; billing dashboard, budgets, alerts and cost visibility.

**Practical:** Sign in using the existing account, identify account/Region, check CLI version and run aws sts get-caller-identity from PowerShell and Bash. Review billing and configure/verify a budget alert if permissions allow. No infrastructure provisioning.

### Lesson 017 — IAM administration and permissions (builds on Lesson 014)
**Objective:** Administer identities and permissions safely.

Topics: IAM users, groups, roles, trust and permission policies; identity-based versus resource-based policies; managed versus inline policies; least privilege, explicit deny, permission boundaries and SCPs; STS temporary credentials, IAM Identity Center profiles and role assumption; Policy Simulator and IAM Access Analyzer.

**Practical:** Inspect the current identity and existing access safely; validate the prefix-scoped S3 policy from Lesson 014 using a simulator or offline test matrix. Do not grant administrator access or create long-lived keys.

### Lesson 018 — VPC networking from first principles
**Objective:** Understand and design AWS network connectivity.

Topics: VPC CIDR and subnet sizing; public/private subnets across Availability Zones; route tables, local routes, Internet Gateway and NAT Gateway; Security Groups (stateful) versus Network ACLs (stateless); DNS attributes, VPC endpoints and Flow Logs; packet path reasoning.

**Practical:** Inspect existing VPCs, subnets, routes and security groups in console and AWS CLI. Draw a two-AZ public/private design and verify route decisions. No paid NAT Gateway creation.

### Lesson 019 — EC2: create and operate a Linux cloud server
**Objective:** Provision, connect to, maintain and remove a virtual machine safely.

Topics: EC2 instances, AMIs, instance families/sizes and launch templates; EBS root/data volumes, snapshots, user data and instance metadata; subnet placement, security groups and addressing; SSH versus Systems Manager Session Manager and instance profiles; instance lifecycle, status checks, stop/start versus terminate and tagging.

**Practical:** Inspect EC2 inventory and pricing first. With explicit approval, launch a smallest-suitable disposable instance using a restricted security group and supported connection method; install a simple web service, validate it, then terminate and verify cleanup. Otherwise complete a console/CLI dry-run and local simulation.

### Lesson 020 — AWS storage: S3, EBS and EFS
**Objective:** Select and operate the appropriate AWS storage model.

Topics: S3 buckets/objects/prefixes, Block Public Access, bucket policies and encryption; versioning, lifecycle and storage classes; EBS block volumes and snapshots versus EFS shared filesystems versus S3 object storage; backup/restore, retention, data protection and cost; AWS CLI object operations and safe deletion.

**Practical:** Inspect storage configuration, then use a private disposable S3 bucket only with approval; upload/download a synthetic file, test access, version/restore and remove the bucket. Explain EBS/EFS trade-offs.

### Lesson 021 — Application delivery: DNS, load balancing and scaling
**Objective:** Understand how an application is exposed and made resilient.

Topics: Route 53 hosted zones and records; Elastic Load Balancing (ALB versus NLB), listeners, target groups and health checks; TLS termination and certificate management; launch templates and Auto Scaling Groups; multi-AZ design, public/private application tiers and outbound access; availability, resilience and cost trade-offs.

**Practical:** Trace a request through DNS → load balancer → target → application. Inspect existing resources or use an architecture/local simulation. Provision a load balancer only with explicit cost approval and teardown plan.

### Lesson 022 — Monitoring, auditing and incident troubleshooting (builds on Lesson 015)
**Objective:** Operate AWS workloads using telemetry and audit evidence.

Topics: CloudWatch metrics, alarms, dashboards, Logs and Logs Insights; CloudTrail management events versus data events and Event history limitations; VPC Flow Logs, EC2 status checks and AWS Config overview; correlate UTC timestamps, event IDs, request IDs, identities and resource IDs; diagnose CPU, disk, network, health-check and AccessDenied incidents; cost and operational alerting.

**Practical:** Analyse synthetic CloudTrail/CloudWatch records and, if available, inspect read-only account telemetry. Explain why S3 data events require explicit configuration and can incur charges.

### Lesson 023 — Terraform and end-to-end AWS operations
**Objective:** Reproduce a small AWS environment predictably and safely.

Topics: Terraform AWS provider, authentication, variables, outputs and state; init/fmt/validate/plan/apply/refresh/destroy; state sensitivity, locking and remote backend concepts; modules, tags, environment separation and drift; CI/CD integration, change review, validation and rollback; cost review, teardown and handover.

**Practical:** Start with local Terraform validation/plan. Provision a minimal AWS resource set only with explicit approval; review plan, apply, validate, make a controlled change, then destroy and verify. Never commit state or secrets.

## PowerShell and Linux Bash parity

For every command-based lesson:
- Show PowerShell and Bash equivalents side by side or in clearly labelled sections.
- Keep AWS CLI arguments and expected AWS-side results equivalent.
- Show platform-specific AWS CLI v2 installation/checks, shell quoting, paths and environment variables.
- Use ConvertFrom-Json/PowerShell objects on Windows and jq or Python on Linux where appropriate.
- State prerequisites and verify tool availability; do not assume jq, Python or AWS CLI is installed.
- Identify console-only tasks honestly; do not invent a CLI equivalent.

## Progress rules

The eight course lessons are **016–023**. Existing lessons **014 (IAM)** and **015 (CloudTrail)** are retained as supplementary AWS lessons and linked to Lessons 017 and 022 respectively. Their completion remains pending until the learner reports completion. Mark each new lesson Delivered when taught and Completed only after the learner reports its checkpoint.
