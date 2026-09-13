# AWS Deployment Kit — ECS + Aurora + Lambda

Everything needed to plan, execute, and reproduce the deployment of a customer-facing web application on AWS. Start here, then work through the files in order.

## The system at a glance

A customer-facing web app with this shape:

```
User → Route 53 (DNS) → CloudFront/WAF → ALB (L7)
     → ECS Fargate (API)  ─┐
     → Lambda (async work) ─┼→ Aurora PostgreSQL (via RDS Proxy)
                            ┘
```

- **Edge:** Route 53, CloudFront, WAF, ACM, Shield
- **Compute:** ECS Fargate (API) in private-app subnets; Lambda for event-driven work
- **Data:** Aurora PostgreSQL (writer + reader) with RDS Proxy, in private-data subnets
- **Network:** VPC across ≥2 AZs; public (ALB/NAT) + private-app + private-data subnets; security groups chained ALB → ECS → Aurora
- **External:** payment gateway, email provider

## What's in this kit

| File | What it is | Read it when |
|------|------------|--------------|
| `README.md` | This index and the shared principles | First — start here |
| `01-deployment-task-list.md` | The master checklist: ~17 phases from account setup to go-live, each task tagged **(SA)** architect / **(Dev)** developer / **(Both)** | Planning and executing the build — the *what* and *in what order* |
| `02-terraform-guide.md` | Terraform-specific implementation: module/repo structure, remote state, environment strategy, phase→module mapping, the infra-vs-pipeline boundary | Implementing the infrastructure as code — the *how* |
| `03-prompt.md` | A self-contained prompt that regenerates this entire kit (architecture, C4 diagrams, task list, Terraform guide) in a fresh AI session | Reproducing, extending, or adapting the kit to a different stack |

## Suggested path

1. Read the architecture summary above and skim **01** to see the full scope.
2. Work the phases in **01** in order — foundation first, then the repeatable pipeline, then the cross-cutting concerns.
3. When you reach the IaC phases, switch to **02** for the Terraform specifics.
4. Keep **03** for when you want to regenerate the material or spin up a variant (different database, region, or compute model).

## Golden rules (span every file)

These principles hold across the task list and the Terraform guide — get them right and the rest follows:

- **Foundation before first deploy.** You can't push an image with no ECR repo or route traffic with no ALB.
- **Migrations before code.** Use expand/contract (backward-compatible) so old and new tasks coexist during rollout.
- **Promote one image, don't rebuild.** The artifact built once in CI is the same image promoted staging → prod.
- **Health checks gate every cutover.** Traffic only reaches tasks that passed their check; deploys auto-roll-back on a CloudWatch alarm.
- **Terraform owns infrastructure; the pipeline owns running code.** Use `ignore_changes` on the ECS task definition and Lambda image so the two never fight over drift.
- **Least privilege + no static keys.** Separate roles per component; CI authenticates via GitHub OIDC.
- **Everything through IaC and the pipeline.** Nothing clicked together in the console; a push is enough to deploy.

## Ownership split

- **Solutions Architect (SA)** leads the foundation: accounts, governance, networking, security baseline, data layer, edge, reliability, cost, DR.
- **Developer (Dev)** leads the repeatable pipeline: container build, task definitions, migrations, deploy, verification.
- **Both** own the cross-cutting layer: CI/CD, deployment strategy, observability, and the go-live checklist.
