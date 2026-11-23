---
title: "Terraform Interview Questions and Answers (183 Questions)"
description: "183 Terraform interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Terraform Mastery Path."
---

# Terraform Interview Questions and Answers

**183 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Terraform Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Is IaC &amp; Why Terraform?](level-1/01-what-is-iac-why-terraform.md)

**Q1.** What problems does clicking through a console have that IaC fixes?

**Answer:** No history, not repeatable, and not reviewable

IaC makes infrastructure changes look like software changes: a diff, a review, a merge, then an apply.

**Q2.** What does 'declarative' mean for Terraform?

**Answer:** You describe the end state and Terraform works out the steps

It compares desired state to actual state and applies only the difference.

**Q3.** What kind of tool is Terraform?

**Answer:** A provisioning tool that creates, updates and destroys infrastructure resources across many providers

Ansible, Chef and Puppet handle configuration inside running machines.

**Q4.** What is the problem with an imperative shell script like `aws s3 mb ...`?

**Answer:** It must be re-run carefully to avoid errors on a second run (bucket already exists)

A declarative tool is idempotent by design.

### [02 · Installing Terraform &amp; Providers](level-1/02-installing-terraform-providers.md)

**Q5.** What does Terraform's core binary know, and what doesn't it know?

**Answer:** It knows the HCL language and plan/apply/state workflow; it has no built-in knowledge of AWS or any other API

That knowledge lives in providers, plugins downloaded separately.

**Q6.** What does required_version = "&gt;= 1.6.0, &lt; 2.0.0" do?

**Answer:** Documents and enforces which CLI versions are acceptable; plan fails immediately on an incompatible version

It avoids confusing syntax or behaviour mismatches later.

**Q7.** Why use tfenv or tfswitch?

**Answer:** Different projects may require different Terraform versions, like nvm for Node

tfenv install 1.9.5 then tfenv use 1.9.5.

**Q8.** How is Terraform distributed?

**Answer:** As a single self-contained binary with no runtime to install

Verify with terraform version.

### [03 · HCL Syntax Basics](level-1/03-hcl-syntax-basics.md)

**Q9.** In `resource "aws_instance" "web" { ... }`, what are the two labels?

**Answer:** The resource type (aws_instance) and the local name (web)

provider and variable blocks take one label; terraform takes none.

**Q10.** What is the difference between `tags = { ... }` and `ebs_block_device { ... }`?

**Answer:** tags is an argument whose value is a map; ebs_block_device is a nested block with no =

The provider's schema decides which form a resource uses.

**Q11.** What is the preferred single-line comment style?

**Answer:** # comment

// and /* */ are also valid.

**Q12.** What can the right-hand side of an argument be?

**Answer:** An expression: a literal, a reference, a function call or a combination

The left side is always an identifier.

### [04 · Providers &amp; the Provider Registry](level-1/04-providers-registry.md)

**Q13.** Where do AWS provider credentials come from, in order, when none are in the provider block?

**Answer:** Environment variables, the shared credentials file, then an attached IAM role

Literal credentials in HCL would be committed to version control and written to state in plaintext.

**Q14.** What does the provider block configure, compared with required_providers?

**Answer:** required_providers says which plugin to download; the provider block configures how to talk to it (region, profile…)

Arguments are defined entirely by each provider, so check its registry page.

**Q15.** What does `version = "~> 5.0"` accept?

**Answer:** Any 5.x version, not 6.0

~&gt; allows the rightmost component to increment.

**Q16.** What does each registry provider page document?

**Answer:** Resources, data sources, argument references, import instructions and guides

The registry also hosts reusable modules.

### [05 · Resources &amp; Data Sources](level-1/05-resources-data-sources.md)

**Q17.** What is the difference between a resource and a data block?

**Answer:** A resource is managed (created, updated, destroyed) by Terraform; a data block only reads something that already exists

terraform destroy never removes anything a data block referred to.

**Q18.** How does Terraform know to create aws_s3_bucket before aws_s3_bucket_versioning?

**Answer:** The reference aws_s3_bucket.reports.id creates an implicit dependency

Explicit dependencies are covered in Level 3.

**Q19.** What does `data "aws_ami" "amazon_linux"` do?

**Answer:** Queries AWS at plan time for the newest matching AMI and exposes its attributes

You'd then use data.aws_ami.amazon_linux.id.

**Q20.** What is &lt;local_name&gt; in resource "aws_s3_bucket" "reports"?

**Answer:** Your name for the instance, scoped to this configuration, used to reference it elsewhere

The type is defined by the provider.

### [06 · Variables &amp; Outputs](level-1/06-variables-outputs.md)

**Q21.** What happens when you pass a string where `type = number` is declared?

**Answer:** It fails at plan time with a clear type-mismatch error

Types catch mistakes early.

**Q22.** In which order do variable sources take precedence (lowest to highest)?

**Answer:** default, terraform.tfvars, *.auto.tfvars, -var-file, -var

Later wins over earlier.

**Q23.** What does object({ engine = string, storage_gb = number }) describe?

**Answer:** A structured value with named, typed fields

It groups related settings instead of many flat variables.

**Q24.** How do you reference a variable named environment?

**Answer:** var.environment

Outputs expose values to whoever ran the configuration or other configurations.

