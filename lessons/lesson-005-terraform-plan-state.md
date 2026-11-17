# Lesson 005 — Terraform: review a saved plan and inspect state safely

**Domain:** Infrastructure as Code / Terraform  
**Duration:** 30 minutes  
**Environment:** PEACEKEEPER, Debian 13, Bash and Vim  
**Version assumption:** Terraform CLI 1.4.0 or later. The `terraform_data` resource was introduced in 1.4. This lab was checked against the current HashiCorp documentation on 2026-11-16.

## Objective

Practise a safe Terraform change lifecycle without connecting to a cloud account or installing a provider:

1. Model a release record with the built-in `terraform_data` resource.
2. Format and validate HCL.
3. Save a plan, inspect it, then apply that exact plan.
4. Inspect the resulting local state.
5. Change a release identifier and recognise a planned replacement.
6. Destroy the disposable lab resource and leave the state empty.

This is a local lifecycle simulation, not a deployment. No cloud credentials, provider plugins, remote infrastructure or production endpoints are used.

## Prerequisites and safety boundary

- Bash, Vim and Terraform CLI 1.4.0+.
- No provider installation or cloud account is required.
- All files and state stay under `~/labs/tech-refresher/lesson-005-terraform-plan-state`.
- The default local backend is used. Do not point this lab at a real backend.
- The only managed object is Terraform's built-in `terraform_data` state record; it does not create an external service.

Check the CLI:

```bash
command -v terraform
terraform version
```

If `terraform` is missing, use HashiCorp's official installation instructions and verify the installed version before continuing. Do not use an unverified binary or an unrelated third-party installer. If the CLI is unavailable during this session, complete the HCL review and resume the execution steps once it is installed.

## 30-minute plan

| Time | Activity |
|---:|---|
| 0–4 min | Check Terraform and create an isolated lab directory |
| 4–12 min | Write the configuration and ignore generated state/plan files |
| 12–18 min | Initialise, format, validate and create a saved plan |
| 18–24 min | Review/apply the plan and inspect local state |
| 24–28 min | Preview a replacement and destroy the lab resource |
| 28–30 min | Verify cleanup and answer the self-check |
| **Total** | **30 minutes** |

## 1. Create the isolated lab (4 minutes)

```bash
mkdir -p ~/labs/tech-refresher/lesson-005-terraform-plan-state
cd ~/labs/tech-refresher/lesson-005-terraform-plan-state
pwd
```

Expected path:

```text
/home/<your-user>/labs/tech-refresher/lesson-005-terraform-plan-state
```

Create the files with Vim. Do not put real environment names, endpoints, credentials or secrets in them.

## 2. Write the configuration (8 minutes)

Create `main.tf`:

```bash
vim main.tf
```

Paste:

```hcl
terraform {
  required_version = ">= 1.4.0"
}

resource "terraform_data" "release" {
  input = {
    environment = var.environment
    artifact    = "orders-api.war"
    release_id  = var.release_id
  }

  # A changed release ID forces replacement of this local record.
  triggers_replace = [var.release_id]
}
```

Create `variables.tf`:

```bash
vim variables.tf
```

Paste:

```hcl
variable "environment" {
  description = "Disposable target label for this simulation."
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "test"], var.environment)
    error_message = "For this lab, environment must be dev or test."
  }
}

variable "release_id" {
  description = "Synthetic release identifier."
  type        = string
  default     = "2026.11.16-001"

  validation {
    condition     = length(trimspace(var.release_id)) > 0
    error_message = "release_id must not be empty."
  }
}
```

Create `outputs.tf`:

```bash
vim outputs.tf
```

Paste:

```hcl
output "release_record" {
  description = "The synthetic release record stored in Terraform state."
  value       = terraform_data.release.output
}
```

Create `.gitignore`:

```bash
vim .gitignore
```

Paste:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
crash.log
crash.*.log
```

Important: keep `.terraform.lock.hcl` under version control in real projects when providers are used. Do not commit state files or saved plan files. State and plan files can contain sensitive values even when terminal output masks them.

## 3. Initialise, format, validate and plan (6 minutes)

Run:

```bash
terraform init
terraform fmt -check
terraform validate
terraform plan -out=initial.tfplan
```

Expected:

- `init` succeeds using the local backend; no cloud provider should be downloaded.
- `fmt -check` reports no formatting changes.
- `validate` reports the configuration is valid.
- The plan proposes one creation:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
```

Review the saved plan before applying it:

```bash
terraform show initial.tfplan
```

Check the resource address is exactly `terraform_data.release`, and the values are the synthetic `dev`, `orders-api.war` and `2026.11.16-001`.

## 4. Apply that exact plan and inspect state (6 minutes)

Only after reviewing the plan, run:

```bash
terraform apply initial.tfplan
terraform state list
terraform state show terraform_data.release
terraform output -json release_record
```

Expected:

- Apply completes with one resource added.
- `terraform state list` contains `terraform_data.release`.
- `state show` displays the synthetic input and output attributes.
- The JSON output contains the release record.

**Operational point:** supplying a saved plan to `terraform apply` is itself approval; Terraform does not prompt again. The saved plan fixes the actions to execute. In a real pipeline, the reviewed plan must be protected, associated with the correct commit/workspace, and applied with controlled access. Never blindly apply a plan produced by an untrusted or stale workflow.

Check for drift between configuration and state without changing anything:

