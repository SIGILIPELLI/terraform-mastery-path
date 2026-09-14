# 03 · Drift Detection & Remediation

**Drift** is when real infrastructure no longer matches what Terraform's
state believes — someone changed a setting in the cloud console, an
auto-scaling event modified something outside Terraform's control, or a
different automation tool touched a resource Terraform also manages.
Level 1 module 07 introduced the concept; this module covers detecting it
systematically and choosing the right remediation.

## Scheduled drift detection in CI

```yaml
# .github/workflows/drift-detection.yml
name: Drift Detection
on:
  schedule:
    - cron: "0 8 * * *"   # daily at 08:00 UTC

jobs:
  detect:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        env: [dev, staging, prod]
    defaults:
      run:
        working-directory: envs/${{ matrix.env }}
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - id: plan
        run: terraform plan -detailed-exitcode -no-color
        continue-on-error: true
      - if: steps.plan.outputs.exitcode == '2'
        run: |
          echo "Drift detected in ${{ matrix.env }}"
          exit 1   # fails the job, triggering the team's normal CI-failure alert
```

`-detailed-exitcode` (introduced in Level 1 module 09) is exactly what
makes automated drift detection practical: exit code `2` distinguishes
"there's a real diff to look at" from `0` ("nothing changed, still in
sync") without any human reading plan text, letting a scheduled job run
across every environment nightly and only make noise when something
actually needs attention.

## Reading a drift-only plan

```text
aws_security_group_rule.web_https: Refreshing state... [id=sgrule-0abc]

Terraform detected the following changes made outside of Terraform since
the last "terraform apply":

  # aws_security_group_rule.web_https has changed
  ~ resource "aws_security_group_rule" "web_https" {
        id                = "sgrule-0abc"
      ~ cidr_blocks       = [
          - "10.0.0.0/8",
          + "0.0.0.0/0",
        ]
    }

Unless you have made equivalent changes to your configuration, or ignored
the relevant attributes using ignore_changes, the following plan may
include actions to undo or respond to these changes.
```

Terraform explicitly labels this as detected drift, distinct from a change
your own `.tf` edits caused — someone widened a security group rule to
`0.0.0.0/0` directly in the console, and the next `plan` both surfaces
that fact loudly *and* proposes reverting it back to the configured
`10.0.0.0/8`, because as far as the configuration is concerned, the
narrower range is still the desired state.

## Remediation option 1: apply, reverting the drift

```bash
terraform apply
```

The default, and usually correct, response — configuration is the source
of truth, so applying pushes reality back in line with it. Appropriate
when the manual change was accidental, unauthorized, or simply
undocumented.

## Remediation option 2: accept the drift into configuration

```hcl
resource "aws_security_group_rule" "web_https" {
  cidr_blocks = ["0.0.0.0/0"]   # updated to match the manually-widened rule
  # ...
}
```

Appropriate when the manual change was actually correct and should
become the new desired state — update the `.tf` file to match reality,
then the next `plan` shows no diff at all, because configuration and
(now-accepted) reality agree.

## Remediation option 3: `ignore_changes` for attributes managed elsewhere

```hcl
resource "aws_autoscaling_group" "web" {
  desired_capacity = 2
  # ...

  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```

For attributes a *different* system legitimately owns after initial
creation (an autoscaling policy adjusting `desired_capacity` based on
load, which Terraform would otherwise see as drift on every plan and
propose to revert), `ignore_changes` tells Terraform to stop diffing that
specific attribute against real-world state entirely after creation — the
correct tool exactly when "drift" isn't actually a problem to fix, but an
expected, ongoing division of ownership.

## Worked example: a security-group drift runbook

```bash
# 1. Scheduled job detects drift, fails, pages the on-call
terraform plan -detailed-exitcode
# exit code 2

# 2. On-call reads the plan output, identifies it's the web_https rule
terraform state show aws_security_group_rule.web_https

# 3. Decision: was this an authorized emergency change, or unauthorized?
#    - Unauthorized -> terraform apply (revert it)
#    - Authorized, should persist -> update the .tf file to match, then apply
#      (a no-op apply at that point, confirming config now matches reality)
```

## How It Actually Works

**Drift is only detectable because every `plan` performs the refresh step
described in Level 1 module 07's "How It Actually Works" — Terraform calls
each resource's `ReadResource` RPC every time, comparing the provider's
live answer against state, regardless of whether your `.tf` files changed
at all.** This is the mechanical reason drift detection needs no special
Terraform feature beyond a scheduled `terraform plan`: refresh-and-diff is
already happening on every single plan you've run throughout this course;
a "drift detection job" is simply a `plan` run on a schedule instead of on
demand, with `-detailed-exitcode` used to make the *absence* of change the
uninteresting, silent case.

**`ignore_changes` works by excluding specific attributes from the
three-way comparison at diff time, not by skipping the refresh read for
those attributes.** Terraform still calls `ReadResource` and still learns
the real, drifted value of `desired_capacity` — `ignore_changes` operates
one step later, at `PlanResourceChange` time, telling the provider to
compute the planned new state *as if* the ignored attribute's configured
value were whatever was just read from reality, rather than whatever your
`.tf` file says. This is why `terraform state show` after an
`ignore_changes`-covered drift still reflects the real, current
autoscaled value — the state file itself stays accurate to reality; only
the *diff* against configuration is suppressed for that attribute.

**Scheduled detection via `-detailed-exitcode` in CI relies on drift
being a pure function of (current configuration, current real
infrastructure) at the moment the job runs — which means a drift job's
result can itself be racy against a concurrent human `apply`.** If a
legitimate `apply` is in-flight when the scheduled detection job's `plan`
runs, the detection job could either see a state-locked failure (module
01's locking preventing the read entirely, depending on backend and
timing) or a transient false-positive diff mid-apply. Real drift-detection
pipelines typically schedule these jobs during low-change windows, or
treat a single non-zero detection as a signal to *investigate* rather than
an automatic trigger to remediate, specifically because of this
possible race with legitimate concurrent applies.

## Exercise

A nightly drift-detection job reports `exit code 2` for the `prod`
environment, and the plan shows an `aws_instance.web` tag was changed
from `Team = "platform"` to `Team = "sre"` directly in the console.
Write the two possible remediation paths (revert vs. accept) as concrete
next actions, and state which one you'd choose by default per this
module's own guidance, absent any other context.