### [07 · State Basics](level-1/07-state-basics.md)

**Q25.** Which three things does Terraform compare on each run?

**Answer:** Configuration, state, and real infrastructure

State is the record of what Terraform believes exists and how it maps to your config.

**Q26.** What does state contain for each resource?

**Answer:** The mapping from the Terraform address to every attribute the provider returned, including computed ones like arn

That's why outputs like bucket_arn work without a separate lookup.

**Q27.** Why is state sensitive?

**Answer:** It records every attribute, so it can contain plaintext secrets such as generated passwords or keys, even if not in your .tf files

Treat it like a secret and restrict access.

**Q28.** Why is local state risky for teams?

**Answer:** It's a local terraform.tfstate file that isn't shared or lockable; remote state (e.g. S3, Terraform Cloud) solves this

Remote state is covered in Level 2.

### [08 · A First Real Resource](level-1/08-first-real-resource.md)

**Q29.** Why start with the local provider's local_file?

**Answer:** It manages files on the machine running Terraform, so you can learn the full lifecycle with no cloud account or credentials

terraform destroy deletes the file again.

**Q30.** What is path.module?

**Answer:** The directory containing the current module's .tf files, keeping paths correct regardless of where terraform is run

Prefer it over bare relative paths.

**Q31.** How does the data "local_file" block relate to the local_file resource?

**Answer:** The resource owns the file; the data block only reads it, with an implicit dependency via the reference

It reuses module 5's distinction.

**Q32.** What does the lesson say about its AWS S3 example?

**Answer:** It isn't run against a live account; treat it as a careful read-through and verify argument names on the registry before applying

Provider argument names change between versions.

### [09 · The Core Workflow: plan, apply, destroy, fmt &amp; validate](level-1/09-plan-apply-destroy-fmt-validate.md)

**Q33.** What do +, ~, - and -/+ mean in plan output?

**Answer:** Create, update in place, destroy, and destroy-and-recreate

-/+ appears when the provider can't update an attribute in place.

**Q34.** What does `terraform plan -out=tfplan` followed by `terraform apply tfplan` guarantee?

**Answer:** The applied changes are exactly what was reviewed, with nothing drifting between plan and apply

It's the standard CI/CD pattern.

**Q35.** What does terraform validate not catch?

**Answer:** Things only the provider knows, like an invalid AMI ID, which show up at plan or apply

It checks syntax, required arguments and types locally, without touching providers or state.

**Q36.** What does the line 'Plan: 1 to add, 1 to change, 1 to destroy.' summarise?

**Answer:** The counts of resources that will be created, updated and destroyed

plan changes nothing.

**Q37.** What does terraform fmt do?

**Answer:** Rewrites .tf files in place to canonical style: two-space indentation and aligned = signs

Like gofmt or prettier, it settles style debates.

**Q38.** What does terraform validate check?

**Answer:** Internal consistency (syntax, required arguments, types) without touching any provider or state

Provider-specific problems appear at plan or apply.

**Q39.** How can fmt be used in CI?

**Answer:** terraform fmt -check fails the pipeline when files aren't formatted

It enforces one style without anyone arguing in review.

**Q40.** Why run fmt and validate before every plan?

**Answer:** They catch cheap mistakes instantly and locally before anything reaches a provider

They're built into Terraform.

### [10 · Capstone — Multi-Resource Configuration](level-1/10-capstone-multi-resource.md)

**Q41.** What does a `validation` block on a variable do?

**Answer:** Declares acceptable values with a custom error message, rejecting a typo like "prd" at plan time

The example uses contains(["dev","staging","prod"], var.environment).

**Q42.** Why does the capstone use a random_id suffix in the bucket name?

**Answer:** S3 bucket names are globally unique, so a random suffix avoids collisions

The name is built in locals as project-environment-suffix.

**Q43.** What are locals for in the capstone?

**Answer:** Computed values reused in the configuration, such as bucket_name and common_tags built with merge()

They avoid repeating expressions.

**Q44.** What does `status = var.enable_versioning ? "Enabled" : "Suspended"` demonstrate?

**Answer:** A conditional expression choosing a value from a boolean variable

It parameterises one configuration for multiple environments.

## Level 2 · Intermediate

### [01 · Modules — Structure &amp; Inputs/Outputs](level-2/01-modules-structure.md)

**Q45.** What is a module?

**Answer:** Any directory containing .tf files; the root module is one too and can call others

Modules package infrastructure as a reusable, parameterised unit.

**Q46.** Are main.tf, variables.tf and outputs.tf special filenames?

**Answer:** No: Terraform merges every .tf file in a directory; the split is a strong community convention

The convention makes modules navigable.

**Q47.** What are a module's inputs and outputs?

**Answer:** Inputs are its variables (parameters the caller sets); outputs are values it exposes to its caller

Like root-level variables and outputs, but for a called module.

**Q48.** What does `count = var.enable_versioning ? 1 : 0` do?

**Answer:** Creates the resource only when the variable is true

It's the conditional-resource idiom.

### [02 · Module Versioning](level-2/02-module-versioning.md)

**Q49.** Why pin a module version?

**Answer:** So a change to the module doesn't silently change every caller on the next terraform init

