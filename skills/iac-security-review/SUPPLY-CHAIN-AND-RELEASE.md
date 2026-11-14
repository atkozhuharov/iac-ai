# Supply Chain and Release Review

> Adapted from Cloudflare's security-audit skill (`skills/security-audit`, commit `c1c8a8c14710`, MIT; see [LICENSE](LICENSE)) and narrowed to infrastructure as code. Evidence comes from the configuration, the plan and the state you can read: run nothing, probe nothing live.

#### When to use this file

Reach for this file when the target resolves dependencies, builds from untrusted contributions, runs CI, creates release artifacts, signs or promotes builds, loads plugins, or updates deployed software. This domain covers trust handoffs from source and dependency to the artifact a user runs. Use `CLOUD-AND-DEPLOYMENT.md` for runtime workload authority.

In Terraform/OpenTofu terms: module `source` and `version` (a `?ref=` branch or tag versus a commit SHA, a registry module from an unknown namespace, a fork), provider `source` and the hashes in `.terraform.lock.hcl`, Terragrunt `terraform { source }`, container image tags versus digests, and CI workflows that plan or apply with cloud credentials — including the OIDC trust policy (`sub`, `aud`, repository and branch conditions) that grants them.


## Core discipline

- A mutable or known-vulnerable dependency is not a finding by itself. Show who can influence resolution, which build consumes it, and what execution or release boundary follows.
- Follow integrity across every handoff: source identity, resolved inputs, build worker, artifact identity, test result, signature/attestation, promotion, and update consumer.
- CI configuration is authorization code. Establish which event triggered a workflow, whose code runs, which secrets and tokens exist, and what it may publish or mutate.
- A checksum fetched from the same untrusted location as the artifact does not establish independent integrity. Identify the trusted root and failure behavior.
- Use `confirmed` for failures the configuration and workflow definitions show. Use `needs_validation` for branch protection, hosted-runner, registry, signing-service, or production promotion facts that are not observable.

## Dependency and build-input attack classes

**Dependency source and namespace confusion**
Resolver configuration can select an unintended public/private namespace, fallback registry, mirror, repository, or source URL. Review package names, source priority, lockfile and checksum use, alternate build files, platform-specific resolution, and first-install versus update behavior.

**Mutable and unbound build inputs**
Builds consume branches, tags, unverified submodules, downloaded tools, generated assets, remote includes, floating CI actions, or container tags whose content can change without source review. Require a lower-trust writer and a path into trusted build output; reproducibility by itself does not prove authenticity.

**Build-context inclusion**
Secrets, local configuration, repository metadata, test fixtures, or developer artifacts enter a package or image because the build context and ignore rules exceed intended release inputs. Confirm that the resulting artifact exposes a real credential, private data, or privileged configuration.

## CI and automation attack classes

**Untrusted code in a privileged workflow**
A pull request, issue comment, fork, dependency update, or external event runs contributor-controlled code with protected secrets, write tokens, deployment authority, or a trusted runner. Compare trigger type, checkout ref, approval gate, environment protection, and permission narrowing. Do not assume repository-host defaults that are not in source.

**Workflow command and expression confusion**
Attacker-controlled branch names, commit messages, issue fields, artifact names, matrix values, or generated output enter shell commands, template expressions, paths, or privileged workflow inputs without canonical validation.

**Cache, artifact, and workspace trust mixing**
A lower-trust job can populate a cache, artifact, shared workspace, or output that a higher-trust job later restores and executes or releases. Review cache keys and namespaces, artifact producer identity, digest binding, retention, and whether promotion re-resolves by mutable name.

**Automation identity overreach**
CI jobs receive permissions beyond the operation, repository, environment, or duration needed, and untrusted job inputs can select the affected resource. Missing least privilege alone is hardening; require a reachable privileged action.

## Release attack classes

**Build-to-promotion substitution**
Tests, review, signature, and publication refer to mutable tags, filenames, channels, or artifact IDs rather than the same immutable digest. Check every copy, repack, architecture merge, and provenance step between build and release.

**Release authorization and signing-policy gaps**
A release or signature is accepted from the wrong workflow, repository, branch, environment, key role, or threshold. Review identity claims inside attestations and verify the consumer validates them, not just a valid signature. Rotation, expiry, and revocation must fail closed where policy requires.

## Universal moves (apply across the above)

- Walk backward from what the apply runs (module, provider, image digest) to every source, generated input, credential, worker, cache, test result, and authorization decision.
- Compare untrusted and protected workflow events side by side. Mark each persisted channel crossing between them and require an immutable identity plus producer trust.

## Validation rules (apply before reporting ANY finding here)

1. Name the lower-trust actor, controllable source/cache/artifact/metadata, consuming trusted job or updater, and resulting unauthorized publication, code inclusion, secret disclosure, or privileged execution.
2. Prove artifact identity across the broken handoff. A different mutable name or unbound digest must reach a real consumer.
3. Verify built-in package-manager, repository-host, registry, and signing defaults for the pinned version. Unknown hosted controls require `needs_validation`.
4. Evidence comes from the configuration, lock files and workflow definitions. Run nothing, publish nothing, and alter no release.
5. Return `confirmed` only with a complete source-visible handoff and meaningful result. Return `needs_validation` with the precise branch, runner, registry, signing, or deployment fact an owner must observe.
