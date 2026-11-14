# Data Isolation and Lifecycle Review

> Adapted from Cloudflare's security-audit skill (`skills/security-audit`, commit `c1c8a8c14710`, MIT; see [LICENSE](LICENSE)) and narrowed to infrastructure as code: the application-level classes are left out and a Terraform state section is added. Evidence comes from the configuration, the plan and the state you can read: run nothing, probe nothing live.

#### When to use this file

Reach for this file when the change touches where data lives or where copies of it go: object storage and its policies, databases and their snapshots, backups and replication, encryption keys, log and metric sinks, lifecycle and retention rules, and Terraform state and its backend. This domain follows one data store through every copy and state transition. Use `CLOUD-AND-DEPLOYMENT.md` for the identities and network paths that reach it.

## Core discipline

- A tenant, environment or owner tag on a resource is not isolation. Find the policy, key, path or network control that enforces it for each reader and writer.
- Trace copies. A private primary store can become readable through a snapshot, replica, backup, export, log sink or state file with a different policy and retention.
- Deletion and retention are lifecycle contracts. Check current, versioned, replicated, backed-up and exported copies within the stated boundary.
- A retention or privacy preference is not automatically a vulnerability. Require an explicit access boundary or deletion guarantee and an unauthorized reader or later operation.
- Use `confirmed` for a complete lineage the configuration and plan show. Use `needs_validation` when account policy, external retention, CDN behavior or backup access is not visible.

## Storage and object attack classes

**Composite-key and namespace collision**
Cache keys, object paths, database uniqueness, search document IDs, temporary files, or deduplication keys omit tenant or environment. Two principals can overwrite or retrieve the same logical key even though application records carry separate owners.

**Blob and signed-reference overreach**
Object keys, attachment IDs, version IDs, shared links, or signed URLs permit operations or namespaces beyond the issuing principal's access, or remain valid after the underlying ACL changes. Bind operation, exact object/version, audience, expiry, and tenant.

**Public or cross-account storage policy**
A bucket, container, share or database policy, ACL or public-access setting grants read or write to a principal outside the intended account, organization or network. Check the effective result of policy, ACL, public-access block and encryption-key policy together: a key policy can deny what a bucket policy allows, and the reverse.

## Copies, backups and replication attack classes

**Analytics, logs, traces, and diagnostics as alternate readers**
Private content or credentials are emitted into systems with broader access, longer retention, or tenant mixing. Confirm the data class and realistic reader; field names, public identifiers, and operator-only content under intended policy are not enough.

**Export and backup scope expansion**
An export, snapshot, portability package, report, or backup includes other tenants, inaccessible object fields, soft-deleted data, secret values, or history above the requester's access. Check per-item authorization after selection and authorization to download the final artifact.

**Backup and replication boundary drift**
Encryption keys, storage accounts, cross-region replicas, restoration environments, or support snapshots have broader identity or tenant scope than primary data. Source can confirm only in-repo policy; hosted access and retention require `needs_validation`.

**Restore reintroduces invalid state**
Backup, undo, undelete, or replica recovery restores data, credentials, memberships, or permissions that current policy no longer allows. Re-authorize restored state and reapply lifecycle changes made after the snapshot.

**Retention and queued-work overrun**
Deletion completes in primary storage while queued processors, retries, exports, analytics, or generated artifacts recreate or retain the data beyond the promised boundary. Find idempotent deletion and tombstone propagation.

## Terraform state and backend attack classes

**State as an alternate reader**
State holds every managed resource's attributes in plaintext, `sensitive` ones included: generated passwords, private keys, connection strings. Whoever can read the backend (bucket, container, table, or a remote backend's workspace) or a `terraform_remote_state` data source pointed at it reads them all. Compare who can read state with who may read the most sensitive value in it.

**Backend policy and history**
The backend's bucket or table policy, encryption and versioning decide who reads current and past state. Versioning keeps every old state, so a secret rotated out of the configuration stays readable in history. Check the backend's access policy and whether CI roles, other workspaces or other accounts reach it.

**State locking and write authority**
A backend without locking, or one that more principals can write than can plan and apply, lets a lower-trust job overwrite state and change what the next apply creates or destroys.

## Universal moves (apply across the above)

- Pick one sensitive data store and draw its writes, replicas, snapshots, backups, exports, log sinks, state entries and deletion path. Mark who can read each.
- Compare the store's policy with its key policy, its snapshot sharing and its backup vault policy. The widest one is the real boundary.
- Start at backups, replicas, log sinks and state. These copies commonly carry a broader policy than the store the change was reviewed for.

## Validation rules (apply before reporting ANY finding here)

1. Name the lower-trust principal, the protected data, the copy or operation that reaches it, and the unauthorized disclosure or mutation.
2. Cite both the intended policy and the path that omits or disagrees with it. Confirm no other layer (key policy, network control, organization policy) enforces the same condition.
3. Evidence comes from the configuration, the plan and the state you can read. Run nothing and touch no real data.
4. If external retention, replication, backup access or account policy decides it, classify `needs_validation` and state the owner-observed check.
5. Return `confirmed` only with a complete lineage and concrete boundary impact. Return `needs_validation` with the exact unresolved storage, access, retention or restore fact.