The version argument works for registry modules; Git sources pin via ref.

**Q50.** What does `~> 5.8` mean?

**Answer:** &gt;= 5.8.0 and &lt; 6.0.0 (non-breaking updates, no major bump)

~&gt; 5.8.1 allows only patch releases.

**Q51.** What does the `//` in a Git module source mean?

**Answer:** It separates the repository root from a subdirectory, e.g. tf-modules.git//s3-bucket

One repo can host many independently versioned modules.

**Q52.** Which Git pin is maximally reproducible?

**Answer:** A specific commit via ?ref=&lt;sha&gt;

Tags are good too; commits can't move.

### [03 · Remote State &amp; Backends](level-2/03-remote-state-backends.md)

**Q53.** What two problems of local state does a backend solve?

**Answer:** It isn't shared with teammates and isn't safe to commit (secrets); a backend stores it somewhere shared and lockable

It also tells Terraform where to store state and how to lock it.

**Q54.** Why does the S3 backend use a dynamodb_table?

**Answer:** S3 has no native locking primitive, so DynamoDB provides state locking

key sets the path in the bucket, so many configurations can share a bucket.

**Q55.** Why can't a backend block use variables or locals?

**Answer:** Terraform must resolve where the state lives before evaluating anything else; use literals or -backend-config at init

Partial configuration supplies values at init time.

**Q56.** What happens when you add a backend block and re-run terraform init?

**Answer:** A state migration prompt offers to copy your current state to the new backend

No resource tracking is lost.

### [04 · Workspaces](level-2/04-workspaces.md)

**Q57.** What does a workspace give you?

**Answer:** Multiple separate state files from one configuration directory, without duplicating .tf code

Every configuration starts with the default workspace.

**Q58.** What is terraform.workspace?

**Answer:** A built-in reference holding the current workspace's name as a string, so config can vary naming, sizing and tags

For example instance_type = terraform.workspace == "prod" ? ... : ...

**Q59.** Where does a non-default workspace's local state live?

**Answer:** terraform.tfstate.d/&lt;workspace&gt;/terraform.tfstate

default keeps the plain terraform.tfstate; the S3 backend namespaces per workspace automatically.

**Q60.** What is a good use of workspaces?

**Answer:** A sandbox per developer or a lightweight dev/staging split

The fuller multi-environment patterns are in Level 3.

### [05 · Provisioners (and Why to Avoid Them)](level-2/05-provisioners.md)

**Q61.** What is a provisioner?

**Answer:** A script or command run against a resource at create or destroy time, an escape hatch recommended as a last resort

remote-exec runs on the created resource; local-exec runs on the machine running Terraform.

**Q62.** What does `self` refer to inside a provisioner?

**Answer:** The resource the provisioner is attached to (e.g. self.public_ip)

It's valid only inside a provisioner block.

**Q63.** When do provisioners run?

**Answer:** Only at create or destroy, never on update; a destroy-time one runs before the resource is destroyed

Changing the script doesn't re-run it against an existing resource.

**Q64.** Why is a failed provisioner hard to handle?

**Answer:** Terraform has no visibility into what the script changed, so it can only mark the resource tainted, not retry a specific step

That's why configuration management tools or images are preferred.

### [06 · Locals &amp; Functions](level-2/06-locals-functions.md)

**Q65.** How does a local differ from a variable?

**Answer:** A local has no default, no type constraint and can't be set from outside; it's a named expression

Use locals to avoid repeating expressions.

**Q66.** Can you define your own functions in HCL?

**Answer:** No: only built-in functions exist, so fluency with the library matters

The lesson stresses this.

**Q67.** What are name_prefix and common_tags in the example?

**Answer:** Locals reused across multiple resources, such as bucket names and tags

Referenced as local.name_prefix, not var.

**Q68.** How do you reference a local?

**Answer:** local.&lt;name&gt; (singular)

Declare with a locals block (plural).

### [07 · Conditional Expressions](level-2/07-conditional-expressions.md)

**Q69.** Why is `condition ? "a" : 5` a type error?

**Answer:** Both branches must produce compatible types, because Terraform must know the result type during validation

It can't know which branch runs until values are known.

**Q70.** How do you create a resource only if a condition holds?

**Answer:** count = condition ? 1 : 0

HCL has no if/else statements, only the conditional expression.

**Q71.** How do you reference a resource that has `count`, even if it is 0 or 1?

**Answer:** With indexing, e.g. aws_cloudwatch_metric_alarm.high_cpu[0].arn

As soon as count is set, the resource is a list of instances.

**Q72.** What do try() and coalesce() do?

**Answer:** try() falls back if the first expression errors; coalesce() returns the first non-null argument

They solve 'use this value unless unavailable' without a full ternary.

### [08 · count vs for_each](level-2/08-count-for-each.md)

**Q73.** How is each instance addressed with count and with for_each?

**Answer:** count by numeric index (web[0]); for_each by key (web["web-1"])

for_each accepts a set(string) or map.

**Q74.** You remove web-2 from the middle of [web-1, web-2, web-3] used with count. What does Terraform propose?

**Answer:** Destroying and recreating index 1 (and index 2), since everything shifts, even though web-3 didn't conceptually change

Identity by position causes needless recreation.

