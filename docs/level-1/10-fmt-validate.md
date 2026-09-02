# 10 · Formatting & Validation

Terraform ships two built-in commands for keeping configurations clean and
correct before anything reaches a provider: `terraform fmt` and
`terraform validate`.

## `terraform fmt`: canonical formatting

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

## Useful `fmt` flags

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

## `terraform validate`: structural correctness

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

## Where these fit in the workflow

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

## A note on linting beyond the built-ins

`terraform validate` deliberately stays narrow — pure HCL/schema
correctness. Broader static analysis (naming conventions, deprecated
argument usage, security misconfigurations like a public S3 bucket) is the
job of third-party tools layered on top, such as `tflint` and security
scanners — covered in Level 4's security scanning module rather than here,
since they're separate tools with their own install and configuration, not
built into the Terraform CLI itself.

## Exercise

Take the "before" example at the top of this module, save it to a scratch
`main.tf`, and run `terraform fmt -diff` on it (no changes are written with
`-diff`) to see exactly what it would change. Then delete the
`instance_type` argument entirely and run `terraform validate` — confirm
the error message names the exact missing argument and the exact line of
the offending block.
