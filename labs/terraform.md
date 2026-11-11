# Lab: Terraform safely

## Goal

Understand the plan/apply lifecycle, state, drift and change review using a local or disposable test target.

## Exercises

1. Initialise a minimal configuration and inspect provider requirements.
2. Declare variables and outputs; avoid hard-coded environment-specific values.
3. Run `terraform fmt`, `validate` and `plan`; inspect every proposed change.
4. Apply only to an isolated sandbox, then inspect state and outputs.
5. Change one property and observe the plan.
6. Simulate drift and understand refresh behaviour.
7. Explore modules and remote state conceptually before enabling shared infrastructure.

## Safety

Never point the exercise at a production account. Protect state files: they may contain secrets. Do not commit state, credentials or plan files to Git.

## Self-check

Explain desired state, state file, provider, resource graph, drift, locking, import and why a reviewed plan matters.

## Documentation

- [Terraform documentation](https://developer.hashicorp.com/terraform/docs)
- [CLI workflow](https://developer.hashicorp.com/terraform/cli/run)
- [State](https://developer.hashicorp.com/terraform/language/state)
