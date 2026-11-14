# IaC AI tools and skills
This is a public repo with skills, instructions, tools and tips I use with my agents for IaC

# Skills
Drop-in [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills) for Terraform/OpenTofu workflows. Each lives under [`skills/`](skills/).

| Skill | What it does |
| --- | --- |
| [terraform-style](skills/terraform-style/SKILL.md) | Style guide for writing Terraform/OpenTofu code — provider/version constraints first, resources in dependency order, and consistent formatting conventions. |
| [iac-module-scaffold](skills/iac-module-scaffold/SKILL.md) | Scaffolds a registry-ready module: standard file layout, typed/validated/described variables, documented outputs, an `examples/` usage, native `.tftest.hcl` tests, and a terraform-docs README — validates clean out of the box. |
| [terraform-cloud](skills/terraform-cloud/SKILL.md) | Triggers Terraform Cloud / TFE remote runs and polls their status and plan/apply logs over the API with `curl`. Works against `app.terraform.io` and any TFE-compatible host (e.g. InfraDots) via `TFC_HOST`. |
| [iac-security-review](skills/iac-security-review/SKILL.md) | Security review of Terraform/OpenTofu/Terragrunt changes and the Kubernetes, Helm and CI config they deploy: identity, network exposure, secrets, storage, state and supply chain. Requires a concrete boundary and result per finding, marks unseen facts as needs-validation, and reads its three companion files only when the change touches their domain. Adapted from [Cloudflare's security-audit skill](https://github.com/cloudflare/security-audit-skill) (MIT). |

## Use with InfraDots
The repository is also an [Agent Plugins](https://agent-plugins.org) package (`plugin.json` at the root), so an InfraDots organization can install it and its agents load the skills on demand.

# Additional tools
1. [rtk](https://github.com/rtk-ai/rtk) for lowering token usage
