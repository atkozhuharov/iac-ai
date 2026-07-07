---
name: terraform-cloud
description: >-
  Trigger Terraform Cloud / TFE remote runs and poll their progress and logs
  over the API with curl. Use when the user wants to run a terraform plan/apply
  against a cloud/remote backend, watch a remote run's status, or fetch the
  plan/apply output for a run by its URL or run ID. Works against app.terraform.io
  and any TFE-compatible host (e.g. InfraDots) — set TFC_HOST.
---

# Terraform Cloud runs over curl

Drive a Terraform Cloud / TFE **remote execution** end to end: find the API
token, kick off `terraform plan`, grab the run URL it prints, then poll the run
status and stream the job log with `curl` — no need to keep the CLI attached.

The same JSON:API works on `app.terraform.io` and any TFE-compatible backend
(InfraDots hosts like `app.infradots.com`, self-hosted TFE, etc.). Everything
below is host-agnostic; just set `TFC_HOST`.

## 0. Pick the host

```bash
export TFC_HOST="${TFC_HOST:-app.terraform.io}"   # or app.infradots.com, etc.
```

If a `cloud {}` / `backend "remote"` block is already configured, read the host
from it instead of guessing:

```bash
# hostname from a cloud {} block (falls back to app.terraform.io, TFC's default)
grep -RhoE 'hostname[[:space:]]*=[[:space:]]*"[^"]+"' *.tf 2>/dev/null \
  | sed -E 's/.*"([^"]+)".*/\1/' | head -1
```

## 1. Find the token (do this first)

Check these sources **in order** and stop at the first hit. Never print the
token — capture it into a variable.

```bash
# a) Host-specific env var. Terraform encodes the host: '.' -> '_', '-' -> '__'
ENVVAR="TF_TOKEN_$(printf '%s' "$TFC_HOST" | sed 's/-/__/g; s/\./_/g')"
TOKEN="${!ENVVAR:-}"

# b) Generic TFE token env var
: "${TOKEN:=${TFE_TOKEN:-}}"

# c) The credentials file written by `terraform login`
CRED="$HOME/.terraform.d/credentials.tfrc.json"
if [ -z "$TOKEN" ] && [ -f "$CRED" ]; then
  TOKEN=$(jq -r --arg h "$TFC_HOST" '.credentials[$h].token // empty' "$CRED")
fi

if [ -z "$TOKEN" ]; then
  echo "No token for $TFC_HOST. Run: terraform login $TFC_HOST" >&2
fi
```

Notes:
- `credentials.tfrc.json` is keyed by exact host string, e.g. `"app.terraform.io"`
  or `"infradots.com:8088"` (port included). List what's available with:
  `jq -r '.credentials | keys[]' "$CRED"`
- A legacy `~/.terraform.d/credentials.tfrc` (HCL, not JSON) may exist instead;
  parse the `token = "..."` line under the matching `credentials "<host>"` block.
- If nothing is found, the user must run `terraform login $TFC_HOST` (opens a
  browser) — this is interactive; do not try to fabricate a token.

Reusable API caller for the rest of this skill:

```bash
tfc() {  # tfc <path> [curl args...]   e.g. tfc /runs/$RUN_ID
  local path="$1"; shift
  curl -sf \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/vnd.api+json" \
    "https://$TFC_HOST/api/v2$path" "$@"
}
```

## 2. Trigger the plan and capture the run URL

With a `cloud {}` / remote backend, `terraform plan` streams live and blocks
until the remote run finishes. To poll instead, run it in the background and
scrape the run URL/ID it prints early:

```bash
terraform plan -no-color 2>&1 | tee /tmp/tf-plan.out &
# The CLI prints a line like:
#   To view this run in HCP Terraform, open:
#   https://app.terraform.io/app/<org>/<workspace>/runs/run-AbCdEf12345
sleep 5
RUN_ID=$(grep -oE 'run-[A-Za-z0-9]+' /tmp/tf-plan.out | head -1)
echo "run id: $RUN_ID"
```