**Q75.** Why does for_each avoid that problem?

**Answer:** A resource's identity in state is its key, so removing one key leaves the others untouched

That's why module authors default to for_each when elements have natural keys.

**Q76.** What are each.key and each.value for a set versus a map?

**Answer:** Identical for a set; distinct for a map

count.index is the 0-based index inside a count block.

### [09 · Composing Modules](level-2/09-composing-modules.md)

**Q77.** How does one module use another's output?

**Answer:** With the reference syntax module.&lt;name&gt;.&lt;output&gt;, e.g. vpc_id = module.network.vpc_id

The reference also creates a dependency.

**Q78.** In the network/app/database example, what order may app and database be created in?

**Answer:** Any order or concurrently, once network is ready; neither depends on the other

Both depend on network through references.

**Q79.** Why pass an object-typed output rather than three separate outputs?

**Answer:** It keeps calling code shorter and makes the module's 'shape' contract explicit in one type constraint

The consumer declares variable "vpc" with an object type.

**Q80.** When is depends_on on a module appropriate?

**Answer:** When ordering matters for reasons Terraform can't infer from references, such as eventual-consistency of an IAM policy attachment

Prefer implicit references.

### [10 · Capstone — Reusable Module Set](level-2/10-capstone-reusable-modules.md)

**Q81.** What does the Level 2 capstone combine?

**Answer:** A reusable networking module and app module composed from a root configuration, with conditionals, for_each and shared locals for naming and tags

It uses every Level 2 concept.

**Q82.** How was the capstone verified?

**Answer:** Reasoned through against documented Terraform and AWS provider behaviour; not run against a real cloud account

The course is honest about this.

**Q83.** Which technique repeats resources by key in the capstone?

**Answer:** for_each

It avoids positional recreation.

**Q84.** What gives consistent names and tags across modules?

**Answer:** Shared locals

Locals centralise naming and tagging.

## Level 3 · Advanced

### [01 · State Locking &amp; Team Workflows](level-3/01-state-locking-team-workflows.md)

**Q85.** What happens without state locking when Alice and Bob apply at the same time?

**Answer:** Whichever finishes last overwrites the other's changes in the state file, even though both sets of real changes happened

The state file, not reality, silently loses information.

**Q86.** What does a second concurrent apply do with locking?

**Answer:** It fails loudly with 'Error acquiring the state lock', showing who holds it and since when

The conflict is visible immediately.

**Q87.** How is locking implemented for the S3 backend?

**Answer:** With a paired DynamoDB table using conditional writes, since S3 has no native lock primitive

The local backend's file lock only protects against processes on the same machine.

**Q88.** When is terraform force-unlock appropriate?

**Answer:** Only when you're certain the process that acquired the lock crashed and won't continue

A last resort.

### [02 · Moving &amp; Importing Resources](level-3/02-moving-importing-resources.md)

**Q89.** What happens if you rename a resource in .tf without updating state?

**Answer:** Terraform plans to create the new address and destroy the old one, since state knows resources by address

terraform state mv updates the address in state so no create or destroy occurs.

**Q90.** Why prefer a moved block over terraform state mv?

**Answer:** It's committed with the code, so everyone who pulls gets the same no-recreate outcome and the migration is part of review

state mv is an imperative one-off command someone must remember.

**Q91.** What does `terraform import aws_s3_bucket.reports acme-reports-2026` do?

**Answer:** Reads the real bucket's attributes into state under that address; it does not write the .tf block for you

You must write a matching resource block or the next plan will want to destroy it.

**Q92.** Do state mv and import touch real infrastructure?

**Answer:** No: they only edit state

They keep state and configuration honest without recreate cycles.

### [03 · Terraform Cloud/Enterprise Concepts](level-3/03-terraform-cloud-enterprise.md)

**Q93.** How does Terraform Cloud differ from an S3 backend?

**Answer:** It also runs plan/apply for you on its own infrastructure, with locking, policy enforcement and run history

Terraform Enterprise is the self-hosted version.

**Q94.** What is a Terraform Cloud workspace compared with the CLI's?

**Answer:** A full unit of configuration, state, variables and run history, roughly one configuration directory

The name overlaps but the concept differs.

**Q95.** What does replacing a backend "s3" block with a cloud block and running terraform login switch?

**Answer:** State storage, locking and run execution to Terraform Cloud, so apply runs remotely

Output streams back to your terminal.

**Q96.** What's a benefit of remote runs?

**Answer:** A laptop with no cloud credentials can trigger plan/apply, and every run is auditable centrally

Credentials and variables are stored in the workspace; variable sets share them.

### [04 · Testing IaC — Plan Review](level-3/04-testing-iac-plan-review.md)

**Q97.** What does `terraform show -json tfplan` produce?

**Answer:** A structured, machine-readable plan that CI and policy engines inspect, instead of parsing terminal output

Attach the human output to PRs for reviewers.

**Q98.** What does terraform test (since 1.6) do?

**Answer:** Runs assertion-based tests defined in .tftest.hcl, entirely against plan or mocked providers, without real infrastructure

Each run block is one test case; command = plan runs only a plan.

**Q99.** What is validate's limit?

**Answer:** It's the floor, not the ceiling: it catches syntax and type errors but not whether the plan does what you intend

