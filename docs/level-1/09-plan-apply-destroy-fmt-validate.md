---
description: "The Core Workflow: plan, apply, destroy, fmt & validate — Part 1: terraform plan / apply / destroy Workflow; Part 2: Formatting & Validation."
---

# 09 · The Core Workflow: plan, apply, destroy, fmt & validate

This module covers the everyday Terraform command workflow end to end: first planning, applying and destroying changes safely, then keeping configurations clean and correct with terraform fmt and terraform validate before you plan.

## Part 1 · terraform plan / apply / destroy Workflow

Almost every Terraform session follows the same four commands. This module
walks through each one's exact behavior and the options you'll reach for
most often.

### `terraform init`

Covered in depth in module 02 — downloads providers, sets up the backend,
writes the lock file. Safe to re-run any time; it's idempotent.

```bash
terraform init
```

Re-run it whenever you add a new provider, add a module source (Level 2),
or change backend configuration.

### `terraform validate`

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

### `terraform plan`

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

### Saving a plan for later

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

### `terraform apply`

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

### `terraform destroy`

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

### Exit codes worth knowing (for scripting)

```bash
terraform plan -detailed-exitcode
# exit code 0: no changes
# exit code 1: an error occurred
# exit code 2: changes are present
```

`-detailed-exitcode` is how CI pipelines distinguish "nothing to do" from
"there's a plan to review" without parsing the human-readable plan text —
covered further when Level 4 builds a CI/CD pipeline around this workflow.

### The full loop in order

```bash
terraform init        # once per clone, or after adding providers/modules
terraform validate    # fast, local sanity check
terraform fmt         # canonical formatting (next module)
terraform plan         # review before touching anything
terraform apply        # make it real
# ... later ...
terraform destroy      # tear it down when you're done
```

### How It Actually Works: the graph walk behind each command

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

### Exercise

Using the `local_file` configuration from module 08's exercise, run
`terraform plan` and copy the exact `+`/`~`/`-` symbol and resource address
it prints for the file being created. Then change the `content` argument's
text and run `terraform plan` again — confirm it now shows `~` (update in
place) rather than `-/+` (replace), and explain in one sentence why a
content change to a local file can be an in-place update while, per this
module's RDS example, changing an engine cannot.

## Part 2 · Formatting & Validation

Terraform ships two built-in commands for keeping configurations clean and
correct before anything reaches a provider: `terraform fmt` and
`terraform validate`.

### `terraform fmt`: canonical formatting

`fmt` rewrites `.tf` files in place to match Terraform's canonical style —
two-space indentation, aligned `=` signs within a block, consistent
spacing. It resolves style debates the same way `gofmt` or `prettier` do:
there's one style, and a command that enforces it, so nobody argues about
it in code review.

```hcl
# before terraform fmt
resource "aws_instance" "web" {
ami = "ami-0c94855ba95c71c99"
  instance_type    = "t3.micro"
    tags = {
    Name = "web"
  }
}
```

```bash
terraform fmt
```

```hcl
# after terraform fmt
resource "aws_instance" "web" {
  ami           = "ami-0c94855ba95c71c99"
  instance_type = "t3.micro"
  tags = {
    Name = "web"
  }
}
```

Notice the `=` signs for `ami` and `instance_type` are aligned — `fmt`
lines up consecutive single-line arguments within a block automatically.

### Useful `fmt` flags

```bash
terraform fmt                 # format files in the current directory
terraform fmt -recursive       # format this directory and every subdirectory
terraform fmt -check           # exit non-zero if any file isn't formatted (no changes made)
terraform fmt -diff            # show what would change, without changing it
```

`-check` is the one that matters for CI: it lets a pipeline fail a pull
request that includes unformatted HCL, the same way a linter would, without
`fmt` silently "fixing" someone's PR out from under them.

```yaml
# example CI step (conceptual, not tied to a specific CI vendor)
- run: terraform fmt -check -recursive
```

### `terraform validate`: structural correctness

`validate` checks that the configuration is internally well-formed —
without needing provider credentials or network access, since it's purely
checking against the schemas Terraform already has for the providers named
in `required_providers` (after `terraform init` has downloaded them):