If the user hands you a run **URL** directly, just extract the ID:

```bash
RUN_ID=$(printf '%s' "$RUN_URL" | grep -oE 'run-[A-Za-z0-9]+' | head -1)
```

Pure-API alternative (no CLI) — create a plan-only run on a workspace:

```bash
WS_ID=$(tfc "/organizations/$ORG/workspaces/$WORKSPACE" | jq -r '.data.id')
RUN_ID=$(tfc /runs -X POST -d @- <<JSON | jq -r '.data.id'
{"data":{"attributes":{"message":"triggered via curl","plan-only":true},
  "type":"runs",
  "relationships":{"workspace":{"data":{"type":"workspaces","id":"$WS_ID"}}}}}
JSON
)
```

## 3. Poll run status

```bash
tfc "/runs/$RUN_ID" | jq -r '.data.attributes.status'
```

Watch until a terminal state:

```bash
while :; do
  S=$(tfc "/runs/$RUN_ID" | jq -r '.data.attributes.status')
  printf '%s  run %s: %s\n' "$(date +%T)" "$RUN_ID" "$S"
  case "$S" in
    planned|planned_and_finished|applied|errored|canceled|discarded|\
    cost_estimated|policy_checked|policy_soft_failed) break ;;
  esac
  sleep 5
done
```

Status lifecycle (common ones):

| Phase    | Statuses (in order)                                                    |
|----------|-----------------------------------------------------------------------|
| queued   | `pending` → `fetching` → `pre_plan_running`                           |
| plan     | `planning` → `planned` (or `planned_and_finished` for plan-only/no-op)|
| gates    | `cost_estimating`/`cost_estimated`, `policy_checking`/`policy_checked`, `policy_soft_failed` |
| apply    | `confirmed` → `apply_queued` → `applying` → `applied`                 |
| terminal | `errored`, `canceled`, `discarded`                                     |

## 4. Fetch the job (plan / apply) log

The run points at a `plan` (and later an `apply`). Each exposes a **pre-signed**
`log-read-url` (archivist) — fetch it **without** the auth header. It streams and
supports `?limit=&offset=`; re-fetch until the run reaches a terminal state to
tail it.

```bash
# --- plan log ---
PLAN_ID=$(tfc "/runs/$RUN_ID" | jq -r '.data.relationships.plan.data.id')
PLAN_LOG=$(tfc "/plans/$PLAN_ID" | jq -r '.data.attributes."log-read-url"')
curl -s "$PLAN_LOG"        # no Authorization header — it's a signed URL

# --- apply log (after apply starts) ---
APPLY_ID=$(tfc "/runs/$RUN_ID" | jq -r '.data.relationships.apply.data.id // empty')
if [ -n "$APPLY_ID" ]; then
  APPLY_LOG=$(tfc "/applies/$APPLY_ID" | jq -r '.data.attributes."log-read-url"')
  curl -s "$APPLY_LOG"
fi
```

Structured plan result instead of raw log:

```bash
tfc "/plans/$PLAN_ID" | jq '.data.attributes | {
  status, "resource-additions", "resource-changes", "resource-destructions"
}'
```

## 5. Act on a planned run (optional)

```bash
# apply a run waiting for confirmation
tfc "/runs/$RUN_ID/actions/apply"   -X POST -d '{"comment":"approved via curl"}'
# or discard it
tfc "/runs/$RUN_ID/actions/discard" -X POST -d '{"comment":"not needed"}'
```

## Gotchas

- **Never log the token.** Redact it from any echoed commands.
- `Content-Type: application/vnd.api+json` is required on API calls; the
  `log-read-url` is plain object storage — send no auth header there.
- Wrong/expired token → `401`; `curl -sf` exits non-zero silently. Drop `-f` or
  check the body when debugging auth.
- `run-`, `plan-`, `apply-`, `ws-` prefixes identify resource IDs in URLs/JSON.
- Host must match the credentials file key **exactly**, including any `:port`.
