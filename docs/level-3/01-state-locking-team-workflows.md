# 01 · State Locking & Team Workflows

Level 2 module 03 introduced remote backends; this module covers the
mechanism that makes them safe for a *team*, not just a single developer:
**state locking**, and the workflow conventions built around it.

## The problem locking solves

```text
Alice: terraform apply   (reads state, computes plan, starts applying)
Bob:   terraform apply   (reads the SAME state, computes a plan, starts applying)
```

Without a lock, two concurrent `apply` runs both read the same starting
state, both compute their own plan against it, and both write their own
final state back when done — whichever finishes last **overwrites** the
other's changes in the state file, even though both sets of real
infrastructure changes actually happened. The state file, not reality, is
what silently loses information.

## How locking looks in practice

```bash
terraform apply
# Acquiring state lock. This may take a few moments...
```

```bash
# a second, concurrent run:
terraform apply
# Error: Error acquiring the state lock
#
# Lock Info:
#   ID:        d1e6d8d1-...
#   Path:      acme-terraform-state/reports/terraform.tfstate
#   Operation: OperationTypeApply
#   Who:       alice@alice-laptop
#   Created:   2026-09-11 14:02:11 UTC
```

The second run refuses to proceed at all — rather than silently plan
against stale state, Terraform fails loudly and tells you exactly who
holds the lock and since when, so the conflict is visible immediately
instead of surfacing later as a corrupted state file.

## Backend-specific locking mechanisms

Locking is implemented per-backend, not by Terraform Core itself:

- **S3 backend**: uses a paired **DynamoDB table** for locking (the
  `dynamodb_table` argument from Level 2 module 03) — S3 objects have no
  native compare-and-swap lock primitive, so the backend uses DynamoDB's
  conditional writes to implement one.
- **Terraform Cloud/Enterprise**: locking is a first-class API feature of
  the platform itself (module 03 of this level covers this in depth).
- **Local backend**: uses an OS-level file lock on `terraform.tfstate`,
  which only protects against concurrent processes on the *same machine*
  — effectively no protection at all for a team, which is the practical
  reason teams move off local state.

## Force-unlocking (last resort)

```bash
terraform force-unlock d1e6d8d1-0000-0000-0000-000000000000
```

Only ever appropriate when you're certain the process that acquired the
lock crashed and will never release it (a killed CI job, a laptop that
lost power mid-apply) — force-unlocking while a run is genuinely still in
progress reintroduces exactly the race condition locking exists to
prevent, potentially corrupting state worse than the stuck lock itself
would have.

## Team workflow conventions built on locking

1. **Never run `apply` from a laptop against shared state for anything
   that matters** — route applies through a single CI pipeline (Level 4)
   so there's one place locking is exercised consistently and one place to
   look for history.
2. **`plan` before every `apply`, and read it** — locking prevents
   concurrent *writes*, not a stale mental model of what a plan will do;
   review discipline is a separate practice against a separate risk.
3. **Keep applies short** — a lock is held for the duration of the
   operation; a `plan`/`apply` that takes twenty minutes because it
   touches hundreds of resources blocks every teammate's run for that
   whole window.

## How It Actually Works

**A lock is acquired before Terraform reads state for planning, and held
until state is written back — the entire plan-and-apply cycle happens
under one lock, not just the final write.** This is deliberate: locking
only during the write would still let two runs both read the same "before"
state and independently compute conflicting plans, with only the last
write winning silently. By holding the lock across the full read-plan-apply
sequence, Terraform guarantees whoever holds it is working from state that
cannot change underneath them for the operation's whole duration — the
second run's `terraform apply` fails at the *acquire* step, before it ever
reads state at all, rather than failing later after wasted work.

**The DynamoDB-backed S3 lock is a conditional-write compare-and-swap, not
a heartbeat or lease.** Acquiring the lock is a single DynamoDB
`PutItem` call with a condition that the lock-ID attribute must not already
exist; releasing it is a `DeleteItem` on that same key. There's no
background renewal process — if the process holding the lock is killed
(a crashed CI runner, a laptop that loses power), the DynamoDB item simply
stays there indefinitely, un-deleted, which is exactly the scenario
`force-unlock` exists for: a human has to make the judgment call that the
lock is now stale rather than the system inferring it automatically,
because Terraform has no reliable way to distinguish "the holder is doing
a very long apply" from "the holder crashed" without risking a live lock
being torn down.

**Locking says nothing about concurrent *reads* of state, only concurrent
plan/apply operations that intend to write.** `data "terraform_remote_state"`
(Level 2 module 03) and `terraform show`/`state list` don't acquire the
write lock at all — only `plan`, `apply`, and `destroy` (and a few `state`
subcommands that mutate state directly, like `mv`/`rm`) do, which is why
a `terraform_remote_state` read from a downstream configuration can safely
run at any time without contending with an in-progress `apply` upstream,
though it may then be reading state that's about to change moments later.

## Exercise

Two teammates both run `terraform apply` against the same S3+DynamoDB
backed state within seconds of each other. Write out, in order, exactly
what each of them sees in their terminal — including what the second
teammate's error message would contain — and explain why `force-unlock`
would be the wrong response if the first `apply` were still genuinely
running.
