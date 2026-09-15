---
description: "count vs for_each — Both count and for_each turn a single resource block into multiple instances. They solve overlapping problems but produce different…"
---

# 08 · count vs for_each

Both `count` and `for_each` turn a single resource block into multiple
instances. They solve overlapping problems but produce different state
addressing schemes, and picking the wrong one is one of the most common
sources of unnecessary resource recreation in real Terraform codebases.

## `count`: index-based repetition

```hcl
variable "instance_names" {
  type    = list(string)
  default = ["web-1", "web-2", "web-3"]
}

resource "aws_instance" "web" {
  count         = length(var.instance_names)
  ami           = "ami-0abcd1234"
  instance_type = "t3.micro"

  tags = {
    Name = var.instance_names[count.index]
  }
}
```

Each instance is addressed by a numeric index: `aws_instance.web[0]`,
`aws_instance.web[1]`, `aws_instance.web[2]`. Inside the block,
`count.index` (0-based) is how each instance picks its own values out of a
list.

## `for_each`: key-based repetition

```hcl
variable "instance_names" {
  type    = set(string)
  default = ["web-1", "web-2", "web-3"]
}

resource "aws_instance" "web" {
  for_each      = var.instance_names
  ami           = "ami-0abcd1234"
  instance_type = "t3.micro"

  tags = {
    Name = each.value
  }
}
```

Each instance is addressed by its **key**, not a position:
`aws_instance.web["web-1"]`, `aws_instance.web["web-2"]`. `for_each`
accepts a `set(string)` or a `map(...)`; inside the block, `each.key` and
`each.value` are available (identical for a set, distinct for a map).

## The critical difference: what happens when the middle element is removed

```hcl
# Before: ["web-1", "web-2", "web-3"]
# After:  ["web-1", "web-3"]   (removed web-2)
```

With **`count`**: `web-3` was `aws_instance.web[2]`; after removal, the
list has only two elements, so what *was* index `2` (`web-3`) no longer
has a corresponding list entry, and everything shifts — Terraform sees
index `1` now means `web-3` instead of `web-2`, and proposes **destroying
and recreating** `aws_instance.web[1]` even though `web-3` itself didn't
conceptually change at all.

With **`for_each`** (using a set/map of names as keys): removing `web-2`
from the set only removes the graph node keyed `"web-2"`. `aws_instance.web["web-1"]`
and `aws_instance.web["web-3"]` are untouched — because their identity in
state is their *key*, not their position, and neither key's associated
value changed.

This is the single reason most real-world module authors default to
`for_each` over `count` whenever the elements have a natural, stable
identity (a name, an ID) — `count` is appropriate mainly when you're
repeating something truly identical N times with no meaningful per-instance
identity beyond "the Nth one" (e.g., `count = 3` identical availability-zone
subnets addressed only by index).

## Worked example: converting a `count` list to a `for_each` map

```hcl
variable "buckets" {
  type = map(object({
    versioning = bool
  }))
  default = {
    reports = { versioning = true }
    logs    = { versioning = false }
  }
}

resource "aws_s3_bucket" "this" {
  for_each = var.buckets
  bucket   = "acme-${each.key}"
}

resource "aws_s3_bucket_versioning" "this" {
  for_each = { for k, v in var.buckets : k => v if v.versioning }
  bucket   = aws_s3_bucket.this[each.key].id

  versioning_configuration {
    status = "Enabled"
  }
}
```

The second `for_each` filters the map down to only the buckets that want
versioning, using the `for ... if` pattern from module 06 — a resource's
`for_each` can be any expression that produces a set or map, including one
computed inline from another variable.

## `count.index` vs `each.key`/`each.value` — quick reference

| | `count` | `for_each` |
|---|---|---|
| Per-instance reference | `count.index` (number) | `each.key`, `each.value` |
| Address in state | `resource[0]`, `resource[1]` | `resource["key1"]`, `resource["key2"]` |
| Accepts | a number | a `set` or `map` |
| Stable identity on reorder/removal | no — shifts by position | yes — keyed |
| Typical use | N identical things by position | things with a natural name/key |

## How It Actually Works

**Both `count` and `for_each` are resolved during graph construction into
a fixed set of resource instance addresses, before any instance is
individually planned — the difference is entirely in how that address is
computed.** For `count`, Terraform evaluates the numeric expression to `N`
and generates addresses `[0]` through `[N-1]` — pure arithmetic, with no
memory of what value previously occupied a given index. For `for_each`,
Terraform evaluates the set/map expression and uses each **key** verbatim
as the instance address — the address is a direct, stable function of the
input data itself, not of iteration order, which is exactly why removing
one map key doesn't perturb any other key's address.

**This is why `for_each`'s diffing is a set/map comparison, while
`count`'s diffing is a list-length comparison.** When Terraform computes a
plan for a `for_each` resource, it compares the *previous* set of keys
(from state) against the *new* set of keys (from the current
configuration) — any key present in both is left alone (even if its
position in the underlying map's iteration order changed, since maps have
no inherent order Terraform relies on); any key only in the new set is a
create; any key only in the old set is a destroy. `count`'s diff, by
contrast, is purely "old length vs. new length," with every index above
the smaller of the two lengths becoming a destroy or create regardless of
whether the *values* at those shifted indices are actually new.

**`count.index` and `each.key`/`each.value` are graph-local variables,
scoped to exactly one resource instance's evaluation.** They aren't global
loop variables the way `for` in an imperative language would produce —
each resource instance is its own independent graph node, and
`count.index`/`each.key` is simply the piece of that node's identity fed
back into its own body expressions during that one instance's
`PlanResourceChange` call. This is also why you cannot reference
`count.index` or `each.key` from a *different* resource's block — they
only exist within the block whose `count`/`for_each` produced them.

## Exercise

Take the `count`-based `aws_instance.web` example at the top of this page
and rewrite it using `for_each` over a `set(string)` of the same three
names, addressing each instance's `Name` tag via `each.value` instead of
`var.instance_names[count.index]`. Then write one sentence on what would
happen to `aws_instance.web["web-2"]` in each version if `"web-1"` were
removed from the input list/set.
