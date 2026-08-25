# CI Pipeline

This repository uses a reusable GitHub Actions workflow as the standard CI/CD entry point for the todo-service. The reusable workflow is defined in [.github/workflows/golden-path-ci.yml](../.github/workflows/golden-path-ci.yml), and each service team adopts it with a small caller workflow such as [.github/workflows/todo-service-ci.yml](../.github/workflows/todo-service-ci.yml).

## What the reusable workflow does

The reusable workflow is the shared golden path for service teams. It validates code quality, test coverage, Terraform configuration, and deployment readiness without duplicating the same job definitions in every repository.

### `lint`

Purpose:

- Install dependencies for the monorepo
- Run ESLint against both the backend and frontend workspaces

Why it exists:

- Prevents syntax issues and common JavaScript quality problems before code reaches merge or deployment
- Keeps frontend and backend standards consistent across services

### `test`

Purpose:

- Runs backend Jest tests with coverage enabled
- Writes a coverage summary to the GitHub Actions job summary

Why it exists:

- Verifies behavior for the Express API
- Enforces the minimum quality threshold of 80% coverage required by the project
- Makes coverage visible to reviewers in the PR summary

### `security-scan`

Purpose:

- Installs Checkov
- Scans `infra/` for policy violations

Why it exists:

- Validates the Terraform infrastructure against security policies before it is planned or applied
- Helps identify risky defaults early, especially for AWS resources and networking

### `terraform-plan`

Purpose:

- Configures AWS credentials via OIDC
- Installs Terraform
- Runs `terraform init` with the backend key configured for this repo
- Runs `terraform plan` against `infra/stacks/dev`
- Uploads the plan artifact for later apply steps

Why it exists:

- Verifies the IaC renders successfully and can be reviewed before changes are deployed
- Ensures AWS access is granted via the expected role-based federation model rather than static credentials
- Produces an auditable plan for the dev environment

### `docker-build`

Purpose:

- Builds both Docker images in PR validation
- Verifies the backend and frontend container definitions are still valid

Why it exists:

- Catches regressions in Dockerfiles before merge
- Validates container buildability without pushing to registries during pull requests

### `terraform-apply`

Purpose:

- Runs only when `run_terraform_apply` is true
- Uses the same OIDC credential flow
- Applies the dev stack Terraform plan
- Prints the resulting service URL

Why it exists:

- Allows the shared workflow to support a controlled deployment path to AWS for the default branch only
- Keeps apply logic out of pull request validation and restricts production-like changes to the main branch

### `build-and-push`

Purpose:

- Logs into Amazon ECR
- Builds and pushes backend and frontend container images to ECR
- Triggers a force-new-deployment for the ECS service

Why it exists:

- Publishes the built artifacts for the dev environment once Terraform has been applied
- Completes the end-to-end golden path for containerized service delivery

## Required checks and why they matter

Each job is a required gate in the golden path:

- `lint`: catches syntax and static quality regressions early
- `test`: proves the service still behaves as expected and keeps coverage above the lab threshold
- `security-scan`: verifies infrastructure is not introducing obvious security policy violations
- `terraform-plan`: confirms the Terraform stack is valid and ready to review before apply
- `docker-build`: confirms both application images build successfully

Together they provide a practical “minimum viable platform validation” baseline for a service team: code quality, application correctness, IaC quality, and deployment readiness.

## Minimum adopter pattern

A new service team can adopt the standard workflow with a minimal caller similar to this:

```yaml
name: Todo Service CI

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
      run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

This is the minimum pattern for the repo in this lab:

- run the reusable pipeline on PRs and `main`
- validate Terraform on every PR
- apply only on pushes to `main`
- use OIDC with the repository role ARN secret

## OIDC secret configuration

The `terraform-plan` job uses AWS OIDC federation instead of long-lived AWS credentials:

```yaml
permissions:
  id-token: write
  contents: read
```

and then:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.aws_role_arn }}
    aws-region: us-east-1
```

To make this work, the repository must define a GitHub Actions secret named `AWS_ROLE_ARN`.

In GitHub:

1. Open the repository
2. Go to Settings → Secrets and variables → Actions
3. Create a new repository secret named `AWS_ROLE_ARN`
4. Paste the IAM role ARN that trusts GitHub OIDC for this repo
5. Save it

This secret is passed from the caller workflow into the reusable workflow as `aws_role_arn`, which is then used by the Terraform validation job. The secret must be configured as an Actions secret, not a Codespaces secret, because GitHub Actions is the environment executing the workflow.

## Why this pattern is useful

This pattern standardizes the platform contract for service teams:

- the same lint/test/security/Terraform checks apply everywhere
- deployment behavior is consistent
- AWS permissions are federated and short-lived
- each repo only needs a small caller workflow and a role ARN secret

That makes the service team’s adoption effort small while keeping the platform team’s operational guardrails consistent.