Combine with fmt -check, plan review and tests.

**Q100.** What does the assert block in a test contain?

**Answer:** A condition and an error_message

Example: aws_s3_bucket.this.bucket == "test-bucket".

### [05 · Policy as Code Concepts](level-3/05-policy-as-code.md)

**Q101.** What is policy as code?

**Answer:** Organisational rules written as version-controlled, automatically enforced code rather than a wiki page

For example 'no public S3 buckets' or 'instances must be tagged with a cost centre'.

**Q102.** What are the two dominant policy engines?

**Answer:** Sentinel (HashiCorp, native to Terraform Cloud/Enterprise) and OPA/Rego (via conftest on a JSON plan)

Both evaluate the structured JSON plan, not .tf source text.

**Q103.** What does the Sentinel allowlist policy do for a plan proposing t3.xlarge?

**Answer:** It fails before apply is ever permitted, regardless of who submitted it

Every aws_instance's planned instance_type must be in the allowlist.

**Q104.** How does conftest evaluate a plan?

**Answer:** terraform show -json tfplan &gt; plan.json, then conftest test plan.json against Rego deny rules

deny[msg] accumulates violation messages.

### [06 · Multi-Environment Patterns](level-3/06-multi-environment-patterns.md)

**Q105.** What does directory-per-environment give each environment?

**Answer:** A fully separate configuration with its own state, backend key and provider credentials (possibly another cloud account)

Shared logic lives once in modules/.

**Q106.** What is the risk of .tfvars-per-environment with one configuration?

**Answer:** Accidentally applying the wrong .tfvars against the wrong state if the backend or workspace selected is wrong

It suits environments sharing the same backend and account.

**Q107.** Why aren't CLI workspaces enough for dev/staging/prod?

**Answer:** They share backend configuration and credentials, while real environments need a stronger isolation boundary

Separate accounts are common.

**Q108.** What does each environment's main.tf mostly contain?

**Answer:** Different input values passed to the same shared modules

It's deliberately small.

### [07 · Dependency Management &amp; the Graph](level-3/07-dependency-graph.md)

**Q109.** How is an implicit dependency created?

**Answer:** By a reference such as vpc_id = aws_vpc.this.id, as a side effect of needing a value

It's the default and the goal.

**Q110.** When is depends_on appropriate?

**Answer:** When an ordering requirement exists that no value reference expresses, such as an IAM policy attachment that must exist first

Terraform's attribute-level inference can't see behavioural ordering.

**Q111.** What does `terraform graph | dot -Tsvg > graph.svg` do?

**Answer:** Renders the DOT-format dependency graph as a diagram, useful to confirm a dependency does or doesn't exist

Large graphs are too dense to read directly.

**Q112.** What can run concurrently?

**Answer:** Resources with no dependency path between them, e.g. subnets that depend only on the VPC

The plan output shows parallelism once the VPC is created.

### [08 · Secrets Management Patterns](level-3/08-secrets-management.md)

**Q113.** Why is hard-coding password = "hunter2" in a .tf file bad even if removed later?

**Answer:** It stays in Git history, readable by anyone with repo access

Never commit secrets.

**Q114.** What does `sensitive = true` on a variable do?

**Answer:** Hides the value in plan and apply output (but it can still be stored in state)

State may still contain the value, so protect state.

**Q115.** Which two places should secrets stay out of as much as possible?

**Answer:** Version control and state

For residual cases limit exposure with encrypted, access-controlled remote state.

**Q116.** What pattern supplies secrets without writing them in code?

**Answer:** Read them from a secrets manager or environment variables at runtime

The lesson covers several such patterns.

### [09 · Refactoring State Safely](level-3/09-refactoring-state-safely.md)

**Q117.** What do moved blocks accomplish when extracting resources into a module?

**Answer:** The plan shows zero changes for them instead of destroy 2 / create 2

Many resource types can't be recreated without downtime or data loss.

**Q118.** How do you split one state into two configurations?

**Answer:** terraform state mv -state-out=extracted.tfstate ... then terraform state push in the destination configuration

It's rarer and higher-stakes than a moved refactor, so back up state first.

**Q119.** Why is a moved refactor safer than recreate?

**Answer:** Real infrastructure isn't touched; only addresses in state change

Same resources, new addresses.

**Q120.** When can directory-per-domain splitting be used?

**Answer:** To split by domain (networking, app), not just by environment

It physically moves state entries between backends.

### [10 · Capstone — Multi-Environment Platform](level-3/10-capstone-multi-environment.md)

**Q121.** How do dev and prod avoid blocking each other's locks?

**Answer:** Same bucket and lock table but disjoint state keys (platform/dev vs platform/prod)

Each environment's state is independently locked and encrypted.

**Q122.** What does each environment's main.tf do?

**Answer:** Calls the same network and app modules with different inputs (AZ count, instance count, monitoring)

Prod uses 3 AZs, 4 instances, monitoring on; dev uses 2, 1, off.

**Q123.** Which parts does the capstone combine?

**Answer:** Directory-per-env, locked remote state, a policy check and a terraform test suite

It builds around the Level 2 modules.

**Q124.** How was the capstone verified?

**Answer:** Reasoned against documented behaviour; not run against a real cloud account

