# Lesson 016 — AWS account access, console, CLI and cost guardrails

**Domain:** AWS foundations  
**Duration:** 30 minutes  
**Audience:** Senior DevOps / infrastructure engineer refreshing AWS administration.  
**Status:** Delivered; completion pending user confirmation.  
**Safety:** Existing account only. This lesson is read-only; do not create cloud resources or access keys.

## Objective

Safely access the user's existing AWS account, identify the active identity and Region, verify AWS CLI v2, authenticate using an approved method, call STS to confirm identity, and inspect billing/cost controls.

## Prerequisites

- An existing AWS account and an authorised sign-in method.
- Windows 11 PowerShell and/or a Debian/Ubuntu Bash shell.
- Browser access to the AWS Management Console.
- Do not use an employer or production account for experiments.
- Never paste account IDs, ARNs, credentials, MFA codes, private keys or tokens into chat or Git.

## Timed plan

| Time | Activity |
|---|---|
| 00–04 | Sign in and identify account identity |
| 04–09 | Check root security and MFA |
| 09–14 | Explore console and Regions |
| 14–20 | Verify/install AWS CLI v2 |
| 20–26 | Authenticate and verify STS identity |
| 26–30 | Inspect billing and cost controls |

## 1. Sign in and identify the account (4 minutes)