```bash
terraform validate
# Success! The configuration is valid.
```

What it catches:

```hcl
resource "aws_instance" "web" {
  ami = "ami-0c94855ba95c71c99"
  # instance_type is required but missing
}
```

```bash
terraform validate
# Error: Missing required argument
#
#   on main.tf line 1, in resource "aws_instance" "web":
#    1: resource "aws_instance" "web" {
#
# The argument "instance_type" is required, but no definition was found.
```

```hcl
variable "instance_count" {
  type = number
}

resource "aws_instance" "web" {
  count = var.instance_count
  # ...
}
```

```bash
terraform apply -var="instance_count=not-a-number"
# Error: Invalid value for variable
#
#   on main.tf line 1:
#    1: variable "instance_count" {
#
# a number is required
```

What `validate` does **not** catch: anything only the live API would know,
such as an AMI ID that doesn't exist in the target region, an IAM
permission that's missing, or a resource name that's already taken. Those
only surface once `plan` (or `apply`) actually calls the provider.

### Where these fit in the workflow

```bash
terraform init
terraform fmt -check -recursive   # fail fast on style
terraform validate                # fail fast on structure
terraform plan
terraform apply
```

Running `fmt -check` and `validate` before `plan` means style and structural
mistakes are caught in seconds, locally or in a fast CI step, well before a
slower `plan` (which does hit the provider's API and can take much longer
for large configurations) even starts.

### A note on linting beyond the built-ins

`terraform validate` deliberately stays narrow — pure HCL/schema
correctness. Broader static analysis (naming conventions, deprecated
argument usage, security misconfigurations like a public S3 bucket) is the
job of third-party tools layered on top, such as `tflint` and security
scanners — covered in Level 4's security scanning module rather than here,
since they're separate tools with their own install and configuration, not
built into the Terraform CLI itself.

### How It Actually Works: two independent passes over two different trees

`fmt` and `validate` sound similar but operate on entirely different
representations of your configuration — this is reasoned through from the
documented separation in Terraform's own codebase between the `hclwrite` and
core-configschema packages.

- **`terraform fmt` never parses semantics at all.** It uses `hclwrite`, a
  layout-preserving AST that represents *tokens and formatting*, not
  evaluated meaning. It reformats whitespace, alignment, and quoting by
  rewriting that token tree back to text — it has no concept of whether
  `resource "aws_s3_bucket" "reports"` is a valid resource type, whether
  `reports` is a duplicate name, or whether an argument you wrote even
  exists in that resource's schema. This is exactly why `fmt` can succeed on
  a file that still fails `validate` immediately afterward.
- **`terraform validate` runs a completely separate pipeline**: parse with
  the semantic `hclsyntax` parser into an evaluation-ready AST, decode each
  block against Terraform Core's schema (right label count and types),
  cross-check every `var.x` / `local.x` / resource reference actually
  exists somewhere in the configuration, and run each provider's *own*
  schema validation for resource-specific argument types and required
  fields (this last step is the only part of `validate` that talks to a
  provider binary at all, and even then only against its static schema —
  never a live API).
- **Neither pass touches state or performs refresh.** Both are safe,
  offline, no-credential operations — which is exactly why they're the
  standard pre-commit / CI gate: they catch structural and schema mistakes
  before a plan ever attempts a provider RPC call that would require real
  credentials.
- **Third-party linters (tflint, Checkov, tfsec) run yet another separate
  pass**, typically walking the same parsed HCL AST that `validate` produces
  but applying their own external rule sets (naming conventions, known
  security misconfigurations) that Terraform Core's schema validation has no
  opinion on — this is why a config can pass `terraform validate` cleanly
  and still fail a linter.

### Exercise

Take the "before" example at the top of this module, save it to a scratch
`main.tf`, and run `terraform fmt -diff` on it (no changes are written with
`-diff`) to see exactly what it would change. Then delete the
`instance_type` argument entirely and run `terraform validate` — confirm
the error message names the exact missing argument and the exact line of
the offending block.
