# Safe & Low-Cost AWS Deployment Prompt

Use this prompt with an AI agent that has AWS CLI / Terraform access (e.g. Claude Code)
whenever you need to deploy something new into an AWS account that already has
workloads running — without disrupting them, and while keeping cost as low as possible.

---

## The Prompt

```
You are helping me deploy a new application into an existing AWS account
that already has other workloads running. Your top priorities, in order:
1. Never modify, delete, or disrupt any existing resource.
2. Keep cost as low as reasonably possible for a dev environment.
3. Make everything traceable and reversible.

Follow this process exactly, in order. Do not create or modify any
resource until Phase 1, 2, and 3 are all complete and I have explicitly
approved the plan in Phase 3.

═══════════════════════════════════════════
PHASE 1 — DISCOVERY (strictly read-only)
═══════════════════════════════════════════

1. Identity, account, and organizational context
   - aws sts get-caller-identity
   - aws configure get region
   - Confirm this is actually the intended dev account, not prod.
   - aws organizations describe-organization (may fail if not in an Org — that's fine,
     note it). If it succeeds, check for Service Control Policies that might block
     resource creation: aws organizations list-policies --filter SERVICE_CONTROL_POLICY

2. My own permissions (fail fast, don't discover mid-build)
   - aws iam get-user (or get-role if using a role)
   - aws iam list-attached-user-policies / list-attached-role-policies
   - Try a dry run: aws ec2 create-vpc --cidr-block 10.255.255.0/24 --dry-run
     (expects a DryRunOperation success/failure, not an actual VPC)
   - Flag now if permissions look insufficient for VPC, RDS, ECS, IAM role creation —
     better to find out before Phase 4 than during it.

3. Existing network footprint
   - aws ec2 describe-vpcs --query 'Vpcs[*].{ID:VpcId,CIDR:CidrBlock,Default:IsDefault,Tags:Tags}'
   - aws ec2 describe-vpcs --query 'Vpcs[*].CidrBlockAssociationSet' (secondary CIDRs)
   - aws ec2 describe-subnets --query 'Subnets[*].{VPC:VpcId,CIDR:CidrBlock,AZ:AvailabilityZone}'
   - aws ec2 describe-vpc-peering-connections
   - aws ec2 describe-transit-gateway-attachments (if used)
   - List every CIDR in use so a new VPC CIDR can be chosen with zero overlap,
     even if peering isn't planned today (future-proofing is free).

4. Quotas that could block creation or affect shared limits
   - aws ec2 describe-vpcs --query 'length(Vpcs)' (default quota: 5/region)
   - aws service-quotas get-service-quota for: VPCs per region, Elastic IPs per
     region, NAT Gateways per AZ, ENIs per region (matters if using Lambda-in-VPC,
     since it shares the account's ENI budget with everything else)
   - aws rds describe-account-attributes (RDS instance quota)

5. Naming and identifier collisions
   - aws s3 ls
   - aws iam list-roles --query 'Roles[*].RoleName'
   - aws rds describe-db-instances --query 'DBInstances[*].DBInstanceIdentifier'
   - aws ecs list-clusters
   - aws ecr describe-repositories (if using your own image repo)
   Flag any name I plan to use that already exists or is close enough to confuse.

6. Existing IaC / state — do not touch or reuse
   - Look for a Terraform backend already in use (S3 bucket + DynamoDB lock table)
     for the existing app. Never point the new project at the same state key.
   - If existing infra is CloudFormation, list stack names
     (aws cloudformation list-stacks) so nothing overlaps or gets nested by accident.

7. Shared resources that require extra care (add-only, never modify)
   - Route 53: aws route53 list-hosted-zones — if shared, I can only ADD a record.
   - ACM certificates: aws acm list-certificates — note the region; ALB certs must
     be in the same region as the ALB, CloudFront certs must be in us-east-1
     regardless of where the app runs.
   - KMS keys: aws kms list-aliases — check for a shared account-level key I might
     accidentally reuse instead of creating my own scoped one.
   - Secrets Manager / SSM Parameter Store: check for existing paths/prefixes so
     new secrets use a distinct naming prefix.
   - SNS topics / EventBridge rules that might be account-wide (e.g. security alerts)
     — don't attach new resources to these without knowing what else listens.

8. Observability already in place
   - aws cloudtrail describe-trails — is CloudTrail already enabled? Don't create
     a duplicate trail; do confirm it will capture the new resources' events.
   - aws cloudwatch describe-alarms — note existing alarm naming/tagging conventions
     to follow for consistency.
   - Check if AWS Config or Security Hub is already enabled (affects whether new
     resources get auto-evaluated against existing rules).

9. Cost baseline (so the new app's cost is measurable, not guessed)
   - Note current month-to-date spend via Cost Explorer or:
     aws ce get-cost-and-usage --time-period Start=<first-of-month>,End=<today> \
       --granularity MONTHLY --metrics "UnblendedCost"
   - Check if AWS Budgets already has an alert configured for this account.
   - This baseline lets us later prove exactly what the new app added to the bill.

Summarize all of Phase 1 in a short written report before proceeding —
include a plain list of every CIDR, name, and shared resource found, and
explicitly flag anything that looked risky or ambiguous.

═══════════════════════════════════════════
PHASE 2 — SELF-ANSWER FIRST, THEN ASK
═══════════════════════════════════════════

Before asking me anything, try to answer it yourself using a read-only
role assumption — but only for facts that genuinely live in AWS. Never
use this step to infer or guess at judgment calls; those always get
asked, never assumed.

2a. Assume a view-only role, strictly scoped
   - Use a role/policy equivalent to AWS managed "ViewOnlyAccess" or
     "ReadOnlyAccess" — never a role with any write/create/delete permission.
   - This role must be DIFFERENT from whatever role/credentials will run
     terraform apply in Phase 4. Read-only discovery and the ability to
     change the account should never be the same session.
   - Session should be time-boxed (e.g. aws sts assume-role with a short
     --duration-seconds) rather than long-lived credentials.
   - If no such role exists yet, ask me to create one before proceeding —
     do not fall back to using write-capable credentials "just to look."

2b. Self-answer discoverable facts using that role
   Examples of what belongs here (pull the answer yourself, don't ask me):
   - "Is there an existing naming/tagging convention?" → inspect tags on
     existing resources (aws resourcegroupstaggingapi get-resources)
   - "Is there a shared Route 53 zone or ACM cert?" → already covered in
     Phase 1, don't re-ask
   - "Is CI/CD already in place?" → check for existing CodePipeline/
     CodeBuild projects, or a .github/workflows directory in the repo
   - "Is there a cost anomaly detector or budget already configured?" →
     aws budgets describe-budgets, aws ce get-anomaly-monitors
   Report what you found and how you found it, so I can correct you if
   the inference is wrong.

2c. Only ask me directly for genuine judgment calls — never infer these
   from account data, even if a plausible-looking answer exists:
   - Does the new app need any connectivity to the existing app or its data?
   - Is there a hard budget ceiling for this dev app, monthly or total?
   - Who else has access to this account and should be notified before deploying?
   - Does this app need to run 24/7, or can it scale down outside work hours?
   - Any compliance/data-residency constraints?
   For each, tell me where I could find the answer myself if I don't know
   it offhand (e.g. "ask your account admin," "check the team's cost doc").

If discovery in 2a/2b is ambiguous or contradictory, say so and ask —
do not silently pick the more convenient interpretation.

═══════════════════════════════════════════
PHASE 3 — PROPOSE BEFORE BUILDING
═══════════════════════════════════════════

Once questions are answered, propose, in writing:
   1. The new VPC CIDR and why it doesn't collide with anything found in Phase 1
   2. The complete resource list to be created, each with its Project/Environment tags
   3. Which existing resources (if any) will be touched — ideally zero, or at most
      one new DNS record in a shared hosted zone
   4. Every place cost is being minimized, explicitly, for example:
        - Fargate Spot vs on-demand for non-critical dev tasks
        - Single NAT Gateway (or no NAT, using VPC Gateway/Interface Endpoints instead)
        - Single-AZ RDS instead of Multi-AZ for dev
        - Smallest viable instance sizes (t4g.micro / db.t4g.micro class)
        - Auto-scaling floor of 1, not a fixed higher count
        - Scheduled scale-to-zero outside working hours, if approved in Phase 2
        - Gateway Endpoints (free) preferred over Interface Endpoints (hourly cost)
          wherever the target service supports it (S3, DynamoDB)
   5. The Terraform project structure and state backend location — confirm it is
      fully separate from the existing app's backend/state key
   6. An estimated monthly cost breakdown per resource (NAT, RDS, ECS/Fargate,
      ALB, data transfer) so I can approve or ask you to cut something
   7. A rollback / teardown plan (terraform destroy scoped to this state only)
   8. Confirmation of the "cheap non-negotiables" below — these are low/no-cost
      and worth doing even in dev, so include them in the plan by default:
        - RDS deletion protection: enabled (prevents an accidental destroy/typo
          from wiping the database with no confirmation)
        - RDS automated backup retention: 1 day minimum (still free-tier friendly)
        - VPC Flow Logs: enabled on the new VPC (pennies at dev traffic volume,
          the only real way to debug "why can't X reach Y" after the fact)
        - ALB health check tuned deliberately (path, interval, healthy/unhealthy
          thresholds) rather than left at defaults — affects how fast a bad
          deploy gets pulled from rotation
        - ECS rollback command documented: know how to run
          aws ecs update-service --task-definition <previous-revision>
          before you need it, not after a bad deploy
        - Terraform S3 backend: versioning enabled + DynamoDB lock table —
          protects against a crashed apply corrupting state (near-zero cost)
        - Secret scanning on the app repo (e.g. gitleaks pre-commit hook, or
          GitHub secret scanning if using GitHub): prevents an AWS key or DB
          password from ever landing in git history — cheap/free, and this is
          how real breaches actually happen, more often than misconfigured
          infra
        - Immutable container image tags (git SHA or version number, never
          `:latest`): without this, the ECS rollback command above has
          nothing concrete to roll back TO, and a bad push overwrites the
          only known-good image silently
        - Scheduled auto-shutdown for dev compute/DB outside working hours
          (EventBridge rule + small Lambda to stop ECS tasks and, if idle,
          the RDS instance): this is the single highest-leverage cost lever
          available, often cutting dev spend 60-70% — worth building even
          if approved cost already looks acceptable without it

Wait for my explicit approval before creating anything.

═══════════════════════════════════════════
PHASE 3.5 — APPLICATION RELEASE PIPELINE
═══════════════════════════════════════════

This phase turns your local code into a running, updatable service. Do this
BEFORE Phase 4 builds the ECS service, since the task definition needs a
real image URI to point at — but plan it now so Phase 4 isn't blocked later.

1. Inspect the local repo first (read-only, in my cloned copy)
   - Does a Dockerfile already exist? If not, help me write one — multi-stage
     build, non-root user, only copy what's needed (use a .dockerignore).
   - Is there a migration tool already in use (Alembic, Flyway, Prisma,
     node-pg-migrate, etc.)? Identify it — don't assume raw SQL is the plan.
   - Are there existing tests I can run locally before this goes anywhere
     near AWS? (Catching a broken app before deploy is far cheaper than
     debugging it inside ECS.)
   - Is there already a CI config (.github/workflows, .gitlab-ci.yml,
     buildspec.yml)? If yes, adapt it rather than replacing it silently.

2. Container registry
   - Create a NEW ECR repository, scoped/named to this app only
     (e.g. newapp-web) — never reuse or push into the existing app's
     ECR repo, even if names look similar.
   - Enable scan-on-push (free tier includes basic scanning) so known
     CVEs in the base image surface before deploy, not after.
   - Tag policy: immutable tags only — use the git commit SHA
     (or a semver tag from a release process), never `:latest`, so the
     ECS rollback command from Phase 3 has something concrete to target.

3. Build and push (first manual run, before automating it)
   - docker build -t <ecr-repo-uri>:<git-sha> .
   - aws ecr get-login-password | docker login ...
   - docker push <ecr-repo-uri>:<git-sha>
   - Confirm the image is visible in ECR and the scan came back clean
     (or note any findings) before referencing it in a task definition.

4. Database migrations — decide the mechanism explicitly, don't skip this
   - Migrations should run as a separate step BEFORE the new app version
     receives traffic, not embedded silently in app startup (a crash-looping
     container running migrations repeatedly can corrupt state).
   - Common safe pattern: an ECS "one-off task" (run-task, not a service)
     that runs the migration tool against the new RDS instance, must exit
     0 before the deploy proceeds.
   - Confirm migrations target the NEW database only — this is a fresh,
     separate Postgres instance, so there is no existing schema/data risk,
     but get the connection string right so it can't accidentally point
     at the existing app's DB by a copy-paste mistake in env vars.

5. ECS service deployment settings (safety net for every future release,
   not just this first one)
   - Deployment type: rolling update with:
       minimum healthy percent: 100
       maximum percent: 200
     (so old tasks aren't killed until new ones pass health checks)
   - Enable the ECS deployment circuit breaker with rollback: if new tasks
     fail health checks repeatedly, ECS automatically reverts to the last
     working task definition instead of getting stuck failing forever.
   - ALB health check path should hit a real endpoint (e.g. /health) that
     verifies the app can reach its database, not just that the process is up.

6. CI/CD automation (so step 3 stops being manual)
   - Build a minimal pipeline (GitHub Actions, CodePipeline, or whatever
     matches what step 1 found): on push to main →
       run tests → build image → push to ECR (tagged with git SHA) →
       run migration task → update ECS service to new task definition
   - Scope the pipeline's AWS credentials narrowly: it should only be able
     to push to THIS app's ECR repo and update THIS app's ECS service —
     never given account-wide deploy permissions.
   - Keep this pipeline entirely separate from any pipeline the existing
     app already uses — new pipeline, new credentials, new IAM role.

7. Release verification (run this on every deploy, not just the first)
   - Confirm new tasks reach RUNNING and pass ALB health checks
   - Hit a couple of real endpoints post-deploy (smoke test), not just /health
   - Confirm CloudWatch Logs show the new revision's log stream, no
     repeating error patterns in the first few minutes
   - Confirm the previous task definition revision is still registered
     (so rollback in Phase 3's non-negotiables actually has a target)



   - terraform init with the new, separate backend
   - terraform plan -out=tfplan — show me the FULL plan output
   - Confirm the plan shows zero changes/destroys to any resource outside the
     new project's tags/names. Stop and flag immediately if anything existing
     appears in the plan.
   - Only after my confirmation: terraform apply tfplan
   - Tag every resource consistently (Project, Environment, ManagedBy=terraform,
     Owner) so cost and cleanup can be filtered by tag later

═══════════════════════════════════════════
PHASE 5 — VERIFY
═══════════════════════════════════════════

   - Confirm ALB target group health checks are passing
   - Hit the app end-to-end (public URL → app → database round trip) before
     pointing any real/shared DNS at it
   - Confirm application logs are flowing to CloudWatch Logs (or wherever intended)
   - Confirm at least one basic alarm exists (e.g. ECS service unhealthy, RDS CPU,
     ALB 5xx rate) so failures are noticed, not discovered by users
   - Re-run the Phase 1 discovery queries for VPCs/subnets/naming and diff against
     the original Phase 1 report — confirm nothing unexpected changed elsewhere
     in the account
   - Pull current cost-to-date again and compare against the Phase 1 baseline to
     confirm actual spend roughly matches the Phase 3 estimate

═══════════════════════════════════════════
PHASE 6 — DOCUMENT (for future-you and teammates)
═══════════════════════════════════════════

Write a short record, saved in the repo or a shared doc, containing:
   - The new VPC CIDR and why it was chosen
   - All resource names/tags created
   - Any shared resources touched (e.g. the one new Route 53 record)
   - The estimated vs actual monthly cost
   - The teardown command
   - Anything unusual found in Phase 1 that future deployments should know about
     (e.g. "account is in an AWS Organization with SCP X restricting Y")

Never perform an action that modifies or deletes an existing resource
without calling it out explicitly and getting my confirmation first, at
any phase, even if a later phase seems to require it.
```

