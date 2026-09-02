# 05 · Resources & Data Sources

Terraform configurations are built almost entirely out of two block types:
`resource` (things Terraform creates and manages) and `data` (things
Terraform only reads).

## Resources: things Terraform owns

A `resource` block tells Terraform "make sure this exists, with these
settings" — and Terraform takes responsibility for creating it, updating it
when the block changes, and destroying it if the block is removed.

```hcl
resource "aws_s3_bucket" "reports" {
  bucket = "acme-reports-2026"
}

resource "aws_s3_bucket_versioning" "reports" {
  bucket = aws_s3_bucket.reports.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

The general shape is `resource "<type>" "<local_name>" { ... }`:

- `<type>` (`aws_s3_bucket`) is defined by the provider — it determines
  which API calls Terraform makes and which arguments are valid.
- `<local_name>` (`reports`) is *your* name for this instance, scoped to
  this configuration — used to reference it elsewhere, as
  `aws_s3_bucket_versioning.reports` does above via
  `aws_s3_bucket.reports.id`.

That second resource block also demonstrates the most common pattern in
Terraform configurations: **referencing one resource's attribute from
another**. Terraform automatically infers that
`aws_s3_bucket_versioning.reports` depends on `aws_s3_bucket.reports` and
creates them in the right order — no separate "depends on" step needed for
this implicit case (module 07 of Level 3 covers explicit dependencies).

## Data sources: things Terraform only reads

A `data` block looks up information about something that already exists —
created outside this configuration (by hand, by another team, by a separate
Terraform state) — without taking any ownership of it.

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
}
```

`data "aws_ami" "amazon_linux"` doesn't create an AMI — it queries AWS for
the newest Amazon Linux AMI matching the filter, at plan time, and exposes
its attributes (like `.id`) for other blocks to use. Running
`terraform destroy` never removes anything a `data` block referred to,
because Terraform never created it in the first place.

## The resource vs. data source decision

| Question | Answer |
|---|---|
| Should Terraform create/destroy this? | `resource` |
| Does this already exist and I just need its details? | `data` |
| Is it managed by another team/system I shouldn't touch? | `data` |
| Do I want it removed on `terraform destroy`? | `resource` (data sources are never destroyed — there's nothing to destroy) |

A common real-world pattern: use a `data` source to look up a shared VPC
that a networking team manages in its own Terraform state, then create
`resource` blocks for your own application's resources *inside* that VPC by
referencing the data source's outputs (like `data.aws_vpc.shared.id`).

## Meta-arguments available on both

A handful of special arguments work on (almost) every resource and data
block, regardless of type — covered fully in Level 2, but worth knowing they
exist:

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  count = 2   # meta-argument: create 2 of these (aws_instance.web[0], [1])

  lifecycle {   # meta-argument: nested block controlling create/destroy behavior
    create_before_destroy = true
  }
}
```

`count`, `for_each`, `depends_on`, `lifecycle`, and `provider` are
meta-arguments — recognized by Terraform itself, not by the provider, which
is why they work identically across every resource type.

## Referencing attributes

Every resource and data source exposes attributes you can reference from
elsewhere in the configuration, in the form
`<type>.<local_name>.<attribute>`:

```hcl
resource "aws_s3_bucket" "reports" {
  bucket = "acme-reports-2026"
}

output "bucket_arn" {
  value = aws_s3_bucket.reports.arn   # computed by AWS, known after apply
}
```

`bucket` was an argument *you* set; `arn` is an attribute *AWS* computes and
Terraform reads back — both are accessed the same way, but only attributes
that are genuinely known ahead of time (like a literal `bucket` name you
chose) are available during `terraform plan`; ones assigned by the provider
(like an ARN containing an AWS account ID) show as `(known after apply)`
in the plan output until the resource is actually created.

## Exercise

Sketch (in HCL, without needing to apply it) a `data` block that looks up an
existing resource of your choosing by a filter or name, and a `resource`
block that uses one of that data source's attributes as one of its own
arguments. Add one comment explaining why you chose `data` instead of
`resource` for the looked-up thing.
