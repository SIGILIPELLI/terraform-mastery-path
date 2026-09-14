# 06 · Multi-Environment Patterns

Level 2 module 04 flagged that CLI workspaces share backend config and
credentials — genuinely separate environments (dev/staging/prod, often
different accounts) need a stronger isolation boundary. This module covers
the two dominant real-world patterns: **directory-per-environment** and
**workspace-per-environment via Terraform Cloud**, plus the shared-module
structure that keeps either from becoming a maintenance burden.

## Pattern 1: directory-per-environment

```text
envs/
  dev/
    main.tf
    backend.tf      # backend key: "dev/terraform.tfstate"
    terraform.tfvars
  staging/
    main.tf
    backend.tf      # backend key: "staging/terraform.tfstate"
    terraform.tfvars
  prod/
    main.tf
    backend.tf      # backend key: "prod/terraform.tfstate"
    terraform.tfvars
modules/
  app/
  network/
```

```hcl
# envs/prod/main.tf
module "network" {
  source     = "../../modules/network"
  cidr_block = "10.2.0.0/16"
  az_count   = 3
}

module "app" {
  source            = "../../modules/app"
  vpc_id            = module.network.vpc_id
  subnet_ids        = module.network.public_subnet_ids
  instance_count    = 4
  enable_monitoring = true
}
```

Each environment is a **fully separate Terraform configuration** with its
own state file, its own backend key, and — critically — its own provider
credentials, which can point at an entirely different cloud account for
prod versus dev. The shared logic lives once in `modules/`; each
environment's `main.tf` is deliberately small, mostly just different input
values passed to the same modules. This is the pattern Level 2's capstone
was implicitly built to scale into.

## Pattern 2: `.tfvars` per environment, one configuration

```hcl
# environments/dev.tfvars
environment     = "dev"
instance_count  = 1
az_count        = 2
```

```hcl
# environments/prod.tfvars
environment     = "prod"
instance_count  = 4
az_count        = 3
```

```bash
terraform apply -var-file=environments/prod.tfvars
```

A lighter-weight variant: one configuration, environment differences
captured entirely as variable values, with the environment selected by
which `-var-file` you pass. This works well when environments genuinely
share the *same* backend/account and differ only in sizing — but it
reintroduces the CLI-workspace risk of accidentally applying the wrong
`.tfvars` against the wrong state if you're not careful about which
backend/workspace is currently selected, which directory-per-environment
avoids structurally.

## Choosing between the two

| | Directory-per-environment | `.tfvars`-per-environment |
|---|---|---|
| State isolation | separate files/backends by construction | same backend unless combined with CLI workspaces |
| Different credentials/accounts per env | natural | awkward — provider config is usually shared |
| Risk of applying wrong env accidentally | low (separate `cd`, separate backend) | higher (forgetting `-var-file` or switching workspace) |
| Duplication | more directories, less logic (all logic in `modules/`) | one directory, more variables |

Most teams managing genuinely separate prod accounts converge on
directory-per-environment; `.tfvars`-driven single configurations are more
common for lower-stakes variants within one account (feature branches,
short-lived preview environments).

## Worked example: promoting a change from dev to prod safely

```bash
cd envs/dev
terraform plan
terraform apply
# ... validate in dev ...

cd ../staging
terraform plan
terraform apply
# ... validate in staging ...

cd ../prod
terraform plan   # same module version, same change, now reviewed twice already
terraform apply
```

Because `dev`, `staging`, and `prod` are separate state files pointing at
the same versioned modules (Level 2 module 02's pinning), a change to
`modules/app` can be rolled out environment by environment, each with its
own independent `plan` for review, rather than one giant `apply` touching
every environment's resources simultaneously.

## How It Actually Works

**Directory-per-environment achieves isolation for free, at the level
Terraform Core actually operates at: each directory is a wholly separate
root module, with its own `.terraform` cache, its own resolved backend
pointer, and therefore its own independent graph.** There is no shared
in-memory or on-disk state connecting `envs/dev` and `envs/prod` at
runtime at all — running `terraform apply` in one has zero mechanical way
to affect the other's state, which is a much stronger guarantee than "we
have a convention of always double-checking which workspace is selected"
provides for the CLI-workspace pattern in Level 2 module 04, where a
single mis-typed `terraform workspace select` is the only thing standing
between you and applying against the wrong environment.

**`modules/app` being referenced identically from three separate root
configurations means three separate graph expansions of the same
`.tf` source, each with its own independently-resolved variable values —
not three instances sharing anything at evaluation time.** This is exactly
the module-composition mechanism from Level 2 modules 01 and 09, just with
the *caller* being an entire separate root configuration/state file
instead of a sibling `module` block in the same root — the module itself
has no idea, and no way to detect, whether it's being called from `dev` or
`prod`; every difference in behavior comes purely from the different input
values each environment's `main.tf` happens to pass in.

**`.tfvars`-per-environment achieves a *weaker* form of the same idea by
reusing one configuration's already-built graph shape and only swapping
the leaf `var.*` values feeding into it** — the risk profile difference in
the comparison table above is a direct consequence of this: because
`terraform apply -var-file=prod.tfvars` and
`terraform apply -var-file=dev.tfvars` both operate against whatever
backend/workspace is *currently selected* rather than a structurally
separate one, the correctness of "which environment does this apply
affect" depends on external, unenforced state (which CLI workspace is
active) rather than being baked into the directory you happen to be
standing in.

## Exercise

Sketch the directory layout (just the tree, not full file contents) for
promoting the Level 2 capstone into three environments —
`envs/dev`, `envs/staging`, `envs/prod` — each calling the existing
`modules/network` and `modules/app`, with a shared `modules/` directory
at the same level as `envs/`. Then write the one argument difference
you'd expect between `envs/dev/main.tf`'s and `envs/prod/main.tf`'s
`module "app"` block, based on this course's own capstone's
`local.is_prod` conditional.
