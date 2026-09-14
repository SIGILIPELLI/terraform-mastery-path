# 08 · Secrets Management Patterns

Level 1 module 07 warned that state can contain plaintext secrets. This
module covers the practical patterns for keeping secrets out of both
version control *and* state as much as possible, and for the residual
cases where a secret unavoidably ends up in state, how to limit exposure.

## Never hardcode secrets in `.tf` files

```hcl
# WRONG — never do this
resource "aws_db_instance" "main" {
  username = "admin"
  password = "hunter2"   # committed to Git, forever, in every clone's history
}
```

Any value written directly into a `.tf` file is committed to version
control the moment the file is — and removing it from a later commit does
**not** remove it from Git history, which anyone with repo access (and
several trivial tools) can still read.

## Pattern 1: `sensitive = true` on variables

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

resource "aws_db_instance" "main" {
  username = "admin"
  password = var.db_password
}
```

```bash
terraform plan
# ~ password = (sensitive value)
```

`sensitive = true` tells Terraform to redact the value from **CLI output**
— plan/apply logs, `terraform show`, error messages that would otherwise
echo it back. It does **not** encrypt or omit the value from the state
file itself; a sensitive-marked value is still stored in plaintext in
`terraform.tfstate`, which is exactly why remote-state encryption (Level 2
module 03's `encrypt = true`) and tight access control on the state
backend remain necessary regardless of this flag.

## Pattern 2: injecting secrets via environment variables, never variable defaults

```bash
export TF_VAR_db_password="$(vault kv get -field=password secret/db)"
terraform apply
```

```hcl
variable "db_password" {
  type      = string
  sensitive = true
  # deliberately NO default — forces the value to come from outside
}
```

Omitting `default` entirely on a secret-holding variable means Terraform
will refuse to proceed without the value being supplied some other way
(a `TF_VAR_` environment variable, `-var`, or a `.tfvars` file that is
itself excluded from version control) — the missing default is a
deliberate guardrail against someone reflexively hardcoding a "just for
now" placeholder default that then gets committed.

## Pattern 3: reading secrets from a secrets manager at plan time

```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  username = "admin"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

The secret never appears in your `.tf` source or in any `.tfvars` file at
all — Terraform fetches it live from the secrets manager during `plan`,
using the plan-time credentials it already has. This still lands the
plaintext value in state (the `data` source's result is recorded there
like any other), so it doesn't eliminate the state-encryption requirement,
but it does eliminate the much larger exposure surface of the secret
sitting in Git history or in a `.tfvars` file that might accidentally get
committed.

## Pattern 4: letting the provider generate the secret, never handling it yourself

```hcl
resource "random_password" "db" {
  length  = 24
  special = true
}

resource "aws_db_instance" "main" {
  username = "admin"
  password = random_password.db.result
}

resource "aws_secretsmanager_secret" "db_password" {
  name = "prod/db/password"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db.result
}
```

Terraform generates the password itself (`random_password`), uses it to
create the database, and immediately writes it into a secrets manager —
no human, CI variable, or external system ever needs to know or transmit
the plaintext value at all; it's generated and consumed entirely within
one `apply`. This still lands in state (both `random_password.db.result`
and the secret resource), reinforcing that state encryption is the one
mitigation that's never optional, no matter which of these patterns you
combine it with.

## What `.gitignore` should always exclude

```text
*.tfvars
*.tfvars.json
.terraform/
*.tfstate
*.tfstate.backup
```

`*.tfvars` is excluded by default in this list because it's the single
most common place a developer pastes a real credential "temporarily"
while testing locally — excluding the pattern entirely removes the chance
of accidentally `git add -A`-ing it later.

## How It Actually Works

**`sensitive = true` operates purely at the CLI presentation layer,
implemented as a taint that propagates through the expression evaluator
alongside the value itself, not as encryption.** When Terraform Core
evaluates an expression that touches a sensitive-marked variable, it marks
the *resulting* value as sensitive too, and that mark propagates through
any further expressions built from it — string-interpolating a sensitive
value into a larger string still marks the result sensitive, which is why
`password = "postgres://admin:${var.db_password}@..."` still gets redacted
in plan output even though the sensitive value is only part of the final
string. The mark is stripped only at the point of writing to state, which
is precisely the gap that makes remote-state encryption non-optional: the
protection this flag provides evaporates the instant something reads the
raw state file rather than going through Terraform's own CLI rendering.

**A `data` source reading a secrets manager at plan time still goes
through the identical provider-RPC read path (`ReadDataSource`) as any
other data source (Level 1 module 05), and its result is persisted into
state via the exact same state-write step every other resource and data
source uses.** There is no separate "secret" code path in Terraform Core
at all — from the engine's point of view, a secrets-manager-fetched
password and a hardcoded AMI ID are the same kind of value flowing through
the same graph nodes; the *only* difference is which pattern above you
choose to keep the plaintext out of your own version-controlled files
before that read ever happens.

**Provider-generated secrets (`random_password`) are computed locally by
Terraform Core itself, with no plan-time "unknown" placeholder the way
provider-assigned IDs normally have** — `random_password` is implemented
by the `random` provider's own resource logic, which generates the value
during `apply` (not plan, since it needs to be stable across runs once
created) and returns it as a normal computed attribute, immediately
available for any downstream resource in the same apply to consume via a
plain reference — this is why the database and its Secrets Manager entry
above can both consume `random_password.db.result` in a single `apply`
with full ordering guaranteed by the same reference-based dependency
graph from module 07, with no manual coordination needed between the
password's generation and its two consumers.

## Exercise

Rewrite the hardcoded-password `aws_db_instance` example at the top of
this page using pattern 4 (provider-generated password stored in Secrets
Manager) end to end, and write one sentence explaining why
`terraform state show aws_db_instance.main` would still display the
plaintext password even after applying `sensitive = true` to every
relevant variable.
