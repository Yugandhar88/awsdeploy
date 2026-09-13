# Deploying the stack with Terraform

Companion to the master task list, tailored to **Terraform** as the IaC tool. Covers the decisions that are specific to Terraform: what it owns, how state and environments are structured, how it maps to the deployment phases, and the gotchas that bite teams.

---

## 1. What Terraform owns (and what it doesn't)

This is the crux. Terraform manages **durable infrastructure**. The **CD pipeline** manages **running code**. Keep the line clean or the two will fight over drift.

**Terraform-managed (changes rarely, on infra PRs):**
- VPC, subnets, route tables, NAT, IGW, security groups, VPC endpoints, flow logs
- Aurora cluster + instances, RDS Proxy, parameter groups, subnet groups
- ECR repositories + lifecycle policies
- ECS cluster, ECS service *definition*, task IAM roles, log groups
- Lambda function *shell* (name, role, VPC config, env, triggers, DLQ)
- ALB, listeners, target groups, ACM certs, Route 53 records, CloudFront, WAF
- IAM roles/policies, KMS keys, Secrets Manager secret *containers*
- CloudWatch alarms, dashboards, auto-scaling policies, budgets

**Pipeline-managed (changes on every app deploy — NOT Terraform):**
- Building and pushing the Docker image to ECR
- Registering new ECS task-definition revisions and updating the service to them
- Publishing new Lambda versions / shifting the alias
- Running database migrations

**The `ignore_changes` boundary.** So the pipeline can update the running image without Terraform reverting it:

```hcl
resource "aws_ecs_service" "api" {
  # ... cluster, load_balancer, network_config ...
  lifecycle {
    ignore_changes = [task_definition, desired_count]
  }
}
```

```hcl
resource "aws_lambda_function" "worker" {
  # ... role, vpc_config, environment ...
  lifecycle {
    ignore_changes = [image_uri]   # or filename / s3_key for zip deploys
  }
}
```

Terraform creates these resources once with a placeholder/initial image; from then on the pipeline (`aws ecs update-service`, CodeDeploy, or `aws lambda update-function-code`) owns what's actually running.

---

## 2. Repository & module structure

```
infra/
├── bootstrap/              # state bucket + lock table — local state, run ONCE
│   └── main.tf
├── modules/                # reusable building blocks
│   ├── network/            # VPC, subnets, NAT, endpoints, SGs
│   ├── security/           # IAM roles, KMS, GuardDuty/Config wiring
│   ├── data/               # Aurora, RDS Proxy, secrets
│   ├── ecr/                # repositories + lifecycle
│   ├── ecs-service/        # cluster, service, task role, autoscaling
│   ├── lambda/             # function shell, triggers, DLQ
│   ├── edge/               # ALB, ACM, Route 53, CloudFront, WAF
│   └── observability/      # log groups, alarms, dashboards
└── envs/
    ├── dev/
    │   ├── backend.tf      # points at dev state
    │   ├── main.tf         # calls modules with dev inputs
    │   └── terraform.tfvars
    ├── staging/
    └── prod/
```

Root configs live under `envs/` and compose the shared child modules. Modules take inputs and expose outputs; they never hardcode environment values.

---

## 3. State management

Use a remote **S3 backend** with locking. Encrypt the state, version the bucket, and lock access down — state contains resource metadata and sometimes secrets.

```hcl
terraform {
  backend "s3" {
    bucket       = "myco-tfstate-prod"
    key          = "app/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true          # S3-native locking (modern)
    # dynamodb_table = "tf-locks" # older lock mechanism, still valid
  }
}
```

**The bootstrap chicken-and-egg:** Terraform needs the state bucket to exist before it can use it as a backend, but you want Terraform to create that bucket. Resolve it with the `bootstrap/` config that runs with **local state** to create the bucket + lock table once, then have every other config use the S3 backend. (Alternatively create those two resources by hand.) Enable `prevent_destroy` on the state bucket.

**Separate state per environment** — never one state file for dev + prod. A blast radius in dev must not be able to touch prod. In a multi-account setup, each env's state lives in its own account.

---

