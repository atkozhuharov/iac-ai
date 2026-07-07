# IaC AI tools and skills
This is a public repo with skills, instructions, tools and tips I use with my agents for IaC

# Skills
Drop-in [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills) for Terraform/OpenTofu workflows. Each lives under [`skills/`](skills/).

| Skill | What it does |
| --- | --- |
| [terraform-style](skills/terraform-style/SKILL.md) | Style guide for writing Terraform/OpenTofu code — provider/version constraints first, resources in dependency order, and consistent formatting conventions. |
| [iac-module-scaffold](skills/iac-module-scaffold/SKILL.md) | Scaffolds a registry-ready module: standard file layout, typed/validated/described variables, documented outputs, an `examples/` usage, native `.tftest.hcl` tests, and a terraform-docs README — validates clean out of the box. |
| [terraform-cloud](skills/terraform-cloud/SKILL.md) | Triggers Terraform Cloud / TFE remote runs and polls their status and plan/apply logs over the API with `curl`. Works against `app.terraform.io` and any TFE-compatible host (e.g. InfraDots) via `TFC_HOST`. |

# Additional tools
1. [rtk](https://github.com/rtk-ai/rtk) for lowering token usage
