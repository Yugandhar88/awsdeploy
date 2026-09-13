# AWS Application Deployment — Master Task List

Stack in scope: containerized API on **ECS Fargate**, **Aurora PostgreSQL**, **Lambda** for async work, fronted by **ALB / Route 53 / CloudFront**.

This list is written from two perspectives: the **solutions architect** (structure, security, reliability, cost, DR) and the **AWS developer** (build, deploy, migrate, verify).

## How to use this

- **Phases 0–8** are one-time **foundation** per environment — mostly architect-led. You don't repeat these on every code change.
- **Phases 9–11** are the **repeatable pipeline** — developer-led, runs on every change.
- **Phases 12–16** are **cross-cutting** — set up once, owned by both, revisited continuously.

Owner tags: **(SA)** architect-led · **(Dev)** developer-led · **(Both)** shared.

Golden sequencing rules: foundation before first deploy · migrations before code · same image promoted across environments (never rebuilt) · health checks gate every cutover · everything through IaC and the pipeline, nothing clicked in the console.

---

## Phase 0 — Account & governance foundation (SA)

- [ ] Define multi-account structure via AWS Organizations (separate dev / staging / prod, plus a tooling/shared-services account)
- [ ] Apply guardrails with Control Tower or SCPs (region restrictions, deny root actions, etc.)
- [ ] Decide and document region strategy (single-region vs multi-region) and why
- [ ] Establish naming conventions and a tagging strategy (env, owner, cost-center, app, data-classification)
- [ ] Define availability and recovery targets up front: SLOs, **RPO**, **RTO**
- [ ] Choose IaC tool (CDK / Terraform / CloudFormation) and stick to it
- [ ] Set up IaC state management (Terraform: S3 backend + DynamoDB lock; CDK/CFN: stack + bootstrap strategy)
- [ ] Bootstrap each account (`cdk bootstrap` / Terraform backend init)
- [ ] Create AWS Budgets + cost anomaly detection with alerts

## Phase 1 — Security baseline (Both)

- [ ] Lock down root: MFA on, no access keys, credentials stored securely
- [ ] Enable CloudTrail across all regions → S3, with log-file validation
- [ ] Enable AWS Config, GuardDuty, Security Hub, and Inspector
- [ ] Create per-environment KMS keys for encryption at rest
- [ ] Create least-privilege IAM roles: ECS **task execution role**, ECS **task role**, **Lambda execution role**, **pipeline deploy role** (one per env)
- [ ] Set up GitHub OIDC federation (or CodeConnections) so CI uses short-lived creds — **no long-lived AWS keys in CI**
- [ ] Apply permission boundaries to CI/deploy roles
- [ ] Store all secrets in Secrets Manager / SSM Parameter Store; plan rotation
- [ ] Enforce TLS in transit everywhere; no plaintext endpoints

## Phase 2 — Networking (SA)

- [ ] Design VPC CIDR plan (leave room to grow)
- [ ] Create subnets across **≥2 AZs**: public (ALB, NAT), private-app (ECS, Lambda), private-data (Aurora, RDS Proxy)
- [ ] Internet Gateway + NAT Gateway (one per AZ for HA in prod; single for cost in non-prod)
- [ ] Route tables per subnet tier
- [ ] Security groups chained least-privilege: ALB SG → ECS SG → Aurora SG
- [ ] VPC endpoints for ECR (api + dkr), S3 (gateway), Secrets Manager, CloudWatch Logs, SSM — so private tasks avoid the NAT gateway
- [ ] Enable VPC Flow Logs
- [ ] Use SSM Session Manager for access — no SSH bastion, no key pairs
- [ ] (Optional) NACLs for defense-in-depth

## Phase 3 — Data layer (Both)

