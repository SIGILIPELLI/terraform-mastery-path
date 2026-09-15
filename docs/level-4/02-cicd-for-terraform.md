---
description: "CI/CD for Terraform — Level 3 module 01 recommended routing applies through CI rather than laptops. This module builds that pipeline concretely with…"
---

# 02 · CI/CD for Terraform

Level 3 module 01 recommended routing applies through CI rather than
laptops. This module builds that pipeline concretely with GitHub Actions —
plan-on-PR, apply-on-merge, using the `-detailed-exitcode` and
`-out=tfplan` patterns from Level 1 module 09 and the policy check from
Level 3 module 05.

## The two-workflow pattern: plan on PR, apply on merge

```yaml
# .github/workflows/terraform-plan.yml
name: Terraform Plan
on:
  pull_request:
    paths: ["envs/prod/**", "modules/**"]

jobs:
  plan:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: envs/prod
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.5"
      - run: terraform init
      - run: terraform fmt -check
      - run: terraform validate
      - run: terraform plan -no-color -out=tfplan
      - run: terraform show -no-color tfplan > plan.txt
      - uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('envs/prod/plan.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: "```\n" + plan.slice(0, 60000) + "\n```",
            });
```

Every pull request touching `envs/prod` or `modules/` gets an automatic
comment showing exactly what `terraform plan` would do — this is Level 3
module 04's "attach the plan to the PR for review" practice, automated so
it can never be forgotten or run against a stale checkout.

```yaml
# .github/workflows/terraform-apply.yml
name: Terraform Apply
on:
  push:
    branches: [main]
    paths: ["envs/prod/**", "modules/**"]

jobs:
  apply:
    runs-on: ubuntu-latest
    environment: production   # requires manual approval via GitHub Environments
    defaults:
      run:
        working-directory: envs/prod
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.5"
      - run: terraform init
      - run: terraform apply -auto-approve
```

`environment: production` ties this job to a GitHub Environment configured
with required reviewers — the job pauses, waiting for a human approval
click, before `terraform apply -auto-approve` actually runs. This is the
"CI pipeline with its own review gate" case Level 1 module 09 flagged as
the legitimate use of `-auto-approve`: the approval gate replaces the
interactive terminal prompt, rather than removing review entirely.

## Injecting the policy check (Level 3 module 05) into the pipeline

```yaml
      - run: terraform show -json tfplan > plan.json
      - run: |
          curl -sL https://github.com/open-policy-agent/conftest/releases/download/v0.53.0/conftest_0.53.0_Linux_x86_64.tar.gz | tar xz
          ./conftest test plan.json --policy policy/
```

Adding two steps between `plan` and `apply` runs the OPA/Rego policies
from Level 3 module 05 as a real CI gate — a failing `conftest test`
fails the workflow step, which (by GitHub Actions' default job-failure
behavior) prevents the workflow from proceeding to any later apply step
in the same job, with zero code needed to wire the failure into the
pipeline's control flow.

## Credentials: OIDC over long-lived keys

```yaml
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform
          aws-region: us-east-1
```

Rather than storing a long-lived AWS access key as a GitHub secret,
`configure-aws-credentials` with `role-to-assume` uses GitHub's own OIDC
token to assume a short-lived IAM role scoped to this specific repository
and workflow — directly extending Level 3 module 08's secrets-management
principle ("never hand out a plaintext long-lived credential when a
narrower, ephemeral one will do") to the CI system itself.

## Worked example: the full plan→policy→approve→apply chain

```text
PR opened, touches envs/prod/
  -> terraform-plan.yml runs: fmt, validate, plan, posts plan as PR comment
  -> reviewer reads the plan comment, approves the PR
PR merged to main
  -> terraform-apply.yml triggers on push to main
  -> plan + policy check (conftest) run again, against the merged code
  -> GitHub Environment pauses for required-reviewer approval
  -> approved -> terraform apply -auto-approve runs
```

Notice the plan is computed **twice** — once for review on the PR, once
again on `main` right before apply. This is deliberate: the PR's plan
could be stale by the time of merge (another PR may have merged and
applied in between), so re-planning against `main`'s actual current state
immediately before apply is what guarantees the applied plan matches
current reality, exactly the discipline Level 1 module 09's `-out=tfplan`
section described for avoiding drift between review and execution.

## How It Actually Works

**Nothing about running Terraform inside CI changes Terraform Core's own
execution model — a GitHub Actions runner is simply another machine
invoking the same CLI, against the same backend, subject to the same
state locking from Level 3 module 01.** This is precisely why the
two-plan pattern above is necessary rather than paranoid: the PR-time
plan and the merge-time plan are two entirely separate `terraform plan`
invocations, potentially minutes or hours apart, and Terraform gives no
guarantee whatsoever that a plan computed at time T is still valid at
time T+1 unless you re-plan (or apply from a saved plan file
immediately, with no gap, which the merge-triggered workflow effectively
approximates by planning and applying within the same job run).

**`-out=tfplan` followed by `apply -auto-approve` (with no plan file
argument) versus `apply tfplan` (with one) are meaningfully different
even inside CI — the pipeline above deliberately re-plans rather than
replaying a saved file from the PR stage, because a plan file cannot
safely cross job boundaries without re-validating against current state
anyway.** A saved plan file, per Level 1 module 09's "How It Actually
Works," is a frozen, serialized set of provider RPC payloads that
Terraform refuses to apply if state has changed since it was generated —
so even if this pipeline *did* try to carry the PR-time plan file forward
to the apply job, a merge to `main` between PR-approval and apply would
correctly cause Terraform to reject it outright, which is exactly the
safety property state locking and plan-file validation exist to provide
across this kind of multi-stage, human-gated pipeline.

**OIDC-based role assumption shifts the trust boundary from "a stored
secret exists and must never leak" to "a short-lived token is minted
per-run and expires automatically" — mechanically, GitHub's runner
requests a signed JWT from GitHub's own OIDC provider, and AWS STS
verifies that JWT's signature and claims (repository, branch, workflow)
against the trust policy on the target IAM role before issuing temporary
credentials.** No credential of any kind exists before the workflow starts
or after it ends, which is a fundamentally different risk profile than a
GitHub Actions secret containing a long-lived access key — a leaked OIDC
token is worthless outside the exact run that requested it, while a
leaked long-lived key remains valid until someone notices and rotates it.

## Exercise

Extend the `terraform-plan.yml` workflow above with a step that fails the
job if `terraform fmt -check` finds unformatted files, positioned *before*
the `terraform plan` step — and explain in one sentence why running
`fmt -check` before `plan` (rather than after) is the more useful ordering
for a contributor reading the CI failure.
