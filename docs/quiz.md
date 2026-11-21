---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Terraform Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this configuration do when it is applied?\n```hcl\nvariable \"enable_monitoring\" {\n  type    = bool\n  default = false\n}\nresource \"aws_cloudwatch_metric_alarm\" \"cpu\" {\n  count = var.enable_monitoring ? 1 : 0\n}\n```\n",
    "options": [
      "Creates no alarm by default; with enable_monitoring = true it creates exactly one, addressed as ...cpu[0]",
      "Fails: count can't be conditional",
      "Creates two alarms",
      "Always creates one alarm"
    ],
    "correctIndex": 0,
    "explanation": "count = condition ? 1 : 0 is the conditional-resource idiom, and a resource with count is addressed by index."
  },
  {
    "question": "`condition ? \"a\" : 5` fails validation. Why?",
    "options": [
      "Strings can't be used in conditionals",
      "Conditionals need parentheses",
      "5 is invalid",
      "Both branches must produce compatible types so Terraform knows the result type before it knows which branch runs"
    ],
    "correctIndex": 3,
    "explanation": "HCL has no if/else statement, only the conditional expression."
  },
  {
    "question": "You use `count` over [\"web-1\", \"web-2\", \"web-3\"] and remove \"web-2\". What happens to the instances?",
    "options": [
      "All three are recreated by design",
      "Only web-2 is destroyed",
      "Everything after the removed index shifts: web[1] now means web-3 and web[2] disappears, causing needless changes; for_each keyed by name touches only web-2",
      "Nothing changes"
    ],
    "correctIndex": 2,
    "explanation": "Identity by position versus by key. Verified with a small simulation: count gives an update to web[1] and a destroy of web[2]; for_each gives one destroy."
  },
  {
    "question": "Which type fits `for_each` over instance names?",
    "options": [
      "A single string",
      "A list(string)",
      "A set(string) or a map, addressed as aws_instance.web[\"web-1\"]",
      "A number"
    ],
    "correctIndex": 2,
    "explanation": "for_each accepts a set of strings or a map; each.key and each.value are available."
  },
  {
    "question": "A teammate renamed `aws_instance.web` to `aws_instance.app_server` in code, and the plan wants to destroy and create. What is the safest fix?",
    "options": [
      "Apply and accept the recreate",
      "Add a moved { from = aws_instance.web to = aws_instance.app_server } block (or terraform state mv)",
      "Delete the state file",
      "Use terraform import"
    ],
    "correctIndex": 1,
    "explanation": "State knows resources by address, so a rename looks like a different resource. A moved block makes the migration part of code review."
  },
  {
    "question": "You imported an existing bucket with `terraform import aws_s3_bucket.reports acme-reports-2026` but haven't written the resource block. What will the next plan do?",
    "options": [
      "Propose to destroy it, because import records state but doesn't write configuration; write the matching resource block first",
      "Do nothing",
      "Create a second bucket",
      "Generate the .tf for you"
    ],
    "correctIndex": 0,
    "explanation": "Import writes to state only."
  },
  {
    "question": "Two engineers run `terraform apply` simultaneously on the same S3 backend with DynamoDB locking. What happens to the second run?",
    "options": [
      "It merges both plans",
      "It runs on a copy of state",
      "It waits silently and overwrites",
      "It fails with 'Error acquiring the state lock' and shows who holds it"
    ],
    "correctIndex": 3,
    "explanation": "Without a lock, the last writer would overwrite the other's state changes."
  },
  {
    "question": "Which backend settings differ between dev and prod so their locks can't block each other?",
    "options": [
      "The provider alias",
      "The module names",
      "The Terraform version",
      "The state key (e.g. platform/dev/terraform.tfstate vs platform/prod/terraform.tfstate)"
    ],
    "correctIndex": 3,
    "explanation": "Same bucket and lock table, different keys, give independent state and locks."
  },
  {
    "question": "Why can't a backend block use variables?",
    "options": [
      "Backends are secret",
      "Variables are slow",
      "It's just a style rule",
      "Terraform must resolve where state lives before evaluating anything else; use literals or -backend-config"
    ],
    "correctIndex": 3,
    "explanation": "Partial configuration supplies the values at init time."
  },
  {
    "question": "Your CI wants machine-readable plan output for a policy engine. Which commands?",
    "options": [
      "terraform fmt -json",
      "terraform validate -json",
      "terraform state list",
      "terraform plan -out=tfplan then terraform show -json tfplan"
    ],
    "correctIndex": 3,
    "explanation": "Sentinel and OPA/conftest evaluate the JSON plan (resource_changes[].change.after), not your .tf text."
  },
  {
    "question": "A plan proposes `t3.xlarge` but your policy allows only t3.micro/small/medium. Which layer should block it before apply?",
    "options": [
      "terraform fmt",
      "A policy check (Sentinel or OPA/conftest) on the plan, in addition to least-privilege IAM",
      "Only IAM",
      "terraform validate"
    ],
    "correctIndex": 1,
    "explanation": "IAM bounds what a pipeline can do; policy bounds what it may configure. Verified with a small policy checker over a JSON plan."
  },
  {
    "question": "A security group rule was widened to 0.0.0.0/0 in the console. How do scheduled pipelines detect it?",
    "options": [
      "terraform output",
      "terraform fmt -check",
      "terraform plan -detailed-exitcode returns 2 when there's a diff, which fails the job",
      "terraform validate"
    ],
    "correctIndex": 2,
    "explanation": "Exit 0 means in sync, 1 means error, 2 means changes."
  },
  {
    "question": "A bucket has all four public_access_block settings set to false. What does `terraform validate` say?",
    "options": [
      "It auto-fixes them",
      "Invalid: public buckets are rejected",
      "A warning about security",
      "Valid: it checks syntax, types and references, not security posture; use an IaC scanner or policy"
    ],
    "correctIndex": 3,
    "explanation": "Schema validity and security posture are orthogonal."
  },
  {
    "question": "A secret password is hard-coded in a .tf file and later removed in a new commit. Is it safe now?",
    "options": [
      "No: it remains in Git history; rotate it, and use sensitive variables or a secrets manager (state can still hold values)",
      "Yes after terraform apply",
      "Yes if the repo is private",
      "Yes, removed from history"
    ],
    "correctIndex": 0,
    "explanation": "Treat state as sensitive too."
  },
  {
    "question": "You lost terraform.tfstate but the cloud resources still exist. What will the next plan propose?",
    "options": [
      "Destroying everything",
      "Creating everything from scratch (failing on collisions or duplicating); restore versioned state from the S3 bucket instead",
      "Renaming resources",
      "Nothing"
    ],
    "correctIndex": 1,
    "explanation": "State loss doesn't touch real resources; versioning and replication on the state bucket are the first lines of defence."
  },
  {
    "question": "Which module design rule keeps a module reusable at scale?",
    "options": [
      "No knowledge of specific teams, environments or accounts; those decisions live in the composition root",
      "Wrap exactly one resource",
      "Hard-code the team name",
      "Avoid variables"
    ],
    "correctIndex": 0,
    "explanation": "A hard-coded team or account ID makes callers fork the module."
  },
  {
    "question": "Which version constraint accepts 5.9.2 but not 6.0.0?",
    "options": [
      "~> 5.8.1",
      "> 6.0",
      "= 5.8.0",
      "~> 5.8 (>= 5.8.0, < 6.0.0)"
    ],
    "correctIndex": 3,
    "explanation": "~> 5.8.1 allows only patch releases (< 5.9.0)."
  },
  {
    "question": "When is `depends_on` justified?",
    "options": [
      "Never",
      "Always, for safety",
      "To pass values between modules",
      "When ordering matters but no value is referenced, such as an IAM policy attachment that must exist first"
    ],
    "correctIndex": 3,
    "explanation": "References already create implicit dependencies, which are preferred."
  },
  {
    "question": "Why does a Terraform plan not show what a change will cost?",
    "options": [
      "Provider schemas describe shape, not price; cost tools read the JSON plan and look up a separate price catalog",
      "Pricing is encrypted",
      "Terraform forbids pricing",
      "Costs are hidden in state"
    ],
    "correctIndex": 0,
    "explanation": "Infracost-style tools walk resource_changes[].change.after."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Why count shifts and for_each doesn't

Write the two address builders: addresses_count(names) maps each position to an address like aws_instance.web[0], and addresses_for_each(names) maps each name to aws_instance.web["web-1"]. The provided plan() compares before and after to show create, update and destroy. Print the plan for removing "web-2" from the middle of the list.

**Starter code**

```python
def addresses_count(names):
    # position -> aws_instance.web[0], aws_instance.web[1], ...
    pass

def addresses_for_each(names):
    # key -> aws_instance.web["web-1"], ...
    pass

def plan(before, after):
    destroy = sorted(a for a in before if a not in after)
    create = sorted(a for a in after if a not in before)
    update = sorted(a for a in after if a in before and before[a] != after[a])
    return {"create": create, "update": update, "destroy": destroy}

old = ["web-1", "web-2", "web-3"]
new = ["web-1", "web-3"]
print("count:", plan(addresses_count(old), addresses_count(new)))
print("for_each:", plan(addresses_for_each(old), addresses_for_each(new)))
```

**Expected output**

```text
count: {'create': [], 'update': ['aws_instance.web[1]'], 'destroy': ['aws_instance.web[2]']}
for_each: {'create': [], 'update': [], 'destroy': ['aws_instance.web["web-2"]']}
```

<details><summary>Show solution and explanation</summary>

<pre><code>def addresses_count(names):
    # count addresses instances by position: web[0], web[1], ...
    return {f&quot;aws_instance.web[{i}]&quot;: n for i, n in enumerate(names)}

def addresses_for_each(names):
    # for_each addresses instances by key
    return {f&#x27;aws_instance.web[&quot;{n}&quot;]&#x27;: n for n in names}

def plan(before, after):
    destroy = sorted(a for a in before if a not in after)
    create = sorted(a for a in after if a not in before)
    update = sorted(a for a in after if a in before and before[a] != after[a])
    return {&quot;create&quot;: create, &quot;update&quot;: update, &quot;destroy&quot;: destroy}

old = [&quot;web-1&quot;, &quot;web-2&quot;, &quot;web-3&quot;]
new = [&quot;web-1&quot;, &quot;web-3&quot;]
print(&quot;count:&quot;, plan(addresses_count(old), addresses_count(new)))
print(&quot;for_each:&quot;, plan(addresses_for_each(old), addresses_for_each(new)))
</code></pre>

<p>With count, identity is position, so removing the middle element shifts web-3 into index 1 (an update, or a replace for attributes that force one) and drops index 2. With for_each, identity is the key, so only web-2 is removed. Verified by running the solution.</p>
</details>

### A plan policy check

Write violations(plan) that walks resource_changes and reports, for every aws_instance being created or updated (skip deletes where change.after is null), an instance_type outside the allowlist and any missing required tags. This is the same idea as a Sentinel or OPA policy over terraform show -json.

**Starter code**

```python
import json

ALLOWED = {"t3.micro", "t3.small", "t3.medium"}
REQUIRED_TAGS = {"Project", "Environment"}

PLAN = json.loads("""
{"resource_changes": [
  {"address": "aws_instance.web",  "type": "aws_instance",
   "change": {"actions": ["create"], "after": {"instance_type": "t3.xlarge", "tags": {"Project": "shop", "Environment": "prod"}}}},
  {"address": "aws_instance.api",  "type": "aws_instance",
   "change": {"actions": ["create"], "after": {"instance_type": "t3.small", "tags": {"Project": "shop"}}}},
  {"address": "aws_s3_bucket.logs", "type": "aws_s3_bucket",
   "change": {"actions": ["create"], "after": {"bucket": "shop-logs"}}},
  {"address": "aws_instance.old", "type": "aws_instance",
   "change": {"actions": ["delete"], "after": null}}
]}
""")

def violations(plan):
    out = []
    # your code: append strings like "aws_instance.web: instance_type t3.xlarge not allowed"
    return out

for v in violations(PLAN):
    print("FAIL", v)
print("passed" if not violations(PLAN) else f"{len(violations(PLAN))} violations")
```

**Expected output**

```text
FAIL aws_instance.web: instance_type t3.xlarge not allowed
FAIL aws_instance.api: missing tags ['Environment']
2 violations
```

<details><summary>Show solution and explanation</summary>

<pre><code>import json

ALLOWED = {&quot;t3.micro&quot;, &quot;t3.small&quot;, &quot;t3.medium&quot;}
REQUIRED_TAGS = {&quot;Project&quot;, &quot;Environment&quot;}

PLAN = json.loads(&quot;&quot;&quot;
{&quot;resource_changes&quot;: [
  {&quot;address&quot;: &quot;aws_instance.web&quot;,  &quot;type&quot;: &quot;aws_instance&quot;,
   &quot;change&quot;: {&quot;actions&quot;: [&quot;create&quot;], &quot;after&quot;: {&quot;instance_type&quot;: &quot;t3.xlarge&quot;, &quot;tags&quot;: {&quot;Project&quot;: &quot;shop&quot;, &quot;Environment&quot;: &quot;prod&quot;}}}},
  {&quot;address&quot;: &quot;aws_instance.api&quot;,  &quot;type&quot;: &quot;aws_instance&quot;,
   &quot;change&quot;: {&quot;actions&quot;: [&quot;create&quot;], &quot;after&quot;: {&quot;instance_type&quot;: &quot;t3.small&quot;, &quot;tags&quot;: {&quot;Project&quot;: &quot;shop&quot;}}}},
  {&quot;address&quot;: &quot;aws_s3_bucket.logs&quot;, &quot;type&quot;: &quot;aws_s3_bucket&quot;,
   &quot;change&quot;: {&quot;actions&quot;: [&quot;create&quot;], &quot;after&quot;: {&quot;bucket&quot;: &quot;shop-logs&quot;}}},
  {&quot;address&quot;: &quot;aws_instance.old&quot;, &quot;type&quot;: &quot;aws_instance&quot;,
   &quot;change&quot;: {&quot;actions&quot;: [&quot;delete&quot;], &quot;after&quot;: null}}
]}
&quot;&quot;&quot;)

def violations(plan):
    out = []
    for rc in plan[&quot;resource_changes&quot;]:
        after = rc[&quot;change&quot;][&quot;after&quot;]
        if rc[&quot;type&quot;] != &quot;aws_instance&quot; or after is None:
            continue
        if after[&quot;instance_type&quot;] not in ALLOWED:
            out.append(f&quot;{rc[&#x27;address&#x27;]}: instance_type {after[&#x27;instance_type&#x27;]} not allowed&quot;)
        missing = REQUIRED_TAGS - set(after.get(&quot;tags&quot;, {}))
        if missing:
            out.append(f&quot;{rc[&#x27;address&#x27;]}: missing tags {sorted(missing)}&quot;)
    return out

for v in violations(PLAN):
    print(&quot;FAIL&quot;, v)
print(&quot;passed&quot; if not violations(PLAN) else f&quot;{len(violations(PLAN))} violations&quot;)
</code></pre>

<p>Policies are queries over the JSON plan&#x27;s planned values (change.after), not over .tf text, which is why one policy can cover every environment. Verified by running the solution.</p>
</details>
