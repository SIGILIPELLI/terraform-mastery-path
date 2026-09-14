# 10 · Capstone Project

This capstone assembles Level 4's modules into one coherent deliverable:
a platform-shaped Terraform layout for a small application, with CI/CD,
drift detection, cost estimation, security scanning, and governance all
wired into the same pipeline, plus a written disaster-recovery and
observability plan. It does not introduce new mechanisms — every piece
below is a direct application of modules 01–09.

## The scenario

A team is standing up a new service — an internal API backed by a
managed database — and needs it to meet the same bar as every other
production service on the platform: reviewed module sources, policy
enforcement, cost visibility, security scanning, and a documented
recovery path, without the team having to reinvent any of it.

## Step 1 — Layered state, per module 06

```text
foundation/        (existing — VPC, subnets, IAM boundaries; not touched)
teams/
  internal-api/     (new — this service's own state)
```

```hcl
# teams/internal-api/main.tf
terraform {
  backend "s3" {
    bucket         = "acme-tfstate"
    key            = "teams/internal-api/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "acme-tf-locks"
  }
}

data "terraform_remote_state" "foundation" {
  backend = "s3"
  config  = { bucket = "acme-tfstate", key = "foundation/terraform.tfstate", region = "us-east-1" }
}

module "api_service" {
  source  = "app.terraform.io/acme/ecs-service/aws"
  version = "~> 2.1"

  vpc_id          = data.terraform_remote_state.foundation.outputs.vpc_id
  private_subnets = data.terraform_remote_state.foundation.outputs.private_subnet_ids
  service_name    = "internal-api"
  container_image = "acme/internal-api:1.0.0"
}

module "database" {
  source  = "app.terraform.io/acme/rds-postgres/aws"
  version = "~> 3.2"

  vpc_id            = data.terraform_remote_state.foundation.outputs.vpc_id
  subnet_ids        = data.terraform_remote_state.foundation.outputs.private_subnet_ids
  instance_class    = "db.t3.medium"
  allocated_storage = 50
}
```

New state, own lock table entry, reads `foundation`'s published outputs
only — no other team's state or foundation's own configuration is
touched. Module sources are pinned to specific versions from the org's
private registry, satisfying the module-provenance governance policy
from module 07 by construction.

## Step 2 — CI/CD pipeline chaining every check before apply

```yaml
# .github/workflows/internal-api-terraform.yml
on:
  pull_request:
    paths: ["teams/internal-api/**"]
jobs:
  plan-and-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: terraform init
      - run: terraform plan -out=tfplan
      - run: terraform show -json tfplan > plan.json

      # module 04 — cost estimation
      - run: infracost breakdown --path . --format json --out-file infracost.json
      - uses: infracost/actions/comment@v3
        with: { path: infracost.json, behavior: update }

      # module 05 — security scanning
      - run: conftest test --policy ../../security-policies/ plan.json

      # module 07 — governance / policy-as-code, enforced via Terraform Cloud
      # (policy set "production-governance" attached to this workspace)

      - uses: actions/upload-artifact@v4
        with: { name: plan-json, path: plan.json }

  apply:
    needs: plan-and-check
    if: github.ref == 'refs/heads/main'
    environment: production   # requires manual approval
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: terraform init
      - run: terraform apply -auto-approve tfplan
```

This is Level 3's CI/CD module (02) and testing module (04) combined
with Level 4's cost and security modules into one ordered gate: plan →
cost estimate posted for human review → security policy check → org-wide
governance policy (evaluated by Terraform Cloud on the same plan) →
manual approval → apply. Any failing static check blocks before a human
approver ever sees the pull request, and the approval gate itself is a
human decision informed by the cost delta and scan results already
posted as PR comments.

## Step 3 — Drift detection, per module 03

```yaml
# .github/workflows/internal-api-drift.yml
on:
  schedule:
    - cron: "0 */6 * * *"
jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - run: terraform init
      - run: terraform plan -detailed-exitcode -out=tfplan; echo "code=$?" >> "$GITHUB_OUTPUT"
        id: plan
      - if: steps.plan.outputs.code == '2'
        run: |
          terraform show -json tfplan > drift.json
          aws s3 cp drift.json s3://acme-tf-audit/teams/internal-api/drift/drift-$(date +%s).json
          curl -X POST "$SLACK_WEBHOOK" -d '{"text":"Drift detected in internal-api"}'
```