## 4. Environment strategy

Two common patterns — pick one deliberately:

- **Directory + backend per environment (recommended for multi-account/prod).** Each `envs/<name>/` has its own `backend.tf` and `.tfvars`. Explicit, isolated blast radius, easy to run different versions per env, works cleanly across separate AWS accounts. More boilerplate.
- **Terraform workspaces.** One config, `terraform workspace select prod`. Less duplication, but easy to apply to the wrong workspace by accident and awkward across accounts. Fine for lightweight/single-account setups; risky for prod isolation.

Default to directory-per-env when prod matters.

---

## 5. Providers & versioning

Pin everything so applies are reproducible:

```hcl
terraform {
  required_version = "~> 1.9"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
  default_tags {
    tags = {
      Environment = var.environment
      Application = "web-app"
      ManagedBy   = "terraform"
    }
  }
}
```

`default_tags` applies your tagging strategy to every resource automatically. Commit the `.terraform.lock.hcl` provider lock file.

---

## 6. Module mapping to the deployment phases

How the master task list's phases translate to Terraform:

- **Phase 0 governance** — Organizations/accounts often a separate bootstrap or a dedicated Control Tower setup; Terraform manages budgets, tagging (`default_tags`), and the state backend.
- **Phase 1 security** → `modules/security`: IAM roles, KMS, and enabling GuardDuty/Config/Security Hub. Set up **GitHub OIDC** as an IAM role so CI assumes it — no static keys.
- **Phase 2 networking** → `modules/network`.
- **Phase 3 data** → `modules/data`: Aurora, RDS Proxy, secrets. `prevent_destroy` on the cluster.
- **Phase 4 registry** → `modules/ecr`.
- **Phase 5 ECS** → `modules/ecs-service` (with the `ignore_changes` boundary above).
- **Phase 6 Lambda** → `modules/lambda` (shell only; code from pipeline).
- **Phase 7 edge** → `modules/edge`.
- **Phases 8–11 pipeline, migrations, deploy, verify** — mostly **outside Terraform**, in CI/CD. Terraform provisions the pipeline's IAM roles and (if using native AWS CI) the CodePipeline/CodeBuild resources.
- **Phase 12 observability / 13 reliability / 14 cost** → `modules/observability` + autoscaling resources; budgets in the root.
- **Phase 15 DR** — backup config and cross-region snapshot settings are Terraform; restore drills are operational.

---

## 7. The Terraform workflow in CI/CD

Run Terraform through the pipeline, not laptops. Per environment:

**On pull request (plan only):**
1. `terraform fmt -check`
2. `terraform validate`
3. `tflint` + a security scanner (`tfsec` / `checkov`)
4. `terraform plan` → post the plan as a PR comment for review

**On merge to main (apply):**
5. `terraform apply` to **dev** automatically
6. Promote to **staging**
7. **Manual approval gate**
8. `terraform apply` to **prod**

Each stage assumes a **per-environment OIDC role** with least privilege. Never store long-lived AWS keys in CI.

---

## 8. Terraform-specific gotchas

- **Secrets in state.** Terraform can write secret values into state in plaintext. Prefer creating the Secrets Manager *container* in Terraform and letting the value be set out-of-band or rotated; mark variables `sensitive = true`; always encrypt state and restrict bucket access.
- **The image boundary (again).** The single biggest source of confusion — get `ignore_changes` right on ECS/Lambda or Terraform and your pipeline will thrash.
- **`prevent_destroy` on stateful resources.** Aurora, the state bucket, and anything holding data should refuse accidental destroys.
- **Avoid `-target`.** Reaching for targeted applies to force ordering usually signals a module dependency that should be expressed with proper `depends_on` / outputs instead.
- **Drift detection.** Run a scheduled `terraform plan` to catch out-of-band console changes; alert if the plan is non-empty.
- **Provider + module version pinning.** Unpinned versions make "it worked yesterday" applies fail. Commit lock files.
- **Data sources for cross-stack refs.** If networking and app are separate states, read VPC/subnet IDs via `terraform_remote_state` or `aws_*` data sources rather than hardcoding.
