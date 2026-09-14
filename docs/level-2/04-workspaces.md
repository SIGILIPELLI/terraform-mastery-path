# 04 · Workspaces

A **workspace** lets one configuration directory manage multiple, separate
state files without duplicating any `.tf` code — useful for quick
variations of the same infrastructure (a personal sandbox per developer,
or a lightweight dev/staging split) without the fuller multi-environment
patterns Level 3 module 06 covers.

## The default workspace

Every configuration starts with exactly one workspace, `default` — you've
been using it, invisibly, throughout Level 1 and so far in Level 2.

```bash
terraform workspace show
# default
```

## Creating and switching workspaces

```bash
terraform workspace new staging
# Created and switched to workspace "staging"!

terraform workspace list
#   default
# * staging

terraform workspace select default
```

Each workspace gets its own state, isolated from every other workspace in
the same configuration — `terraform apply` in `staging` creates resources
tracked in `staging`'s state only, entirely independent of what `default`
or any other workspace has created.

## Using `terraform.workspace` in configuration

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcd1234"
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"

  tags = {
    Name        = "web-${terraform.workspace}"
    Environment = terraform.workspace
  }
}
```

`terraform.workspace` is a built-in reference (not a variable you declare)
holding the current workspace's name as a string, letting one configuration
adapt its behavior — instance sizing, naming, tag values — per workspace
without any conditionals outside what's shown above.

## Where workspace state actually lives

For the default local backend, each non-default workspace's state file
lands in `terraform.tfstate.d/<workspace>/terraform.tfstate` — `default`
alone keeps using the plain `terraform.tfstate` in the working directory.
For a remote backend like S3, the backend itself namespaces state per
workspace (the S3 backend appends `env:/<workspace>/` to the configured
`key` automatically); you never manage that path yourself.

## Worked example: a sandbox-per-developer pattern

```bash
terraform workspace new alice-sandbox
terraform apply   # creates resources tagged Environment = "alice-sandbox"

terraform workspace new bob-sandbox
terraform apply   # entirely separate resources, own state
```

Both developers run the exact same `.tf` files; `terraform.workspace`
threaded into `tags.Environment` (as shown above) is enough to keep their
resources distinguishable and their states from ever colliding.

## Why workspaces are *not* a substitute for separate environments

Workspaces share the same backend configuration, the same variable
defaults (unless you explicitly branch on `terraform.workspace`), and the
same provider credentials — there is no built-in way to say "the `prod`
workspace must use a different AWS account." For anything where dev and
prod need genuinely different credentials, approval gates, or blast-radius
isolation, Level 3 module 06 covers directory-per-environment instead,
which keeps environments in entirely separate backend configurations and
state files by construction, not by convention.

## How It Actually Works

**A workspace is a state-storage namespace, and nothing else.** Switching
workspaces does not re-evaluate your configuration differently in any way
Terraform enforces on its own — it only changes which state file `plan`
and `apply` read and write. Every difference in behavior you see between
workspaces (a bigger instance type in `prod`, different tags) exists
purely because *your own* `.tf` code happens to reference
`terraform.workspace` in a conditional. Delete every `terraform.workspace`
reference from a configuration and workspaces still work exactly the same
mechanically — they'd just produce identical resources in each one, in
separate state files.

**`terraform workspace select`/`new` mutates a tiny local pointer file, not
your working directory's `.tf` files.** For the local backend this pointer
lives at `.terraform/environment`; for a remote backend, the backend
provider itself is asked which workspace is "current" and the same file
still records the local client's selection. Every Terraform command in
that directory reads this pointer first, before touching any state, which
is why forgetting you're in the wrong workspace is a common real-world
mistake: the `.tf` files on disk give you no visual cue at all about which
workspace's state a command is about to touch — only `terraform workspace
show` does.

**Backends implement per-workspace isolation differently, but the state
manager interface hides that from the graph builder.** The S3 backend
namespaces by object key (`env:/<workspace>/<key>`); other backends might
use a separate database row or a separate object path scheme entirely.
Terraform Core's plan/apply logic never sees this — it calls the same
"read current state" / "write new state" interface regardless of backend,
with the backend implementation solely responsible for translating
"current workspace" into wherever that backend physically stores it.

## Exercise

Using the `terraform.workspace` conditional pattern above, write the
`tags` block you'd add to an `aws_s3_bucket` resource so that a bucket
created in the `staging` workspace is named `acme-staging-artifacts` and
one created in `prod` is named `acme-prod-artifacts`, using
`terraform.workspace` directly in the bucket name via string
interpolation — no `count` or `for_each` needed.
