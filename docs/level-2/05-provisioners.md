# 05 · Provisioners (and Why to Avoid Them)

A **provisioner** runs a script or command against a resource at
create-time (or destroy-time) — the escape hatch for configuration a
provider's API can't express directly. HashiCorp's own documentation
recommends treating them as a last resort, and this module explains both
how they work and why that warning exists.

## `remote-exec`: run commands on the created resource

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcd1234"
  instance_type = "t3.micro"
  key_name      = "deploy-key"

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
    ]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("~/.ssh/deploy-key.pem")
      host        = self.public_ip
    }
  }
}
```

`self.public_ip` refers to *this same resource's* own computed attribute —
`self` is only valid inside a provisioner block, standing in for the
resource the provisioner is attached to.

## `local-exec`: run a command on the machine running Terraform

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcd1234"
  instance_type = "t3.micro"

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> inventory.txt"
  }
}
```

Unlike `remote-exec`, this never touches the created resource itself — it
runs entirely on whatever machine is executing `terraform apply`, which is
why it needs no `connection` block.

## `when = destroy`: cleanup provisioners

```hcl
resource "aws_instance" "web" {
  # ...

  provisioner "local-exec" {
    when    = destroy
    command = "echo 'destroying ${self.id}' >> destroy-log.txt"
  }
}
```

A destroy-time provisioner runs *before* the resource is actually
destroyed, and only if the resource successfully existed in state —
useful for deregistering a node from a load balancer or service registry
before Terraform tears down the instance underneath it.

## Why provisioners are a last resort

1. **No dependency graph awareness of what's inside the script.** Terraform
   tracks the *resource* as a graph node, but has zero visibility into
   what `apt-get install nginx` actually changed — if it partially fails,
   Terraform can only report the resource as tainted, not diagnose or
   retry the specific failed step.
2. **They run only at create or destroy, never on update.** Changing the
   `inline` script list doesn't re-run it against an already-created
   instance — provisioners are one-shot, tied to the resource's lifecycle
   transition, not to configuration drift detection the way normal
   resource attributes are.
3. **A failed provisioner marks the resource "tainted."** The resource
   *was* created (or destroyed) successfully at the API level, but
   Terraform now considers its state unreliable and will destroy and
   recreate it on the next `apply` — which can be far more disruptive than
   the original provisioning failure.
4. **Better alternatives usually exist**: provider-native mechanisms
   (`user_data` for EC2, which the instance itself executes on first boot
   without any SSH connection from Terraform's process at all), a
   purpose-built image (baked with Packer so the instance never needs
   post-create configuration), or a dedicated configuration-management
   tool (Ansible) run as a separate, retryable step after `apply`.

## Worked example: preferring `user_data` over `remote-exec`

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcd1234"
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl enable --now nginx
  EOF
}
```

Same outcome as the `remote-exec` example above, but `user_data` is a
plain resource attribute the provider passes straight to the cloud API at
launch — no SSH connection Terraform needs to establish or firewall rule
Terraform needs punched open, no dependency on the instance being
network-reachable from wherever `apply` runs, and it re-applies
automatically to any new instance created from the same configuration
(including one recreated after a taint) without a distinct provisioner
step at all.

## How It Actually Works

**Provisioners execute entirely outside the provider-RPC model this course
otherwise relies on.** Every other piece of Terraform's execution
described so far — `PlanResourceChange`, `ApplyResourceChange` — is a
structured, typed RPC call between Terraform Core and a provider plugin,
whose result Terraform can inspect and record precisely. A provisioner is,
by contrast, Terraform Core literally opening an SSH/WinRM connection (for
`remote-exec`) or forking a subprocess (for `local-exec`) and streaming raw
stdout/stderr — success or failure is just the process's exit code, with
none of the structured "here's exactly which attribute changed" detail a
provider RPC returns.

**Taint is the mechanism that keeps a failed provisioner from silently
lying about resource state.** When a `create`-time provisioner fails after
the underlying `ApplyResourceChange` already succeeded, Terraform Core
can't roll back the already-created resource (it has no generic "undo"
call), so instead it writes the resource into state with an internal
"tainted" flag. The *next* `plan` sees that flag and — regardless of
whether the configuration changed at all — proposes destroying and
recreating that specific resource, because a tainted resource is
considered to be in an unknown, unverified state rather than a known-bad
one Terraform could otherwise reason about.

**`self` resolves via the same node the provisioner is attached to in the
graph, evaluated only once the resource's own apply has produced values.**
Provisioners are graph-walked as a sub-step of their owning resource's
apply node — they run strictly after that resource's `ApplyResourceChange`
returns, which is precisely why `self.public_ip` is guaranteed to be a
real, non-unknown value by the time the provisioner block executes:
provisioners are ordered dependents of their own resource, not independent
graph nodes that could race ahead of it.

## Exercise

Rewrite the `remote-exec` example at the top of this page (installing
nginx over SSH) as a `user_data`-based `aws_instance` instead, and write
one sentence explaining why the `user_data` version has no `connection`
block and needs no SSH key at all.
