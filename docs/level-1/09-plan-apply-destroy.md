# 09 · terraform plan / apply / destroy Workflow

Almost every Terraform session follows the same four commands. This module
walks through each one's exact behavior and the options you'll reach for
most often.

## `terraform init`

Covered in depth in module 02 — downloads providers, sets up the backend,
writes the lock file. Safe to re-run any time; it's idempotent.

```bash
terraform init
```

Re-run it whenever you add a new provider, add a module source (Level 2),
or change backend configuration.

## `terraform validate`

Checks the configuration is internally consistent — correct HCL syntax,
required arguments present, types matching — without touching any provider
or state:

```bash
terraform validate
# Success! The configuration is valid.
```

It catches typos and type errors instantly and locally; it does **not**
catch things only the provider would know (like an invalid AMI ID) — that
only shows up during `plan` or `apply`, when Terraform actually talks to the
API.

## `terraform plan`

Computes and displays what would change, without changing anything:

```bash
terraform plan
```

Plan output uses a consistent set of symbols:

```text
Terraform will perform the following actions:

  # aws_s3_bucket.reports will be created
  + resource "aws_s3_bucket" "reports" {
      + bucket = "acme-reports-2026"
      ...
    }

  # aws_instance.web will be updated in-place
  ~ resource "aws_instance" "web" {
        id            = "i-0abc123"
      ~ instance_type = "t3.micro" -> "t3.small"
    }

  # aws_s3_bucket.old_logs will be destroyed
  - resource "aws_s3_bucket" "old_logs" {
      - bucket = "acme-old-logs" -> null
    }

Plan: 1 to add, 1 to change, 1 to destroy.
```

`+` create, `~` update in place, `-` destroy, and (not shown above)
`-/+` destroy-and-recreate — used when a change requires replacing the
resource entirely because the provider doesn't support updating that
particular attribute in place (e.g., changing an RDS instance's engine).
Terraform always tells you *which* case applies and why, right above the
diff for that resource.

## Saving a plan for later

```bash
terraform plan -out=tfplan
terraform apply tfplan
```

Saving the plan to a file and applying *that exact file* guarantees the
applied changes are precisely what was reviewed — nothing can have drifted
between plan and apply (e.g., someone else applying a conflicting change
in between). This is the standard pattern in CI/CD pipelines (Level 4
covers this fully): plan in one job, require human approval, apply the
saved plan in a separate job.

## `terraform apply`

Without a saved plan file, `apply` computes a fresh plan, shows it, and
prompts for confirmation:

```bash
terraform apply
# ... plan output ...
# Do you want to perform these actions?
#   Terraform will perform the actions described above.
#   Only 'yes' will be accepted to approve.
#
#   Enter a value: yes
```

```bash
# skip the interactive prompt (common in scripts/CI — use deliberately)
terraform apply -auto-approve

# target only specific resources (an escape hatch, not a routine habit)
terraform apply -target=aws_s3_bucket.reports
```

`-auto-approve` is appropriate for CI pipelines with their own review gate
(a required PR approval before the pipeline runs) but risky to reach for by
habit in a terminal — it removes the last chance to catch a plan you didn't
expect. `-target` similarly is meant for narrow, deliberate fixes (e.g.
recovering from a partial failure) rather than routine use, since applying
only part of a dependency graph can leave state and configuration
out of sync in ways later plans then need to reconcile.

## `terraform destroy`

Computes a plan to remove every resource currently tracked in state, shows
it, and prompts the same way:

```bash
terraform destroy
# Plan: 0 to add, 0 to change, 3 to destroy.
#
# Do you really want to destroy all resources?
#   Enter a value: yes
```

```bash
# destroy just one resource and everything that depends on it
terraform destroy -target=aws_s3_bucket.reports
```

`destroy` only removes what's in *state* — resources that exist in the real
provider but were never tracked by this Terraform state (created some other
way) are untouched, because as far as this state file is concerned they
don't exist.

## Exit codes worth knowing (for scripting)

```bash
terraform plan -detailed-exitcode
# exit code 0: no changes
# exit code 1: an error occurred
# exit code 2: changes are present
```

`-detailed-exitcode` is how CI pipelines distinguish "nothing to do" from
"there's a plan to review" without parsing the human-readable plan text —
covered further when Level 4 builds a CI/CD pipeline around this workflow.

## The full loop in order

```bash
terraform init        # once per clone, or after adding providers/modules
terraform validate    # fast, local sanity check
terraform fmt         # canonical formatting (next module)
terraform plan         # review before touching anything
terraform apply        # make it real
# ... later ...
terraform destroy      # tear it down when you're done
```

## How It Actually Works: the graph walk behind each command

Reasoned through from Terraform Core's documented execution model — these
commands were not run against a live cloud account for this lesson.

- **`terraform validate` never touches the graph or any provider RPC.** It
  only runs HCL syntax parsing plus Terraform's own static schema checks
  (right block labels, referenced variables exist, type constraints are
  satisfiable) — no `Configure`, no `ReadResource`, no network access at
  all, which is why it's safe to run with no credentials configured.
- **`terraform plan` builds two graphs, not one.** First a "refresh graph"
  (walks existing state, calls `ReadResource` per resource to detect drift),
  then a "plan graph" that layers your desired configuration on top and
  calls `PlanResourceChange` per node. Independent resources (no edge
  between them in the DAG) are refreshed and planned *concurrently*, bounded
  by `-parallelism` (default 10) — this is why plan/apply time doesn't scale
  linearly with resource count as long as resources are independent.
- **A saved plan (`-out=tfplan`) is a serialized, frozen execution list.**
  Because it already contains resolved provider RPC payloads, running
  `terraform apply tfplan` skips refresh and re-diffing — it replays exactly
  those calls in the recorded graph order. This is also why Terraform
  refuses to apply a stale plan file if the state has changed since it was
  generated (a mismatched serial/lineage check against current state).
- **`terraform apply` walks the *same* DAG but calls `ApplyResourceChange`
  instead of `PlanResourceChange`,** and updates the in-memory state
  incrementally as each node completes — successfully-applied resources are
  persisted to the state file (via the backend) even if a later resource in
  the same apply fails, which is why a partially-failed apply doesn't lose
  track of what did succeed.
- **`terraform destroy` reverses the graph.** It's not a separate code path
  from apply — it's a plan computed as "destroy everything in state," and
  the destroy *order* is the dependency graph walked in reverse (a subnet
  is destroyed only after every instance inside it), because destroying in
  forward dependency order would fail against most real cloud APIs (you
  generally can't delete a VPC while instances still reference it).
- **Exit codes are graph-walk outcomes, not arbitrary conventions:** `0` =
  graph walk completed with no changes needed, `1` = an error occurred
  during parsing/planning/applying, `2` (plan only, with `-detailed-exitcode`)
  = the graph walk succeeded and found a non-empty diff — useful precisely
  because it lets CI distinguish "plan failed" from "plan succeeded and
  there's drift to review."

## Exercise

Using the `local_file` configuration from module 08's exercise, run
`terraform plan` and copy the exact `+`/`~`/`-` symbol and resource address
it prints for the file being created. Then change the `content` argument's
text and run `terraform plan` again — confirm it now shows `~` (update in
place) rather than `-/+` (replace), and explain in one sentence why a
content change to a local file can be an in-place update while, per this
module's RDS example, changing an engine cannot.