Scheduled every six hours, feeding into the same audit-storage prefix
convention as regular applies — matching the observability pipeline from
module 09, so drift and deliberate changes both land in one queryable
timeline for this service's state.

## Step 4 — Observability, per module 09

```bash
export TF_LOG=INFO
export TF_LOG_JSON=1
terraform apply -auto-approve tfplan 2>&1 | tee apply.jsonl
aws s3 cp apply.jsonl s3://acme-tf-audit/teams/internal-api/applies/apply-$(date +%s).jsonl
aws s3 cp plan.json   s3://acme-tf-audit/teams/internal-api/plans/plan-$(date +%s).json
```

Every apply's structured log and the plan JSON that preceded it are
archived under the same team-scoped audit prefix, alongside Terraform
Cloud's own run metadata (requester, approver, plan summary) pulled via
its Audit Trails API. Together these answer, for any change to this
service: what was proposed, who approved it, what actually happened, and
whether the same or a related resource later drifted.

## Step 5 — Disaster recovery plan, per module 08

```text
1. State: S3 backend with versioning + cross-region replication to
   us-west-2 (inherited from the shared tfstate bucket's existing config).
2. Recovery path: `aws s3api list-object-versions` → identify last known
   good version → `terraform state push` → `terraform plan` (expect zero
   diff) before trusting the recovered state.
3. No state backup scenario: `terraform import` each of the ~6 resources
   in this service's state (ECS service, task definition, RDS instance,
   security groups) — small enough to be fully reconstructable manually
   within an incident, unlike a large monolithic state would be.
4. This service currently has no warm-standby DR region — documented as
   an accepted risk given its internal, non-customer-facing nature; the
   database's automated snapshots (RDS default) are the only cross-region
   recovery path for data, not for the compute layer.
```

Written explicitly rather than assumed — including the honest limitation
in point 4 — because a DR plan that only covers state recovery and
ignores whether the *application* has a failover path is incomplete, and
capstone-quality work says so rather than glossing over it.

## How It Actually Works

**Nothing in this capstone introduces a new mechanism — its only genuine
content is the ordering and composition of nine independently correct
mechanisms into one pipeline where each stage's output becomes the next
stage's input: the same `terraform show -json` plan artifact is consumed,
unmodified, by the cost estimator, the security scanner, and the
governance policy evaluator, in sequence, before any apply is attempted.**
This is the payoff of every module's "How It Actually Works" section
emphasizing the JSON plan as a stable, documented interface rather than
an internal implementation detail — a platform team can add or remove
any one of these checks independently, because each one is a separate
consumer of the same artifact rather than a step baked into another
tool's internals.

**The layered-state boundary (module 06) is what makes every other piece
of this pipeline safe to run per-team rather than org-wide: because
`internal-api`'s state, lock, plan, and CI run are all scoped to this one
team, a failing security scan or a stuck lock on this pipeline has zero
mechanical effect on any other team's state, lock, or pipeline** — the
blast-radius argument from module 06 is the reason this entire capstone
can be reasoned about as "one team's pipeline" at all, rather than as one
component of a single shared apply that every other team's changes would
also have to pass through.

**The governance policy set (module 07) and the security-scanning
Conftest check (module 05) are deliberately two separate enforcement
points evaluating the same plan.json for different reasons — Conftest
runs inside this team's own CI job using policies the team could, in
principle, see and modify, while the Terraform Cloud policy set is
attached externally by the platform team and cannot be bypassed by
editing this repository** — the redundancy is intentional: a team-owned
CI check is fast feedback during development, while the org-owned policy
set is the actual, non-bypassable governance boundary, and this
capstone's pipeline runs both because neither one alone would provide
both properties.

## Exercise

Extend this capstone's pipeline with one more gate: a step that fails
the pull request if the Infracost-reported monthly cost delta exceeds
$500, *unless* the pull request has a label named `cost-approved`. Using
what you know from modules 04 and 07 about where cost data and policy
enforcement each live, decide whether this new gate belongs in the CI
job's own script (like the Conftest step) or in the Terraform Cloud
policy set, and justify the choice in terms of who should be able to
bypass it and how.