The course is honest about this.

## Level 4 · Master

### [01 · Large-Scale Module Design](level-4/01-large-scale-module-design.md)

**Q125.** What makes a module worth reuse at scale?

**Answer:** It encapsulates a decision (how the org does encrypted, replicated storage), not just a resource type

Thin wrappers add indirection with no value.

**Q126.** What should reusable modules know about teams or environments?

**Answer:** Nothing: environment-specific decisions belong in the composition root

A hard-coded team name or account ID makes callers work around or fork the module.

**Q127.** Why group related arguments in one object variable (e.g. encryption)?

**Answer:** It keeps the interface stable and explicit, like an API contract, with optional fields and defaults

Variables and outputs are a module's API.

**Q128.** What is a composition root?

**Answer:** An opinionated configuration that wires several reusable modules for one team's stack

For example platforms/data-team.

### [02 · CI/CD for Terraform](level-4/02-cicd-for-terraform.md)

**Q129.** What is the two-workflow pattern?

**Answer:** Plan on pull request (commented on the PR) and apply on merge to main

It runs fmt -check, validate and plan on PRs touching envs/prod or modules.

**Q130.** What does `environment: production` on the apply job do?

**Answer:** Ties it to a GitHub Environment with required reviewers, so it pauses for human approval before terraform apply runs

It's a controlled deployment gate.

**Q131.** Why attach the plan to the PR automatically?

**Answer:** It can never be forgotten or run against a stale checkout

Reviewers see exactly what will change.

**Q132.** Why pin terraform_version in setup-terraform?

**Answer:** Every run uses the same CLI version, avoiding behaviour differences

Reproducible pipelines.

### [03 · Drift Detection &amp; Remediation](level-4/03-drift-detection-remediation.md)

**Q133.** What is drift?

**Answer:** Real infrastructure no longer matching what Terraform's state believes (console edits, autoscaling, other tools)

Detect it systematically.

**Q134.** What does `terraform plan -detailed-exitcode` return when there's a diff?

**Answer:** Exit code 2 (0 means in sync, 1 means error)

A scheduled job can fail only when something needs attention.

**Q135.** How does Terraform label detected drift in a plan?

**Answer:** 'changes made outside of Terraform since the last apply', distinct from changes your own .tf edits caused

For example a security group rule widened to 0.0.0.0/0 in the console.

**Q136.** When is the drift-detection job scheduled in the lesson?

**Answer:** Daily by cron across dev, staging and prod via a matrix

It exits 1 when exit code 2 is detected to trigger normal CI alerts.

### [04 · Cost Estimation Concepts](level-4/04-cost-estimation-concepts.md)

**Q137.** Why doesn't Terraform Core know what anything costs?

**Answer:** Provider schemas describe shape, not price; pricing lives in separate billing/price-list data

instance_type is just a string to the provider schema.

**Q138.** What do cost tools like Infracost build on?

**Answer:** terraform show -json output: resource_changes with planned attribute values (change.after)

They cross-reference attributes like instance_type and volume_size against a price catalog.

**Q139.** Why do cost estimators go stale?

**Answer:** They need a per-resource-type mapping, which lags when providers add new resource types or attributes

There are also limits to estimating cost from configuration alone.

**Q140.** What does -out=tfplan save?

**Answer:** The plan in Terraform's binary plan format

show -json converts it to documented JSON.

### [05 · Security Scanning for IaC](level-4/05-security-scanning-iac.md)

**Q141.** Why does terraform validate pass a bucket with all public-access blocks set to false?

**Answer:** It checks types, required arguments and references; schema validity and security posture are orthogonal

Provider schemas don't know that false is dangerous.

**Q142.** What do IaC security scanners do?

**Answer:** Statically analyse configuration (or the plan) for risky patterns like public buckets before apply

They reason about your configuration to catch misconfiguration.

**Q143.** When should scanning run?

**Answer:** Before apply, such as in CI on pull requests

Shift security left.

**Q144.** A world-readable bucket is perfectly valid HCL. What does that imply?

**Answer:** Valid syntax doesn't mean safe, so security checks need separate tooling

It's as much a bug as a syntax error.

### [06 · Designing a Platform's Terraform Architecture](level-4/06-platform-architecture.md)

**Q145.** What is the core tension in platform Terraform architecture?

**Answer:** Blast radius versus duplication/coordination: one big state is simple but a lock or bad plan affects everyone; many states shrink blast radius but need cross-state contracts

Choose where to draw the boundaries deliberately.

**Q146.** How do you read another state's values across a boundary?

**Answer:** A terraform_remote_state data source reading the other state's outputs (or an explicit output/input contract)

module.network.vpc_id stops being a local reference.

**Q147.** What may the database configuration depend on from the network state?

**Answer:** Only its outputs, not internal resource addresses or module structure

Outputs are the contract.

**Q148.** What is the downside of one giant state with all modules?

**Answer:** A lock held by anyone blocks everyone, and a bad plan anywhere risks touching unrelated resources

The upside is local references.

### [07 · Governance at Scale](level-4/07-governance-at-scale.md)

**Q149.** Why isn't per-team IAM enough for governance?

**Answer:** IAM bounds what a pipeline can do but can't express configuration rules like 'no S3 bucket without encryption'; policy as code fills that gap

