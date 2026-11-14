---
name: iac-security-review
description: >-
  Security review of Terraform, OpenTofu and Terragrunt changes and the Kubernetes, Helm and CI configuration
  they deploy: IAM and workload identity, network exposure, secrets, storage, backups and state, and module or
  provider supply chain. Use when reviewing an infrastructure change or plan for security, or when asked whether
  one is safe. Not for application code.
license: MIT
metadata:
  adapted-from: "https://github.com/cloudflare/security-audit-skill (MIT) at c1c8a8c14710"
---

# IaC Security Review

Find changes that let a lower-trust principal cross a real trust boundary, then give the owner the evidence,
a priority and the smallest configuration change that closes it. A candidate without a concrete affected
principal, resource and outcome is not a finding.

This is review guidance, not an audit workflow. Read the change, the configuration around it and the plan;
write no files, run nothing from the repository, and never probe a live endpoint, account or cluster. When the
deciding fact lives outside what you can read, say exactly which fact is missing instead of guessing.

## What to read

- **The diff** — what the author changed, and the resources that read what changed.
- **The plan** — each resource's `before`/`after` and its actions. `replace` and `delete` on stateful resources,
  and an `after` that widens a policy or a CIDR, matter more than the diff's line count.
- **The configuration around it** — variable defaults, `tfvars`, module inputs, provider blocks, and the
  environment overlays (workspaces, Terragrunt `inputs`, Helm values) that decide what is actually deployed.

Read a companion file only when the change touches its domain:

| The change touches | Read |
|---|---|
| IAM roles and policies, trust policies, workload identity, security groups, firewalls, load balancers, ingress, Kubernetes RBAC, pod security, Helm values | [CLOUD-AND-DEPLOYMENT.md](CLOUD-AND-DEPLOYMENT.md) |
| Module or provider sources and versions, lock files, CI workflows and their cloud credentials (OIDC trust), images, registries | [SUPPLY-CHAIN-AND-RELEASE.md](SUPPLY-CHAIN-AND-RELEASE.md) |
| Buckets, databases, snapshots, backups, replication, KMS keys, log sinks, lifecycle rules, Terraform state and its backend | [DATA-ISOLATION-AND-LIFECYCLE.md](DATA-ISOLATION-AND-LIFECYCLE.md) |

## Where infrastructure changes usually go wrong

Each is a place to look, not a finding by itself:

- **Identity** — `*` actions or resources, a trust policy without an `ExternalId`, source-account or OIDC `sub`
  condition, a role a workload can assume that reaches other accounts or tenants, broad Kubernetes bindings.
- **Reachability** — `0.0.0.0/0` or `::/0` to an admin, database, SSH/RDP, metrics or control-plane port; a public
  IP, listener or ingress in front of something internal; a private endpoint or firewall rule removed.
- **Secrets** — a credential in a variable default, `tfvars`, `user_data`, a container env, an output not marked
  `sensitive`, or a resource argument that lands in plaintext state.
- **Data** — public access blocks, bucket or key policies, encryption, versioning or deletion protection turned
  off on data that holds something; a snapshot, replica or backup shared more widely than its source.
- **Audit** — logging, flow logs or audit trails disabled or redirected where the change also widens access.
- **Supply chain** — a module or provider moved to an unpinned ref, an unknown source, or a fork; a CI job that
  runs untrusted code with cloud credentials.

## Require a boundary and a result

For every candidate, name:

1. the lower-trust principal (an internet client, another tenant or account, a workload, a CI job from a fork);
2. what it can now do or reach, through which resource address;
3. the control that was supposed to stop it, and why it doesn't;
4. the affected resource or data, and the concrete result.

A missing best practice, a guessed provider default or a deployment you can't see is not a finding. Enabling
versioning, adding tags or turning on a scanner's favourite flag is hardening: mention it only if asked.

## Respect what the source can show

Account policies, organization SCPs, network attachments, admission controllers, CI branch protection and the
values a pipeline injects are real controls. If one decides the outcome and it isn't in what you can read, do
not assume it is present or absent: mark the finding **needs validation** and name the exact fact an owner must
check (for example "whether the `prod` account's SCP denies `s3:PutBucketPolicy` with a public principal").

## Severity

Only a confirmed finding gets a severity, and it cannot exceed the demonstrated result.

- **critical** — an unauthenticated actor gets code execution, full data-store access, or administrative cloud
  access (an `AdministratorAccess` role assumable from any account, a public database with no auth).
- **high** — an explicit control is fully defeated with real consequences: a public bucket holding customer data,
  cross-account or cross-tenant read or write, SSH open to the internet on a host with credentials, a CI job from
  a fork that can deploy.
- **medium** — a real boundary violation with a limited blast radius or uncommon preconditions.
- **low** — disclosure of non-secret internals, or an effect needing sustained effort for little gain.
- **informational** — confirmed but minimal on its own; useful as a step in a larger finding.

The high/medium test: does the change fully defeat an explicit control for an action with real consequences, or
only weaken it? If you can't state the damage, the severity is lower than it feels.

When the reviewer has its own scale, map onto it: InfraDots reviews use critical and high → **blocker**,
medium → **warning**, low → **info**, informational → **nit**; a needs-validation finding is **info** and
names the missing fact.

## Recommend the smallest fix

State the invariant the configuration must enforce and the narrowest change that enforces it at the resource
that makes the decision: a condition on the trust policy, a CIDR narrowed to the load balancer's security group,
`sensitive = true` on the output, a pinned module ref. Give the resource address and the file. Prefer one exact
change over a list of hardening advice.

## Reporting a finding

For each one: a title naming the boundary, severity or **needs validation**, the resource address and
`file:line`, the principal → resource → result chain, the evidence (the plan's `after`, the policy statement,
the variable that sets it), and the fix. Keep findings that share a root cause together.

## Anti-patterns

1. Scanner checklist items presented as vulnerabilities.
2. Defense-in-depth advice with no reachable boundary violation.
3. Guessing provider, account, network or cluster behavior that isn't in the configuration.
4. Treating a secret *reference* (a Secrets Manager ARN, a `data` lookup, a key ID) as a disclosed secret.
5. Reporting an effect stronger than the change produces, or one that only the same principal can trigger.
6. Severity on a needs-validation finding.
7. Probing, applying or running anything to "check".

---

Adapted from Cloudflare's [security-audit skill](https://github.com/cloudflare/security-audit-skill)
(`skills/security-audit`, commit `c1c8a8c14710`), © 2025-2026 Cloudflare, Inc., MIT; see [LICENSE](LICENSE).
The full-audit workflow (agent waves, run artifacts, validators, sandboxed execution) is left out; the triage
discipline, severity anchors and the three companion files are kept and narrowed to infrastructure as code.
