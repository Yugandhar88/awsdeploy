# Prompt — AWS deployment architecture & plan (ECS + Aurora + Lambda)

Paste the whole of this into a new session to regenerate the architecture, diagrams, deployment steps, task list, and Terraform guide.

---

## Role & lens

Act as **both an AWS Solutions Architect and an AWS Developer**. Bring both lenses to every answer: the architect view (structure, security, reliability, cost, DR, Well-Architected) and the developer view (build, deploy, migrate, verify, concrete mechanics). Cross-check your own output for missing pieces before finalizing.

## System in scope

A customer-facing web application on AWS with this stack:

- **Compute:** containerized API on **ECS Fargate**; **Lambda** for event-driven / async work
- **Data:** **Aurora PostgreSQL** (writer + reader), with **RDS Proxy** for connection pooling
- **Edge / routing:** **Route 53** (DNS) → **ALB** (Layer 7; NLB only where a static IP or L4 is needed) → target groups; **CloudFront** + **S3** for the static frontend
- **Networking:** VPC with public + private-app + private-data subnets across ≥2 AZs
- **External dependencies:** a payment gateway and an email provider

## What to produce

1. **Architecture overview** — explain how code gets deployed on AWS and how traffic flows end to end (User → Route 53 → ALB/NLB → target group → ECS/Lambda → Aurora). Clarify where Route 53, ALB, and NLB each fit, and where ECS, Aurora, and Lambda sit (public vs private subnets, security-group chaining ALB → ECS → Aurora).

2. **Full AWS component list**, grouped by layer: edge (Route 53, CloudFront, WAF, Shield, ACM, API Gateway), networking (VPC, subnets, IGW, NAT, VPC endpoints, security groups, flow logs, multi-AZ), load balancing, compute (ECS, ECR, Lambda, auto-scaling), data & caching (Aurora, RDS Proxy, ElastiCache, S3), messaging/events (SQS, SNS, EventBridge), security & secrets (IAM, Secrets Manager, KMS, Cognito), observability (CloudWatch, X-Ray, CloudTrail), and CI/CD.

3. **C4 model diagrams**, one per level, mapped onto this stack, each as a diagram with prose between:
   - **C1 System Context** — the app among its users (customer) and external systems (payment gateway, email provider)
   - **C2 Container** — web frontend (S3/CloudFront), API service (ECS Fargate), async worker (Lambda), database (Aurora)
   - **C3 Component** — inside the API service: controllers, auth, orders service, products service, data-access layer
   - **C4 Code** — classes inside one component (e.g. Orders: controller, service, repository interface + impl, entity, payment client) as a class diagram

4. **Deployment steps**, split into one-time **foundation** vs the repeatable **per-deploy pipeline**. Make the sequencing rules explicit: foundation before first deploy; **migrations before code** (expand/contract, backward-compatible); same image promoted across environments (never rebuilt); health checks gate every cutover; everything through IaC and the pipeline.

5. **A detailed, checkbox task list** covering the full lifecycle in ~16–17 phases: account & governance, security baseline, networking, data, container & registry, ECS service, Lambda, edge/DNS, CI/CD, DB migrations, deployment strategy & rollback, verification, observability, reliability & scaling, cost optimization, DR & backup, and a go-live checklist. Tag each task by owner: **(SA)** architect-led, **(Dev)** developer-led, **(Both)**. Deliver it as a Markdown file.

6. **A Terraform-specific implementation guide** as a separate Markdown file, covering:
   - The **infra-vs-pipeline boundary**: Terraform owns durable infrastructure; the CD pipeline owns running code. Use `ignore_changes = [task_definition, desired_count]` on the ECS service and `ignore_changes` on the Lambda image so the pipeline can deploy without Terraform reverting it.
   - Repo & module structure (`bootstrap/`, `modules/`, `envs/<env>/`)
   - Remote **S3 state** with locking, the bootstrap chicken-and-egg, separate state per environment
   - Environment strategy (directory-per-env vs workspaces; recommend directory-per-env for multi-account/prod)
   - Provider/version pinning and `default_tags`
   - Mapping each deployment phase to a Terraform module
   - The Terraform CI/CD workflow (fmt → validate → tflint/tfsec → plan on PR → apply per env with a manual approval gate before prod; OIDC roles, no static keys)
   - Gotchas: secrets in state, `prevent_destroy` on stateful resources, avoiding `-target`, drift detection

## Constraints & decisions to honor

- Least-privilege IAM throughout; separate ECS task execution role vs task role vs Lambda role vs pipeline deploy role
- **GitHub OIDC federation** for CI — no long-lived AWS keys
- Multi-account structure (dev / staging / prod + tooling) via AWS Organizations
- Encryption at rest (KMS) and in transit (TLS) everywhere; secrets in Secrets Manager, consumed by task definitions at runtime
- Health endpoint (`/health`) + graceful **SIGTERM** shutdown; blue/green or rolling deploys with **auto-rollback on CloudWatch alarm** and a deployment circuit breaker
- Image tags = git SHA (never `:latest`); ECR scan-on-push
- Defined **SLOs, RPO, RTO**; automated backups + PITR; scheduled **restore drills**; load test before go-live
- Immutable image promoted unchanged from staging to prod

## Output preferences

- Concise, lead with the answer, minimal preamble
- Use inline diagrams for architecture/C4 content
- Deliver the task list and the Terraform guide as downloadable Markdown files
- Call out any piece I'm missing rather than only answering what's asked
