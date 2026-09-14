# 09 · Observability for Infrastructure Changes

Every module so far has assumed you're watching a `plan`/`apply` run in
a terminal, in real time. At platform scale, most applies happen in CI,
unattended, and the question shifts from "what does this apply do" to
"how would anyone know, after the fact, what changed, when, and who
approved it" — without re-running `terraform show` on every historical
plan by hand.

## Why `apply` output alone isn't observability

```text
aws_security_group_rule.web_https: Modifying... [id=sgr-0abc123]
aws_security_group_rule.web_https: Modification complete after 1s
```

This line, scrolling past in a CI log, is the entire record of a
security group rule changing — unless someone is watching that specific
job's log at that specific moment, or knows to go find it later, it's
effectively invisible. CI log retention windows expire; logs from a
hundred daily pipeline runs across dozens of repos aren't searchable as
a coherent timeline. Observability for infrastructure changes means
deliberately exporting three things every apply produces — the plan
diff, the actual resource-level events, and the identity/approval
metadata — into systems built to retain and query them.

## Structured logging: `TF_LOG_CORE` / `TF_LOG` in JSON

```bash
export TF_LOG=INFO
export TF_LOG_JSON=1
terraform apply -auto-approve 2>&1 | tee apply.jsonl
```

```json
{"@level":"info","@message":"aws_instance.app: Creation complete after 42s","@module":"terraform.ui","@timestamp":"2026-03-04T10:15:22.104Z","type":"apply_complete","hook":{"resource":{"addr":"aws_instance.app"},"action":"create","elapsed_seconds":42}}
```

`TF_LOG_JSON=1` switches Terraform's CLI UI output from human-readable
text to structured JSON lines, with a stable schema per event type
(`apply_complete`, `diagnostic`, `refresh_complete`, etc.). This is what
turns "log output" into "log data" — a log-shipping agent can parse each
line as a JSON object and forward it to a log aggregator (CloudWatch
Logs, Datadog, an ELK stack) with `resource.addr`, `action`, and
`elapsed_seconds` as queryable fields, rather than a blob of text
requiring regex to extract anything.

## Retaining the plan diff itself, not just the apply log

```bash
terraform plan -out=tfplan
terraform show -json tfplan > plan-$(date +%s).json
aws s3 cp plan-*.json s3://acme-tf-audit/production/plans/
```

The apply log tells you what happened to each resource during execution;
it doesn't tell you *why* — what the reviewer actually saw and approved.
Archiving every plan's JSON output (the same artifact consumed by cost
estimation and security scanning in modules 04–05) to durable storage,
keyed by timestamp and workspace, is what lets someone later answer "what
exactly was proposed in the change that modified this security group on
March 4th" without needing the CI job's log to still exist.

## Correlating change events with an audit trail: who, what, when

```json
{
  "workspace": "production",
  "run_id": "run-CZcmQ9mVXwFwSMPr",
  "vcs_commit": "a1b2c3d",
  "triggered_by": "jane@acme.com",
  "approved_by": "platform-lead@acme.com",
  "plan_summary": { "add": 1, "change": 2, "destroy": 0 },
  "applied_at": "2026-03-04T10:15:00Z"
}
```

Terraform Cloud/Enterprise emits exactly this kind of run metadata via
its Audit Trails API/webhook — every run's triggering commit, requester,
approver, and plan summary, independent of the resource-level apply log.
Piping this into the same aggregation system as the structured apply
logs and archived plan JSON turns three separate artifacts (who approved
it, what was proposed, what actually happened) into one queryable
timeline per infrastructure change, which is what an incident
retrospective or a compliance audit actually needs — not any single one
of those three in isolation.

## Worked example: wiring drift detection into the same observability pipeline

```yaml
# .github/workflows/drift-detect.yml
on:
  schedule:
    - cron: "0 */6 * * *"
jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - run: terraform plan -detailed-exitcode -out=tfplan; echo "exitcode=$?" >> "$GITHUB_OUTPUT"
        id: plan
      - if: steps.plan.outputs.exitcode == '2'
        run: |
          terraform show -json tfplan > drift.json
          aws s3 cp drift.json s3://acme-tf-audit/production/drift/drift-$(date +%s).json
          curl -X POST "$SLACK_WEBHOOK" -d "{\"text\":\"Drift detected in production — see drift.json in audit bucket\"}"
```

Level 4's drift-detection module (03) covers *finding* drift with
`-detailed-exitcode`; wiring the detected drift's JSON plan into the same
S3 audit prefix and alerting channel as regular applies means drift
shows up in the same timeline as deliberate changes — someone reviewing
"what happened to this resource last week" sees both the applies your
team ran and any drift Terraform caught, in one place, instead of drift
detection living in a separate, disconnected pipeline nobody checks.

## How It Actually Works

**`TF_LOG_JSON` doesn't add new information to what Terraform Core
already tracks internally — it switches the *serialization* of the same
UI event stream Core always emits to drive the human-readable CLI
output.** Every "Creating...", "Modifying...", "complete after Ns" line
you've seen throughout this course originates from Core's UI hook
system emitting a typed event (resource address, action, timestamp,
elapsed time) after each provider RPC completes; the human-readable
renderer and the JSON renderer are two different consumers of that exact
same event stream. This is why JSON logging costs nothing in terms of
information — it's strictly a format change on data Core was already
producing, which is also why it's safe to enable everywhere without
changing apply behavior at all.

**A saved plan file's JSON representation and the audit-trail metadata
from Terraform Cloud/Enterprise are two independent systems recording
two different scopes: the plan JSON is Core's understanding of one
specific proposed diff, while the run's audit metadata is Terraform
Cloud/Enterprise's record of the *workflow* around that diff (who
triggered it, who approved it) — Core itself has no concept of
"approval" at all.** Approval, requester identity, and run sequencing
are entirely implemented in the orchestration layer sitting on top of
Core (Terraform Cloud/Enterprise, or an equivalent CI system's own
approval gates) — which is exactly why self-managed CI pipelines have to
build and retain this metadata themselves (as in the JSON example
above) if they want it, since nothing about `terraform plan`/`apply`
running in a bare CI job produces an approval record on its own.

**Drift detected via `-detailed-exitcode` (exit code 2) produces exactly
the same JSON plan schema as a deliberate change's plan — Core has no
separate code path or output format for "this diff exists because
someone edited the cloud console" versus "this diff exists because I
changed the .tf file" — both are just a computed difference between
current real-world state (refreshed from the provider) and current
configuration.** This is precisely why drift's JSON output can flow into
the identical audit-storage and alerting pipeline as regular applies
with zero special-casing: from Core's perspective, and therefore from
the observability tooling's perspective, drift is just another plan
diff — the only thing distinguishing it is that no corresponding
`apply` follows it in the same run, which the pipeline in the worked
example detects via the plan-only exit code, not via anything in the
plan's own content.

## Exercise

A production security group rule was changed outside Terraform (someone
edited it directly in the cloud console) three weeks ago, and it was
never reconciled. Using only the observability mechanisms in this
module — structured apply logs, archived plan JSON, audit-trail run
metadata, and the scheduled drift-detection pipeline — describe how you
would locate: (a) when the drift was first detected, and (b) whether any
*deliberate* Terraform apply around that same time might have been the
actual cause instead of an out-of-band console edit.
