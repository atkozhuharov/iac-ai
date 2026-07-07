---
name: iac-module-scaffold
description: >-
  Scaffold a well-structured, registry-ready Terraform/OpenTofu module — standard
  file layout (main/variables/outputs/versions), typed+validated+described
  variables, documented outputs, an examples/ usage, native .tftest.hcl tests, a
  terraform-docs README, and a fmt/validate check. Use when the user wants to
  create a new module, restructure an existing one to conventions, or add
  examples/tests/docs to a module.
---

# Terraform / OpenTofu module scaffold

Generate a module that matches Terraform Registry conventions and passes
`fmt`/`validate` out of the box. Works for either binary — detect once:

```bash
TF_BIN="$(command -v tofu || command -v terraform)"
```

## Standard layout

```
modules/<name>/            # or a repo named terraform-<PROVIDER>-<NAME> to publish
├── main.tf                # resources (+ locals/data; split into locals.tf/data.tf if large)
├── variables.tf           # all inputs — typed, described, validated
├── outputs.tf             # all outputs — described
├── versions.tf            # required_version + required_providers (pinned)
├── README.md              # terraform-docs injects inputs/outputs here
├── .terraform-docs.yml    # doc config
├── examples/
│   └── basic/             # a runnable usage — this is also what tests exercise
│       ├── main.tf
│       └── outputs.tf
└── tests/
    └── basic.tftest.hcl   # native `terraform test` / `tofu test`
```

Rules: **one root of resources per module**, no `provider` blocks inside the
module (callers configure providers), no hardcoded backend. Keep the surface
small — expose inputs, don't leak implementation.

## File templates

**versions.tf** — pin the language and every provider:

```hcl
terraform {
  required_version = ">= 1.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"     # resolves per-registry (terraform.io / opentofu.org)
      version = ">= 5.0, < 6.0"
    }
  }
}
```

**variables.tf** — every variable gets `description` + `type`; add `default` only
when genuinely optional, `validation` when the domain is constrained, `sensitive`
for secrets, `nullable = false` when null is invalid:

```hcl
variable "name" {
  description = "Name prefix for resources created by this module."
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,30}$", var.name))
    error_message = "name must be lowercase alphanumeric/hyphens, 2-31 chars."
  }
}

variable "instance_count" {
  description = "Number of instances to create."
  type        = number
  default     = 1

  validation {
    condition     = var.instance_count > 0
    error_message = "instance_count must be greater than 0."
  }
}

variable "tags" {
  description = "Tags applied to all resources."
  type        = map(string)
  default     = {}
}
```

**main.tf** — reference the canonical resource as `this`; merge module-level tags:

```hcl
locals {
  tags = merge(var.tags, { "managed-by" = "terraform", "module" = "name" })
}

resource "aws_example" "this" {
  name  = var.name
  count = var.instance_count
  tags  = local.tags
}
```

**outputs.tf** — describe every output; mark sensitive ones:

```hcl
output "ids" {
  description = "IDs of the created instances."
  value       = aws_example.this[*].id
}
```

**examples/basic/main.tf** — a real caller (doubles as the test fixture):

```hcl
module "example" {
  source = "../.."
  name   = "example-basic"
}

output "ids" {
  value = module.example.ids
}
```

## Tests — native `.tftest.hcl`

`tests/basic.tftest.hcl` (Terraform 1.6+ / OpenTofu). `command = plan` is fast
and needs no credentials; use `command = apply` for real end-to-end where cheap.

```hcl
run "validates_inputs" {
  command = plan

  variables {
    name           = "unit-test"
    instance_count = 2
  }

  assert {
    condition     = length(aws_example.this) == 2
    error_message = "expected 2 instances"
  }
}

run "rejects_bad_name" {
  command = plan
  variables { name = "Bad_Name" }

  expect_failures = [var.name]   # validation should reject this
}
```

Run:

```bash
$TF_BIN test
```

## README via terraform-docs

`.terraform-docs.yml`:

```yaml
formatter: markdown table
output:
  file: README.md
  mode: inject          # writes between the BEGIN/END markers, leaves prose intact
sort:
  enabled: true
  by: required
```

Seed `README.md` with the injection markers, then generate:

```markdown
# <module name>

Short description. Usage:

```hcl
module "example" {
  source = "..."
  name   = "my-name"
}
```

<!-- BEGIN_TF_DOCS -->
<!-- END_TF_DOCS -->
```

```bash
terraform-docs .          # reads .terraform-docs.yml, injects inputs/outputs
```

(If `terraform-docs` isn't installed: `brew install terraform-docs`. If skipping
it, still hand-write the Inputs/Outputs tables.)

## Verify before done

```bash
$TF_BIN fmt -recursive
$TF_BIN init -backend=false     # no backend needed to validate a module
$TF_BIN validate
$TF_BIN test                    # if tests were scaffolded
terraform-docs .                # keep README in sync
```

All four should pass/clean. `validate` catches type and reference errors without
touching real infra.

## Conventions / gotchas

- **Publishable name**: registry modules live in a repo `terraform-<PROVIDER>-<NAME>`
  (e.g. `terraform-aws-vpc`); the module name is derived from it. Local modules
  under `modules/<name>/` don't need this.
- **No providers/backends inside the module** — callers own those. A module that
  declares its own `provider` block can't be reused with aliased providers.
- **Pin providers** in `versions.tf` with a range (`>= x, < y`), not a floating
  or exact pin — reproducible yet patch-updatable.
- **Describe everything.** Undocumented variables/outputs produce an empty
  terraform-docs table and a module nobody can use safely.
- **`this` naming** for the module's primary resource is the community idiom;
  it makes references and outputs predictable.
- Same layout serves Terraform and OpenTofu unchanged — `.tftest.hcl` and
  `terraform-docs` both work with `tofu`.