```bash
terraform plan -detailed-exitcode
rc=$?
printf 'terraform plan exit code: %s (0 means no changes; 2 means changes; 1 means error)\n' "$rc"
test "$rc" -eq 0
```

Expected: exit code `0`, meaning no changes are proposed.

## 5. Preview a replacement, then clean up (4 minutes)

Change only the synthetic release identifier through a command-line variable override:

```bash
terraform plan \
  -var='release_id=2026.11.16-002' \
  -out=release-change.tfplan

terraform show release-change.tfplan
```

Expected: Terraform plans to replace `terraform_data.release` because `triggers_replace` changed. Look for the replacement action `-/+` and a summary of one add and one destroy.

Do **not** apply this second plan. This exercise is about recognising the lifecycle change before approval.

Now destroy the original lab resource using a reviewed destroy plan:

```bash
terraform plan -destroy -out=cleanup.tfplan
terraform show cleanup.tfplan
terraform apply cleanup.tfplan
terraform state list
```

Review that the destroy plan addresses only `terraform_data.release` before applying it. Expected: one resource destroyed and `terraform state list` prints no resource addresses.

The lab directory and its local state can be retained for review. If you remove it later, first confirm that the state list is empty.

## Quick self-check

1. Why does changing `release_id` force replacement, while changing only `environment` would update the stored input?
2. What does `terraform plan` do that `terraform apply` does not?
3. Why is applying a saved plan non-interactive, and why must the plan file be protected?
4. What does `terraform state show` reveal that `terraform state list` does not?
5. Why should a production team use a secured remote backend with state locking rather than sharing local state files?

**Expected outcome:** a valid local Terraform configuration; one reviewed creation plan; a successful local apply and state inspection; a visible replacement proposal; and an empty state after cleanup.

## Production and interview relevance

This is the core change-control pattern for Terraform: initialise the correct working directory, validate, create a plan, review its resource actions, then apply the exact approved plan. In production, state is a sensitive inventory and coordination mechanism. Protect it with backend access controls, encryption, backups and locking; avoid concurrent applies; pin Terraform/provider versions; and never edit state JSON manually.

Be ready to explain the difference between a speculative plan and a saved plan, why a saved plan can become stale, what `-/+` means, and how you would handle unexpected drift without making an unreviewed change.

## Completion checkpoint

Report:

- Terraform version.
- Whether `init`, `fmt -check` and `validate` passed.
- Initial plan summary.
- Whether `terraform apply initial.tfplan` succeeded.
- Whether the state inspection showed the expected release record.
- Whether the second plan showed a replacement.
- Whether cleanup left `terraform state list` empty.
- One production safeguard you would insist on.
- Confidence from 1–5.

Do not paste state files, plan files, credentials or sensitive configuration. Do not claim completion until you have run the checks.

## References

- [HashiCorp — Terraform `terraform_data` resource](https://developer.hashicorp.com/terraform/language/resources/terraform-data)
- [HashiCorp — Create a Terraform plan](https://developer.hashicorp.com/terraform/tutorials/cli/plan)


## In simple words — what did we actually do?

Terraform is a tool that lets us describe the infrastructure we want in configuration files instead of creating everything manually.

In this exercise, we did not create real servers or cloud resources. We used Terraform's built-in `terraform_data` resource to simulate a small deployment record for a Java application.

Imagine the record contains:

- Environment: DEV
- Application: `orders-api.war`
- Release: `2026.11.16-001`

We then practised four important ideas:

1. **Configuration — describe what we want.** The HCL files tell Terraform what resource and values should exist.
2. **Plan — preview the changes.** Terraform compares the configuration with its state and tells us what it intends to create, change or destroy.
3. **Apply — carry out the reviewed changes.** Terraform executes the actions in the plan we approved.
4. **State — keep an inventory of what Terraform manages.** The state records the resource and its attributes so Terraform can compare the desired configuration with what it previously managed.

We changed the release ID from `2026.11.16-001` to `2026.11.16-002`. Because we deliberately configured `triggers_replace`, Terraform proposed replacing the simulated record. We inspected that proposal but did not apply it. Finally, we created and reviewed a destroy plan and removed the lab resource.

### Why does this matter?

Imagine provisioning application infrastructure for development, testing and production. Instead of manually creating each resource, a team can describe the required infrastructure as code and use Terraform to make repeatable, reviewable changes.

The plan is especially important in regulated environments: it gives engineers and approvers a chance to understand the impact before changes are made.

### How does Terraform fit into a DevOps delivery flow?

| Technology | Responsibility |
|---|---|
| Jenkins | Orchestrates the delivery workflow. |
| Maven / Ant | Builds the Java application. |
| Terraform | Creates and manages the required infrastructure. |
| Docker | Packages the application and its dependencies. |
| Kubernetes | Runs and manages containerised applications. |

A pipeline might build and test a WAR with Maven, generate a Terraform plan, request approval, apply the approved infrastructure changes, deploy the application, and validate the result.

### Three things to remember

- **Infrastructure as Code:** describe infrastructure in files so it can be reviewed and reproduced.
- **Plan before Apply:** inspect the proposed changes before executing them.
- **Protect state:** it is Terraform's record of managed resources and can contain sensitive information. Secure it and avoid editing it manually.

The lab was a safe simulation of Terraform's change lifecycle, not a real infrastructure deployment.