- [ ] Provision Aurora PostgreSQL, **Multi-AZ**, in private-data subnets
- [ ] Decide Aurora Serverless v2 vs provisioned instances
- [ ] Configure writer + reader endpoints; add read replicas as needed
- [ ] Set up **RDS Proxy** for connection pooling (critical for Lambda)
- [ ] Tune parameter groups (connection limits, logging, SSL enforcement)
- [ ] Enable encryption at rest (KMS) and enforce SSL connections
- [ ] Enable automated backups + point-in-time recovery; set retention
- [ ] Schedule snapshots; enable cross-region snapshot copy if DR requires it
- [ ] Store DB credentials in Secrets Manager; enable automatic rotation
- [ ] Create a least-privilege migration DB user

## Phase 4 — Container image & registry (Dev)

- [ ] Create ECR repository per service; enable **scan-on-push** and **immutable tags**
- [ ] Write a multi-stage Dockerfile: minimal base image, run as **non-root**, include a container HEALTHCHECK
- [ ] Adopt an image tagging strategy (git SHA — **never `:latest`** in prod)
- [ ] Add an ECR lifecycle policy to expire old images
- [ ] (Optional) Generate an SBOM for supply-chain visibility

## Phase 5 — ECS service (Dev + SA)

- [ ] Create ECS cluster (Fargate); configure capacity providers (Fargate + Fargate Spot for non-prod)
- [ ] Author task definition: image, CPU/memory, port mappings, log config (awslogs/FireLens), env vars, and **secrets pulled from Secrets Manager/SSM at runtime**
- [ ] Separate task execution role (pull image, fetch secrets) from task role (app's AWS permissions)
- [ ] Expose a **health endpoint** (`/health`) in the app and configure the target-group health check to poll it
- [ ] Handle **SIGTERM** for graceful shutdown; set an appropriate `stopTimeout`
- [ ] Configure service: desired count, min/max healthy %, deployment controller (rolling or CodeDeploy blue/green), **deployment circuit breaker with rollback**
- [ ] Register service with the ALB target group
- [ ] Set up service auto-scaling (target tracking on CPU or ALB request count)
- [ ] (Multi-service) Configure Service Connect / service discovery

## Phase 6 — Lambda (Dev)

- [ ] Package functions (zip or container image)
- [ ] Create least-privilege execution role per function
- [ ] Configure VPC access if it must reach Aurora (subnets, SG, ENIs) — prefer going through RDS Proxy
- [ ] Inject env vars + secrets
- [ ] Use **versions + aliases** (e.g. a `prod` alias); add provisioned concurrency if latency-sensitive, reserved concurrency to cap blast radius
- [ ] Wire event source mappings (SQS / EventBridge / S3) and attach **dead-letter queues** for failures
- [ ] Tune timeout and memory; make handlers **idempotent** (events can be retried)

## Phase 7 — Edge, DNS & load balancing (SA)

- [ ] Request and validate an ACM certificate in-region for the ALB (and a separate one in us-east-1 if using CloudFront)
- [ ] Create the ALB in public subnets; HTTPS listener, HTTP→HTTPS redirect, TLS 1.2+ security policy
- [ ] Create target groups with health checks
- [ ] Set up Route 53 hosted zone + alias records to the ALB / CloudFront
- [ ] Put CloudFront in front of the S3 static frontend with sensible caching
- [ ] Attach WAF (managed rule groups + rate limiting) to ALB/CloudFront
- [ ] Confirm Shield Standard coverage; evaluate Shield Advanced if warranted

## Phase 8 — CI/CD pipeline (Dev + SA)

- [ ] Set up repo, branch strategy, and protected branches
- [ ] Connect source via CodeConnections/OIDC
- [ ] **Build stage:** install → lint → unit tests → build image → scan → push to ECR (tag = git SHA)
- [ ] **Test stage:** integration and contract tests
- [ ] Treat the built image as the artifact and **promote the same image** through environments — don't rebuild per env
- [ ] Auto-deploy to staging
- [ ] **Manual approval gate** before production
- [ ] Deploy to production
- [ ] Send notifications (SNS → Slack/email) on start / success / failure
- [ ] Scope pipeline with least-privilege, per-environment deploy roles
- [ ] Run IaC deploy + **drift detection** in the pipeline

## Phase 9 — Database migrations (Dev)

- [ ] Pick a migration tool (Flyway / Liquibase / Alembic / Prisma / etc.)
- [ ] Run migrations as a dedicated pipeline step or a one-off ECS task with DB access
- [ ] Use the **expand/contract pattern** so old and new code can coexist during rollout
- [ ] Always run migrations **before** the app deploy
- [ ] Never ship a destructive migration in the same release as code that still needs the old schema
- [ ] Test every migration on staging first; keep a rollback/down path

## Phase 10 — Deployment strategy & rollback (SA + Dev)

- [ ] Choose the strategy: rolling vs blue/green (CodeDeploy) vs canary
- [ ] For blue/green: configure test listener, validation hooks, and **auto-rollback on CloudWatch alarm**
- [ ] Enable the ECS deployment circuit breaker
- [ ] For Lambda: canary/linear traffic shifting via CodeDeploy + aliases
- [ ] Keep the previous task-def revision / image tag ready for fast rollback
- [ ] (Optional) Use feature flags to **decouple deploy from release**

## Phase 11 — Verification (Dev)

- [ ] Run post-deploy **smoke tests** against the live environment
- [ ] Set up CloudWatch Synthetics canaries for critical user flows
- [ ] Confirm health checks green and error rates normal before marking the deploy successful

## Phase 12 — Observability (Both)

- [ ] Emit **structured (JSON) logs**; ship to CloudWatch Logs; set retention (don't keep forever)
- [ ] Enable Container Insights and Lambda Insights
- [ ] Build CloudWatch dashboards for key metrics
- [ ] Enable **X-Ray** distributed tracing across ECS and Lambda
- [ ] Define SLOs; create alarms on key SLIs (5xx rate, p99 latency, CPU, DB connections, queue depth) → SNS → on-call
- [ ] (Optional) CloudWatch RUM for frontend real-user monitoring

## Phase 13 — Reliability & scaling (SA)

- [ ] Configure ECS service auto-scaling policies
- [ ] Configure Aurora read-replica auto-scaling / Serverless v2 scaling
- [ ] Confirm Multi-AZ everywhere and **test failover** deliberately
- [ ] Implement retries with backoff, circuit breakers, and timeouts on external calls
- [ ] **Load test** before production to find the breaking point

## Phase 14 — Cost optimization (SA)

- [ ] Activate cost allocation tags
- [ ] Keep AWS Budgets + anomaly detection alerts live
- [ ] Use Fargate Spot for non-prod / batch workloads
- [ ] Right-size ECS tasks and Lambda memory
- [ ] Buy Savings Plans / Compute Savings Plans for steady baseline load
- [ ] Set S3 lifecycle rules and cap log retention
- [ ] Review NAT gateway spend (VPC endpoints cut it)
- [ ] Shut down non-prod environments off-hours

## Phase 15 — DR & backup (SA)

- [ ] Document RPO/RTO and the recovery runbook
- [ ] Confirm Aurora automated backups + PITR are on
- [ ] Set up cross-region snapshot copy if the DR plan requires it
- [ ] Protect other stateful stores (S3 versioning / cross-region replication)
- [ ] Run **restore drills** on a schedule — an untested backup isn't a backup
- [ ] Verify infra is reproducible from IaC in another region/account

## Phase 16 — Go-live checklist (Both)

- [ ] All alarms configured and routed to on-call
- [ ] Runbooks written for common incidents
- [ ] Rollback tested end-to-end (not just assumed)
- [ ] Load test passed at expected peak + headroom
- [ ] Security review / WAF tuning / (if applicable) pen test done
- [ ] DNS TTL lowered ahead of cutover
- [ ] Backups verified and a restore tested
- [ ] Cost budget alarms active
- [ ] Well-Architected review completed
- [ ] Stakeholder sign-off