Open the [AWS Management Console](https://console.aws.amazon.com/).

Use the existing account's applicable sign-in method:

- **Root user:** the email address used to register the AWS account.
- **IAM user:** account ID or alias, IAM username and password.
- **IAM Identity Center:** the organisation's AWS access portal.

In the account menu, identify the account and current identity/role. Do not share the account ID or ARN.

Root has unrestricted account access and is not equivalent to a normal administrator role. Do not create root access keys; reserve root for tasks that specifically require it.

## 2. Root security and MFA (5 minutes)

If authorised to inspect root security:

1. Open IAM and the root user's security credentials, or use the account menu's security credentials link.
2. Check whether MFA is configured.
3. If not, follow AWS's current MFA setup guidance.
4. Prefer a phishing-resistant passkey or security key where supported; an authenticator app is an alternative.

Target operating model:

- Root: emergency and root-only operations.
- IAM Identity Center: normal human access.
- Temporary credentials: preferred for CLI and SDK access.
- Long-term access keys: avoid unless there is a justified requirement.

Do not create users, keys or change permissions just to complete this lesson. Lesson 017 covers IAM administration.

## 3. Console navigation and Regions (5 minutes)

Explore these console areas without changing anything:

| Area | Operational purpose |
|---|---|
| IAM | Identities, roles and policies |
| EC2 | Virtual machines and related resources |
| VPC | Cloud networking |
| S3 | Object storage |
| CloudWatch | Metrics, logs and alarms |
| Billing and Cost Management | Spending visibility |

Select **Canada (Central), `ca-central-1`** in the Region selector for future exercises unless a lesson states otherwise.

A Region is a geographical AWS location. Many resources are regional; some services and resources are global. Changing the console's Region selector does not move resources or change their Region.

Also identify the account menu, service search and CloudShell entry. CloudShell is a browser-based shell; it is not the same thing as the locally installed AWS CLI.

## 4. Verify or install AWS CLI v2 (6 minutes)

Assumption: AWS CLI v2. Commands below are shown for both Windows PowerShell and Linux Bash.

### Windows 11 — PowerShell

Check installation:

```powershell
Get-Command aws -ErrorAction SilentlyContinue
aws --version
```

If missing, download the official MSI from [AWS CLI v2 for Windows](https://awscli.amazonaws.com/AWSCLIV2.msi), verify that the download is from the official AWS domain, and run the installer. Then open a new PowerShell window:

```powershell
aws --version
```

Expected format: `aws-cli/2.x.x ... Windows ...`.

### Debian/Ubuntu — Bash

Check installation:

```bash
command -v aws
aws --version
```

If missing, use AWS's official Linux installation guide: [Installing or updating the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). Follow its architecture-specific installation and signature-verification steps. Do not run an installer script fetched from an unverified source.

Then verify:

```bash
aws --version
```

Expected format: `aws-cli/2.x.x ... Linux ...`.

## 5. Authenticate and verify identity (6 minutes)

Use the method your account administrator has approved. Do not create long-lived access keys as a workaround for a failed login.

### Option A — IAM Identity Center

In PowerShell or Bash:

```bash
aws configure sso
```

Follow the prompts using values supplied by your organisation:

- SSO session name: `aws-learning`
- SSO start URL: organisation's access portal URL
- SSO Region: the Region where Identity Center is configured
- Select the authorised account and permission set
- CLI default Region: `ca-central-1`
- Output format: `json`
- Profile name: `aws-learning`

Authenticate:

```bash
aws sso login --profile aws-learning
```

Verify:

```bash
aws sts get-caller-identity --profile aws-learning --region ca-central-1
```

### Option B — Browser-based `aws login`

AWS CLI v2.32.0 or later supports browser-based sign-in with supported IAM user or root credentials. Check the installed version first:

```bash
aws --version
```

If the account and CLI version support this method, follow the current [AWS CLI sign-in documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sign-in.html). The basic flow is:

```bash
aws login --region ca-central-1
aws sts get-caller-identity --region ca-central-1
```

For a root-user session, remember that CLI calls retain root-level authority. Use only for this short read-only identity check, then end the session:

```bash
aws logout
```

For IAM users, required permissions and supported sign-in conditions apply. If access is denied, stop and ask the account administrator to provide the approved access method; do not broaden permissions yourself.

### Inspect local CLI configuration

These commands work in PowerShell and Bash:

```bash
aws configure list
aws configure list-profiles
```

Default file locations:

| Windows | Linux |
|---|---|
| `C:\\Users\\<username>\\.aws\\config` | `~/.aws/config` |
| `C:\\Users\\<username>\\.aws\\credentials` | `~/.aws/credentials` |

Do not display, upload or commit credentials files. They may contain secrets or sensitive authentication material.

### Expected STS result

The response is JSON similar to:

```json
{
  "UserId": "EXAMPLE-USER-ID",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/example-user"
}
```

Interpretation:

- `Account`: AWS account handling the request.
- `Arn`: identity making the API request.
- `UserId`: identifier AWS associates with that identity.

An assumed role or Identity Center session normally produces an assumed-role ARN rather than an IAM user ARN. The command is read-only and creates no infrastructure.

## 6. Billing and cost guardrails (4 minutes)

Open **Billing and Cost Management** and inspect:

1. Billing dashboard.
2. Bills/current charges.
3. Cost Explorer, if enabled and available.
4. Budgets and existing alerts.

Know the common cost-bearing resources before future labs:

| Resource/service | Potential cost |
|---|---|
| EC2 | Compute while running |
| EBS volumes and snapshots | Storage |
| NAT Gateways | Hourly and data-processing charges |
| Load balancers | Hourly and usage charges |
| Public IPv4 addresses | May incur charges |
| Data transfer | Especially outbound transfer |
| CloudWatch | Log ingestion, storage and other usage |

AWS Budgets can alert on actual or forecasted costs, but updates are periodic and an alert is **not** a hard spending cap. If no budget exists, record that as a follow-up; do not assume the Free Tier makes every usage pattern free.

No resources are created in this lesson.

## Exercise and expected outcome

Complete the following using the existing account:

1. Identify the current identity type (root, IAM user or assumed role/Identity Center).
2. Check root MFA status if authorised.
3. Select Canada Central (`ca-central-1`) in the console.
4. Verify AWS CLI v2 locally.
5. Authenticate using the approved method, or record that an administrator must first provide one.
6. Run `aws sts get-caller-identity` and confirm that the returned account/identity matches the intended account.
7. Inspect billing and existing budgets without changing resources.

**Quick self-check:** Can you explain which identity signed the STS request, which account it belongs to, which Region your next CLI command will target, and where you would look for current charges?

## Production and interview relevance

These checks prevent wrong-account/wrong-Region operations, routine use of root, unsafe long-lived credentials and unobserved cloud spend. Be ready to distinguish **authentication** (who you are), **authorization** (what you may do), and **account/Region selection** (where the operation is directed).

## Completion checkpoint

Report only non-sensitive status; never send account IDs, ARNs, credentials, MFA codes or configuration files.

```text
Lesson 016 — AWS foundations

[ ] Existing AWS account accessed
[ ] Root MFA checked (if authorised)
[ ] Account identity identified
[ ] Canada Central (ca-central-1) selected
[ ] AWS CLI v2 verified or installed
[ ] Approved CLI authentication tested, or deferred
[ ] STS caller identity verified
[ ] Billing and existing budgets inspected

CLI version:
Authentication method: SSO / aws login / console only
STS identity check: Passed / Not completed
Root MFA: Enabled / Already enabled / Needs attention
```

**Status:** Delivered; completion remains pending the learner's report.

## Official references

1. [AWS Management Console](https://console.aws.amazon.com/)
2. [AWS CLI v2 installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
3. [AWS CLI sign-in documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sign-in.html)
4. [IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
5. [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)

## For Dummies 🧸

Think of AWS as a huge building with thousands of rooms:

- Your AWS account is your building.
- Root is the master key.
- IAM decides which keys other people receive.
- MFA is an extra lock on the door.
- A Region is the geographical location of the resources.
- The Console is the control panel.
- The CLI is a remote control operated from your computer.
- Billing tells you what the building is costing you.

Today we check that you have the right key, know which building you are controlling, and can see what it costs. We do not build anything yet.