You need both layers.

**Q150.** What do hard-mandatory and soft-mandatory enforcement levels mean?

**Answer:** hard-mandatory fails the run with no override; soft-mandatory can be overridden by someone with permission, for legitimate exceptions

There's also advisory.

**Q151.** Why centralise one policy set attached to many workspaces?

**Answer:** The platform team writes and versions it once, attached by tag, instead of every team copying policy logic into pipelines

For example a production tag.

**Q152.** What is governance at scale about?

**Answer:** Who may change what, which rules apply to which workloads, and enforcing it without a platform team reviewing every plan by hand

It sits on top of policy-as-code.

### [08 · Disaster Recovery for State &amp; Infra](level-4/08-disaster-recovery.md)

**Q153.** If a state file is lost, what happens to the real infrastructure?

**Answer:** It keeps running, but the next plan sees empty state and proposes creating everything, which fails loudly or duplicates resources

State, not .tf files, is the disaster-recovery-critical asset.

**Q154.** What does S3 versioning on the state bucket give you?

**Answer:** Recovery from an accidental delete or bad overwrite by promoting a previous version

It's not optional hardening for production.

**Q155.** What does cross-region replication protect against?

**Answer:** Losing the entire bucket or region, not just one object version

The example replicates to a DR region.

**Q156.** Why are .tf files easy to recover?

**Answer:** They're text in version control

The state file is the single point of failure.

### [09 · Observability for Infrastructure Changes](level-4/09-observability-infra-changes.md)

**Q157.** Why isn't apply output alone observability?

**Answer:** It scrolls past in a CI log that may expire and isn't searchable as a timeline

You need to retain the plan diff, resource events and approval metadata.

**Q158.** What does TF_LOG_JSON=1 do?

**Answer:** Emits structured JSON lines with stable schemas so a log shipper can parse resource.addr, action and elapsed_seconds without regex

It turns log output into log data.

**Q159.** Why retain the JSON plan as well as the apply log?

**Answer:** The apply log shows what happened but not why: what the planned diff was

Store plan JSON in an audit bucket.

**Q160.** Which three things should every apply export?

**Answer:** The plan diff, the resource-level events and identity/approval metadata

Into systems built to retain and query them.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q161.** What does the Level 4 capstone deliver?

**Answer:** A platform-shaped Terraform layout with CI/CD, drift detection, cost estimation, security scanning and governance in one pipeline, plus a written DR and observability plan

It introduces no new mechanisms.

**Q162.** How does the capstone relate to modules 01–09?

**Answer:** Every piece is a direct application of them

It assembles them.

**Q163.** What does the written plan cover?

**Answer:** Disaster recovery and observability

Alongside the code.

**Q164.** What is a service in the scenario?

**Answer:** An internal API on a new team's platform, with CI/CD gating changes

The pipeline wires policy, security and cost checks.

## Mixed questions across all levels

**Q165.** What does this configuration do when it is applied?

```hcl
variable "enable_monitoring" {
  type    = bool
  default = false
}
resource "aws_cloudwatch_metric_alarm" "cpu" {
  count = var.enable_monitoring ? 1 : 0
}
```

**Answer:** Creates no alarm by default; with enable_monitoring = true it creates exactly one, addressed as ...cpu[0]

count = condition ? 1 : 0 is the conditional-resource idiom, and a resource with count is addressed by index.

**Q166.** `condition ? "a" : 5` fails validation. Why?

**Answer:** Both branches must produce compatible types so Terraform knows the result type before it knows which branch runs

HCL has no if/else statement, only the conditional expression.

**Q167.** You use `count` over ["web-1", "web-2", "web-3"] and remove "web-2". What happens to the instances?

**Answer:** Everything after the removed index shifts: web[1] now means web-3 and web[2] disappears, causing needless changes; for_each keyed by name touches only web-2

Identity by position versus by key. Verified with a small simulation: count gives an update to web[1] and a destroy of web[2]; for_each gives one destroy.

**Q168.** Which type fits `for_each` over instance names?

**Answer:** A set(string) or a map, addressed as aws_instance.web["web-1"]

for_each accepts a set of strings or a map; each.key and each.value are available.

**Q169.** A teammate renamed `aws_instance.web` to `aws_instance.app_server` in code, and the plan wants to destroy and create. What is the safest fix?

**Answer:** Add a moved { from = aws_instance.web to = aws_instance.app_server } block (or terraform state mv)

State knows resources by address, so a rename looks like a different resource. A moved block makes the migration part of code review.

**Q170.** You imported an existing bucket with `terraform import aws_s3_bucket.reports acme-reports-2026` but haven't written the resource block. What will the next plan do?

**Answer:** Propose to destroy it, because import records state but doesn't write configuration; write the matching resource block first

Import writes to state only.

**Q171.** Two engineers run `terraform apply` simultaneously on the same S3 backend with DynamoDB locking. What happens to the second run?

**Answer:** It fails with 'Error acquiring the state lock' and shows who holds it

Without a lock, the last writer would overwrite the other's state changes.

**Q172.** Which backend settings differ between dev and prod so their locks can't block each other?

**Answer:** The state key (e.g. platform/dev/terraform.tfstate vs platform/prod/terraform.tfstate)