---

## "Before this goes to prod" checklist — not needed for dev

These are real hardening steps, but they add cost, complexity, or operational
overhead that isn't justified for a dev environment. Do NOT build these into
the dev deployment by default. Keep this list attached to the project so
whoever promotes this app to staging/prod later knows what was deliberately
skipped:

   - **WAF on the ALB** — cost + complexity for protecting an app with no
     real external traffic yet
   - **Secrets rotation** (Secrets Manager auto-rotation) — unnecessary
     overhead for dev credentials
   - **GuardDuty / Security Hub** — only rely on these if already enabled
     account-wide for free; don't enable per-app in dev
   - **IAM permission boundaries** — an org-wide governance control,
     not a per-app dev concern
   - **Container image scanning on push** — nice to have, not decision-critical
     pre-prod
   - **Multi-AZ RDS** — dev can run single-AZ; only pay for the failover
     capability once it's actually being tested or is customer-facing
   - **CloudFront / edge caching** — adds cost and cache-invalidation
     complexity not needed until real user traffic exists

Add one line to the Phase 6 documentation explicitly stating this dev setup
skips the above, so nobody assumes this Terraform is prod-ready as-is.

---

## Why each phase exists (quick reference)

| Phase | Protects against |
|---|---|
| 1. Discovery | Building on wrong assumptions, hitting quotas, name collisions, missing SCP blocks |
| 2. Self-answer + ask | Guessing on judgment calls only you or your team can answer, while not
   pestering you for facts a read-only role can find on its own |
| 3. Propose | Committing cost or risk before you've seen the full picture |
| 3.5. Release pipeline | App can't actually be deployed/updated safely, migrations
   corrupting state, no rollback target, manual error-prone releases |
| 4. Build | Terraform accidentally touching/destroying existing resources |
| 5. Verify | Silent failures, cost surprises, undetected side effects elsewhere in the account |
| 6. Document | Repeating the whole discovery process next time, or teammates deploying blind |

## How to use this

1. Paste the prompt into your AI agent session (with AWS CLI/Terraform tool access).
2. Let it complete Phase 1 fully — read the discovery report yourself too, don't just skim.
3. In Phase 2, review what it self-answered via the read-only role and correct anything
   it inferred wrong; answer the judgment-call questions directly (use the
   "where to find it" pointers if you're unsure).
4. Review the Phase 3 cost breakdown and resource list carefully before approving.
5. Watch the `terraform plan` output in Phase 4 for anything unexpected before approving apply.
6. Don't skip Phase 5 and 6 — they're what makes the *next* deployment easier and safer.
