# 07 · Dependency Management & the Graph

Every prior module in this course has invoked "the graph" to explain why
something behaves the way it does. This module makes the dependency graph
itself the subject — how to see it, how implicit and explicit dependencies
differ mechanically, and how to reason about what can run concurrently.

## Visualizing the graph

```bash
terraform graph | dot -Tsvg > graph.svg
```

`terraform graph` prints the configuration's dependency graph in DOT
format (Graphviz's graph-description language); piping it through `dot`
renders an actual diagram. For anything beyond a handful of resources the
rendered graph is usually too dense to read directly, but it's genuinely
useful for confirming a suspected dependency exists (or, more often,
confirming one *doesn't* — that two resources you assumed were
independent really are, and can safely be reasoned about separately).

## Implicit dependencies: reference-based (the default, and the goal)

```hcl
resource "aws_vpc" "this" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.this.id   # <- creates the graph edge
  cidr_block = "10.0.1.0/24"
}
```

`aws_subnet.main` referencing `aws_vpc.this.id` is what creates the
dependency edge — Terraform Core parses every expression in every
resource block, and any `aws_vpc.this` reference anywhere becomes an edge
from that referencing resource back to `aws_vpc.this`. This is how nearly
every dependency in a well-written configuration is established: as a
side effect of actually needing a value, never as a separate declaration.

## Explicit dependencies: `depends_on`

```hcl
resource "aws_iam_role_policy" "app" {
  role   = aws_iam_role.app.id
  policy = data.aws_iam_policy_document.app.json
}

resource "aws_instance" "app" {
  ami           = "ami-0abcd1234"
  instance_type = "t3.micro"
  depends_on    = [aws_iam_role_policy.app]
}
```

Used only when `aws_instance.app` needs the IAM policy attachment to have
already happened, but nothing in `aws_instance.app`'s own arguments
actually *references* `aws_iam_role_policy.app` — a real-world case where
an underlying API has a behavioral ordering requirement Terraform's
attribute-level dependency inference has no way to see, since there's no
value being passed between them at all.

## Reading a plan's parallelism

```text
aws_vpc.this: Creating...
aws_vpc.this: Creation complete after 2s [id=vpc-0abc]
aws_subnet.a: Creating...
aws_subnet.b: Creating...
aws_subnet.a: Creation complete after 1s [id=subnet-0aaa]
aws_subnet.b: Creation complete after 1s [id=subnet-0bbb]
```

`aws_subnet.a` and `aws_subnet.b` both depend on `aws_vpc.this` but not on
each other — the graph has no edge between them — so once `aws_vpc.this`
completes, Terraform's graph walker starts both concurrently (bounded by
`-parallelism`, default 10), which is exactly why their "Creating..."
lines interleave in the output above rather than appearing strictly one
after the other.

## `terraform.tf`'s often-overlooked ordering trap: implicit-looking code that isn't

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcd1234"
  instance_type = "t3.micro"
  tags          = { Name = "web-${random_pet.suffix.id}" }
}

resource "random_pet" "suffix" {}
```

Despite `random_pet.suffix` being declared *after* `aws_instance.web` in
the file, the graph edge (`web`'s `tags` references `random_pet.suffix.id`)
means `random_pet.suffix` is created first — **file order has no bearing
on execution order.** This surprises newcomers reading `.tf` files
top-to-bottom expecting sequential-script semantics; the graph, built from
references, is the only thing that determines order, regardless of how
the blocks happen to be arranged on disk.

## How It Actually Works

**Terraform Core's graph is a directed acyclic graph (DAG) built in two
passes: first every resource/module/output/local/data node is created,
then every expression in every node is walked to add an edge for each
reference found.** This second pass is purely syntactic reference
resolution — it doesn't execute or evaluate anything yet, it just notices
"this argument's expression mentions `aws_vpc.this`" and adds an edge.
Only after the full graph exists does Terraform validate it's acyclic
(a resource cannot, even transitively through several hops, depend on
itself) and then compute a valid topological ordering — any ordering
consistent with every edge — for the actual walk.

**The walker processes the DAG using a work-queue model, not a strict
sequential topological sort: any node whose dependencies have *all*
already completed becomes immediately eligible to run, and multiple
eligible nodes run concurrently up to the `-parallelism` limit.** This is
mechanically why `aws_subnet.a` and `aws_subnet.b` above interleave in the
CLI output — both become eligible at the exact same moment (the instant
`aws_vpc.this` finishes), and the walker's worker pool picks up both
without waiting for either to finish first. It's also why increasing
`-parallelism` on a configuration with many independent resources can
meaningfully speed up `apply` — the ceiling is the DAG's actual
"width" (how many nodes are simultaneously eligible) intersected with
whatever concurrency limit you allow, not some fixed per-resource cost.

**`depends_on` inserts a graph edge with no accompanying data flow — it
constrains ordering without the receiving node getting a value out of
it.** Every *implicit* edge (a real reference) carries a value alongside
the ordering guarantee: the referencing node literally cannot evaluate
its own expression until the referenced node's value exists, so ordering
and data availability are the same constraint enforced once. A
`depends_on` edge is Terraform Core adding ordering with no corresponding
expression dependency — which is exactly why it's a blunter, more manual
tool: nothing about the referencing resource's *evaluation* actually
requires the dependency to run first except the explicit edge you wrote,
so it's on you to make sure that edge reflects a real requirement, since
Terraform can't infer or verify *why* it's there the way it can for a
value reference.

## Exercise

Given `aws_route_table.public` (referencing `aws_vpc.this.id`),
`aws_route.internet` (referencing both `aws_route_table.public.id` and
`aws_internet_gateway.this.id`), and `aws_internet_gateway.this`
(referencing `aws_vpc.this.id`), draw (in words, listing which nodes have
edges to which) the dependency graph among these four resources, and
state which two of them could be created concurrently with each other.
