# AWS Infrastructure Engineering — 8-part practical course

**Audience:** Senior DevOps / infrastructure engineer learning AWS from the beginning.  
**Format:** Eight focused tutorials, approximately 30 minutes each.  
**Approach:** Console walkthrough plus AWS CLI, with equivalent PowerShell and Linux Bash commands wherever commands apply.

## Course outcome

By the end, build, secure, operate, monitor, troubleshoot and tear down a small AWS environment. Understand how AWS account access, IAM, VPC networking, EC2 compute, storage, load balancing, monitoring and infrastructure as code fit together.

## Safety and cost rules

- Use a dedicated learning account where possible; never experiment in an employer or production account.
- Secure the root user with MFA and do not use it for everyday administration.
- Prefer IAM Identity Center / temporary credentials; do not create root access keys or paste credentials into lessons, chat, scripts or Git.
- Set a budget and billing alerts before creating billable resources. Alerts are not hard spending caps.
- Use the smallest suitable resources, note the Region, and delete resources at the end of each lab.
- NAT Gateways, load balancers, public IPv4 addresses, EBS volumes, snapshots, data transfer and some logging features can incur charges. Verify current pricing before enabling them.
- Never expose SSH/RDP to the world except for a tightly controlled, temporary lab where explicitly justified; prefer Systems Manager Session Manager where supported.
- All examples use fictional names, disposable resources and non-sensitive data.

## Tutorial 1 — AWS account, sign-in, console and cost guardrails

**Objective:** Understand the AWS account boundary and safely access the Management Console and CLI.

Topics:
- AWS account, Region, Availability Zone and global services.
- Root user versus IAM Identity Center user, permission sets and IAM roles.
- MFA, recovery details and safe administrative access.
- AWS Management Console, service search, CloudShell and account/Region selectors.
- Install/check AWS CLI v2; authenticate with temporary credentials; verify identity with STS.
- Billing dashboard, Free Tier/Free Plan eligibility as applicable, AWS Budgets and cost alerts.

Practical outcome: securely sign in, identify the active account and Region, run aws sts get-caller-identity, and establish a budget alert before provisioning infrastructure.

## Tutorial 2 — IAM administration and AWS CLI operations

**Objective:** Grant a human and a workload only the permissions each needs.

Topics:
- Users, groups, roles, trust policies, identity policies and resource policies.
- Managed versus customer-managed policies; policy evaluation and explicit deny.
- AssumeRole and STS temporary credentials.
- IAM Access Analyzer and policy simulator.
- AWS CLI profiles and IAM Identity Center SSO.

Practical outcome: create a narrowly scoped read-only role/policy for a lab resource, test allowed and denied actions, and use the same identity from the console and CLI.

## Tutorial 3 — VPC networking from first principles

**Objective:** Build and troubleshoot an isolated AWS network.

Topics:
- VPC CIDR, subnet sizing, public and private subnets across Availability Zones.
- Route tables, local routes, Internet Gateway and NAT Gateway (including cost implications).
- Security Groups (stateful) versus Network ACLs (stateless).
- DNS settings, VPC endpoints and VPC Flow Logs.
- Trace a packet path and diagnose routing versus firewall failures.

Practical outcome: create a small VPC with public/private subnet design, inspect routes and security rules, and verify expected reachability without exposing private resources.

## Tutorial 4 — EC2 compute: create and operate a cloud server

**Objective:** Provision, connect to, maintain and safely remove a Linux virtual machine.

Topics:
- EC2 instances, AMIs, instance families/sizes, tenancy and key pairs.
- EBS root/data volumes, snapshots, user data and instance metadata.
- Security Groups and subnet placement.
- SSH versus Systems Manager Session Manager.
- Instance states, stop/start versus terminate, status checks and recovery.
- Tags and basic operational inventory.

Practical outcome: launch a small disposable Linux EC2 instance, connect safely, install a simple service, validate it, then terminate it and verify associated resources are cleaned up.

## Tutorial 5 — AWS storage and data services

**Objective:** Select and operate the right AWS storage type.

Topics:
- S3 buckets, objects, prefixes, versioning, lifecycle and storage classes.
- Bucket policies, Block Public Access, encryption and access logging.
- EBS block storage versus EFS shared file storage versus S3 object storage.
- Snapshots, backup and restore considerations.
- AWS CLI object operations and safe deletion.

Practical outcome: create a private S3 bucket, upload/download a sample file, test access controls, enable versioning, restore an earlier object version, and remove the lab bucket safely.

## Tutorial 6 — Cloud networking services and application delivery

**Objective:** Expose an application safely and understand highly available AWS network patterns.

Topics:
- Route 53 hosted zones and DNS records.
- Elastic Load Balancing: Application Load Balancer versus Network Load Balancer.
- Target groups, health checks, listeners and TLS termination.
- Auto Scaling Groups and launch templates.
- Public/private application tiers and outbound connectivity choices.
- Architecture trade-offs, resilience and cost.

Practical outcome: deploy a simple disposable web service behind a load balancer or use a local/design simulation if cost controls make provisioning inappropriate; test health checks and explain failover behaviour.

## Tutorial 7 — Monitoring, logging, auditing and troubleshooting

**Objective:** Operate an AWS workload using evidence rather than guesswork.

Topics:
- CloudWatch metrics, dashboards, alarms and Logs.
- CloudTrail management events versus data events.
- VPC Flow Logs and EC2 status checks.
- AWS Config overview and configuration drift.
- CloudWatch Logs Insights basics.
- Correlate timestamps, request IDs, identities and resource IDs.
- Diagnose CPU, disk, network, health-check and AccessDenied incidents.

Practical outcome: create a metric alarm or log filter, inspect a CloudTrail event, and work through a synthetic incident using timestamps and resource identifiers.

## Tutorial 8 — Infrastructure as Code and end-to-end operations

**Objective:** Reproduce a small AWS environment predictably and tear it down safely.

Topics:
- Terraform AWS provider, authentication, variables, outputs and state.
- Plan/apply/destroy lifecycle; state security and remote-state concepts.
- Reusable modules, tagging and environment separation.
- Drift detection and change review.
- CI/CD integration concepts and deployment rollback.
- Cost review, cleanup verification and operational handover.

Practical outcome: use Terraform to provision a minimal, tagged lab network/resource set, inspect the plan, apply, validate, make a controlled change, and destroy the environment. No secrets in source control or state files.

## Command parity requirement

Every AWS lesson must provide aligned examples for PowerShell and Linux Bash. AWS CLI commands are generally identical across operating systems; shell quoting, paths, environment variables, JSON processing and installation steps differ and must be shown separately. State prerequisites and expected output. Do not assume tools are installed.

## Progress tracking

Record each tutorial as Not started, Delivered, or Completed. A tutorial is only Completed after the learner reports the checkpoint. The previous standalone AWS IAM and CloudTrail lessons (014 and 015) are useful supplementary material; this eight-part course deliberately starts with account access and foundational AWS operations.