Same bucket and lock table, different keys, give independent state and locks.

**Q173.** Why can't a backend block use variables?

**Answer:** Terraform must resolve where state lives before evaluating anything else; use literals or -backend-config

Partial configuration supplies the values at init time.

**Q174.** Your CI wants machine-readable plan output for a policy engine. Which commands?

**Answer:** terraform plan -out=tfplan then terraform show -json tfplan

Sentinel and OPA/conftest evaluate the JSON plan (resource_changes[].change.after), not your .tf text.

**Q175.** A plan proposes `t3.xlarge` but your policy allows only t3.micro/small/medium. Which layer should block it before apply?

**Answer:** A policy check (Sentinel or OPA/conftest) on the plan, in addition to least-privilege IAM

IAM bounds what a pipeline can do; policy bounds what it may configure. Verified with a small policy checker over a JSON plan.

**Q176.** A security group rule was widened to 0.0.0.0/0 in the console. How do scheduled pipelines detect it?

**Answer:** terraform plan -detailed-exitcode returns 2 when there's a diff, which fails the job

Exit 0 means in sync, 1 means error, 2 means changes.

**Q177.** A bucket has all four public_access_block settings set to false. What does `terraform validate` say?

**Answer:** Valid: it checks syntax, types and references, not security posture; use an IaC scanner or policy

Schema validity and security posture are orthogonal.

**Q178.** A secret password is hard-coded in a .tf file and later removed in a new commit. Is it safe now?

**Answer:** No: it remains in Git history; rotate it, and use sensitive variables or a secrets manager (state can still hold values)

Treat state as sensitive too.

**Q179.** You lost terraform.tfstate but the cloud resources still exist. What will the next plan propose?

**Answer:** Creating everything from scratch (failing on collisions or duplicating); restore versioned state from the S3 bucket instead

State loss doesn't touch real resources; versioning and replication on the state bucket are the first lines of defence.

**Q180.** Which module design rule keeps a module reusable at scale?

**Answer:** No knowledge of specific teams, environments or accounts; those decisions live in the composition root

A hard-coded team or account ID makes callers fork the module.

**Q181.** Which version constraint accepts 5.9.2 but not 6.0.0?

**Answer:** ~&gt; 5.8 (&gt;= 5.8.0, &lt; 6.0.0)

~&gt; 5.8.1 allows only patch releases (&lt; 5.9.0).

**Q182.** When is `depends_on` justified?

**Answer:** When ordering matters but no value is referenced, such as an IAM policy attachment that must exist first

References already create implicit dependencies, which are preferred.

**Q183.** Why does a Terraform plan not show what a change will cost?

**Answer:** Provider schemas describe shape, not price; cost tools read the JSON plan and look up a separate price catalog

Infracost-style tools walk resource_changes[].change.after.

## Practical coding questions

### Why count shifts and for_each doesn't

Write the two address builders: addresses_count(names) maps each position to an address like aws_instance.web[0], and addresses_for_each(names) maps each name to aws_instance.web["web-1"]. The provided plan() compares before and after to show create, update and destroy. Print the plan for removing "web-2" from the middle of the list.

Starter code:

```
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

Expected output:

```text
count: {'create': [], 'update': ['aws_instance.web[1]'], 'destroy': ['aws_instance.web[2]']}
for_each: {'create': [], 'update': [], 'destroy': ['aws_instance.web["web-2"]']}
```

??? success "Show a solution"

    ```
    def addresses_count(names):
        # count addresses instances by position: web[0], web[1], ...
        return {f"aws_instance.web[{i}]": n for i, n in enumerate(names)}

    def addresses_for_each(names):
        # for_each addresses instances by key
        return {f'aws_instance.web["{n}"]': n for n in names}

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

    With count, identity is position, so removing the middle element shifts web-3 into index 1 (an update, or a replace for attributes that force one) and drops index 2. With for_each, identity is the key, so only web-2 is removed. Verified by running the solution.

### A plan policy check

Write violations(plan) that walks resource_changes and reports, for every aws_instance being created or updated (skip deletes where change.after is null), an instance_type outside the allowlist and any missing required tags. This is the same idea as a Sentinel or OPA policy over terraform show -json.

Starter code:

```
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

Expected output:

```text
FAIL aws_instance.web: instance_type t3.xlarge not allowed
FAIL aws_instance.api: missing tags ['Environment']
2 violations
```

??? success "Show a solution"

    ```
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
        for rc in plan["resource_changes"]:
            after = rc["change"]["after"]
            if rc["type"] != "aws_instance" or after is None:
                continue
            if after["instance_type"] not in ALLOWED:
                out.append(f"{rc['address']}: instance_type {after['instance_type']} not allowed")
            missing = REQUIRED_TAGS - set(after.get("tags", {}))
            if missing:
                out.append(f"{rc['address']}: missing tags {sorted(missing)}")
        return out

    for v in violations(PLAN):
        print("FAIL", v)
    print("passed" if not violations(PLAN) else f"{len(violations(PLAN))} violations")
    ```

    Policies are queries over the JSON plan's planned values (change.after), not over .tf text, which is why one policy can cover every environment. Verified by running the solution.

---

More free practice: the [Terraform Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
