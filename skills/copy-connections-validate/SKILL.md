---
name: copy-connections-validate
description: Use this skill as part of the Copy Connections workflow to validate a copy plan before execution. Checks for naming collisions, type compatibility, schema compatibility, and transformation portability. Can also be used standalone to answer "what would break if I copied these?" without committing to execute. Invoke from the copy-connections coordinator or directly when the user asks to validate an existing plan.
---

# Copy Connections — Validate

This skill takes a copy plan and checks it for issues before execution. It produces `.copy-connections/validation_report.yaml`. Execute refuses to run without a passing report.

Validate can also run standalone — a user who wants to know "would this copy work?" without committing to execute can run just this skill against an existing plan.

## Required checkpoints

1. **Load the plan and verify it's current.** If `.copy-connections/copy_plan.yaml` doesn't exist, tell the user to run scope first. If it exists but references entities (destinations, connections, groups) that no longer match the current Fivetran state, flag it and suggest re-scoping.
2. **Every check is read-only.** Validate does not write to Fivetran. If a check would require a write to verify (e.g., dry-running a connection creation), note it as an `info`-level item rather than attempting the write.
3. **Classify every finding by severity** (`blocking`, `warning`, `info`). Severity drives behavior: blocking fails the plan; warnings require acknowledgment; info is informational.
4. **Write a structured report, not just prose.** The report is machine-readable (execute reads it) and human-readable (user reads it). Use the schema in `schemas/validation_report.schema.yaml`.
5. **Summarize clearly at the end.** Blocking issues → explain what needs to change and direct the user to re-scope. Warnings → list them with codes so the user can acknowledge. Info-only pass → say so plainly.

## Step-by-step

### 1. Load the plan

Read `.copy-connections/copy_plan.yaml`. If missing, stop and tell the user: "No copy plan found. Run scope first to create one."

Compute a checksum of the plan (simple hash is fine) and record it in the validation report metadata. Execute will verify the plan hasn't been modified between validate and execute — if it has, validate must re-run.

### 2. Verify plan freshness against current state

Before running checks, do a quick sanity pass:

- Call `list_groups` and `list_destinations` — do the source and target (if existing) referenced in the plan still exist?
- Call `list_connections` for the source group — are the source connections still there with the same IDs?

If anything has changed (destination deleted, connection moved, new connection added that wasn't in the plan), flag as **blocking** with code `PLAN_STALE`. The user needs to re-scope.

This is the only outward-facing state check at the top level. Per-connection deep checks happen in step 3.

### 3. Run destination-level checks

For the destination portion of the plan:

**Naming collision** (`blocking`, code `DESTINATION_NAMING_COLLISION`).
If the plan says "create new group with name X" and a group with name X already exists, that's blocking. Exception: if the existing group's destination is the exact one the plan intends to create (same config), degrade to `warning` with `DESTINATION_ALREADY_EXISTS` — user might have partially run execute before, which is fine under the idempotency model.

**Type support** (`blocking`, code `DESTINATION_TYPE_UNSUPPORTED`).
If the plan specifies a destination type Fivetran doesn't support, blocking. (Unlikely given we just created it in scope, but check anyway for standalone validate runs.)

**Portability caveats** (`info`, code `DESTINATION_FIELD_NOT_PORTED`).
If the plan says "copied portable config from X" and there are non-portable fields in the source destination that obviously didn't carry over (warehouse size, networking specifics, role assignments), list them as info so the user knows what they'll want to review post-copy.

**Same source and target** (`warning`, code `DESTINATION_SAME_AS_SOURCE`).
If the plan's source and target destinations are the same destination, warn. Technically valid (copy-and-rename within a destination) but almost always a mistake.

### 4. Run per-connection checks

For each connection in the plan, run the following checks. Accumulate findings into the report.

**Naming collision** (`blocking`, code `CONNECTION_NAMING_COLLISION`).
If a connection with the target name already exists in the target group, that's blocking. Exception: idempotency — if the existing connection's config exactly matches the plan, degrade to `warning` with code `CONNECTION_ALREADY_EXISTS` (user resuming after partial execute).

**Service support** (`blocking`, code `CONNECTION_SERVICE_UNSUPPORTED`).
If the source connector service isn't available to the target account, blocking. Unusual — services usually are account-wide — but possible for services that require specific entitlements.

**Schema config replicability** (per-table and per-column).
For each table in the source schema config:
- If `enabled_patch_settings.allowed: false` with reason `SYSTEM_TABLE`, the enabled flag can't be patched. This is fine — system tables exist in the target too, and will be either on or off by default. `info`, code `SYSTEM_TABLE_NOT_PATCHABLE`.
- If `enabled_patch_settings.allowed: false` with reason `DELETED`, the table was deleted at source. The target won't have this table (unless it was also deleted there, unlikely for a fresh copy). `warning`, code `SOURCE_TABLE_DELETED` — the plan's `enabled: false` for this table is harmless, but surface so the user knows.
- If `enabled_patch_settings.allowed: false` with reason `OTHER`, surface the reason text. `warning`, code `TABLE_UNPATCHABLE_OTHER`.

For each column:
- `SYSTEM_COLUMN` → `info`, code `SYSTEM_COLUMN_NOT_PATCHABLE`.
- `DELETED` → `warning`, code `SOURCE_COLUMN_DELETED`.
- `OTHER` → `warning`, code `COLUMN_UNPATCHABLE_OTHER`.

Columns with hashing enabled (`hashed: true`) should always be patchable — if they're not, that's a harder warning because the privacy intent won't carry over. `warning`, code `HASHED_COLUMN_UNPATCHABLE`.

**Source connection state** (`info` or `warning`).
If the source connection is in an error state:
- `auth_token_expired`, `auth_failure`, or similar → `warning`, code `SOURCE_AUTH_BROKEN`. The user likely knows but surface again. Copy replicates config, not auth.
- `setup_state: incomplete` → `warning`, code `SOURCE_SETUP_INCOMPLETE`. Config may be partial.
- Healthy → no finding.

**Credential fields consistency** (`blocking`, code `CREDENTIAL_FIELDS_CHANGED`).
Re-fetch `get_connection_details` for the source connection. If the set of masked fields differs from what the plan recorded, the user has modified the source in the Fivetran UI between scope and validate. Blocking — re-scope.

**Custom transformations at connection level** (`info`, code `CUSTOM_TRANSFORMATION_NOT_COPYABLE`).
If the source connection has a transformation configured via the Transformations product (not a Quickstart package, not column config), that won't carry through this plugin. List the transformation name and flag for manual recreation. This is `info` not `warning` because we've told the user during scoping; this is just a reminder.

**Pause-after-trial override** (`info`, code `PAUSE_AFTER_TRIAL_OVERRIDDEN`).
Confirm the plan has `pause_after_trial: false`. If the source had it set to `true`, note this is intentional. If it wasn't overridden, that's a bug in the plan — `blocking`, code `PAUSE_AFTER_TRIAL_NOT_OVERRIDDEN` — re-scope.

### 5. Run per-transformation checks

For each Quickstart package in the plan:

**Package compatibility with target destination** (`blocking`, code `PACKAGE_INCOMPATIBLE_WITH_DESTINATION`).
Call the Quickstart package metadata endpoint. Check the target destination type is in the package's supported list. If not, blocking.

**Dependency presence** (`blocking`, code `PACKAGE_MISSING_DEPENDENCY`).
Quickstart packages depend on a connection being present. If the dependency connection isn't in the plan (user excluded it), blocking — either add the dependency connection or remove the package.

**Version/latest check** (`info`, code `PACKAGE_VERSION_OUTDATED`).
If the source uses an older version of the package and a newer version is available, note it. User can upgrade post-copy.

### 6. Compute overall status

- **Any blocking findings** → status is `fail`. Execute will refuse to run.
- **No blocking, any warnings** → status is `pass-with-warnings`. Execute will run but require the user to acknowledge warnings.
- **No blocking, no warnings** → status is `pass`. Execute runs without further ceremony.

Info-only findings do not affect status.

### 7. Write the report

Write `.copy-connections/validation_report.yaml` conforming to `schemas/validation_report.schema.yaml`. Include:

- Metadata (timestamp, plan checksum, plan path)
- Overall status
- Summary counts
- Destination findings, connection findings (keyed by source_id), transformation findings
- Acknowledgments required list (copies of every warning code for easy execute-side lookup)

### 8. Summarize to the user

Give a clear verbal summary. Lead with the overall status.

**If fail:** state the status, list the blocking issues with codes and fix directions, tell the user to run scope again.

**If pass-with-warnings:** state the status, list the warnings by code with context, remind the user that execute will require acknowledgment. Don't try to auto-acknowledge here — that's execute's job.

**If pass:** state the status, optionally mention info-level findings as FYI, tell the user they can run execute.

## Key product knowledge

**`enabled_patch_settings` severity is about intent leakage.** If the source had a column blocked or hashed (a privacy action) and the setting can't be patched on the target, the privacy intent doesn't carry over — worth a warning. If a system table has `enabled: true/false` at source that doesn't carry over, the system table's own default is almost certainly fine — info-level at most.

**Idempotency is validated, not prevented.** Users who re-run after a partial execute will hit `*_ALREADY_EXISTS` findings. Don't make this blocking by default — the user intended to continue. Degrade to warning, require acknowledgment.

**Plan staleness vs. field drift.** `PLAN_STALE` at the top level (destination deleted, connection removed) is blocking — the plan refers to things that no longer exist. `CREDENTIAL_FIELDS_CHANGED` at the per-connection level is blocking for a related reason — the source was modified between scope and validate, and the plan might be missing credential fields. Both require re-scoping. Don't try to "just update the plan" from validate — that's not validate's job and it would mask an audit gap.

**Custom transformations need to be surfaced multiple times.** Once in scope, once in validate, once in the results file post-execute. Users forget between scoping and execution. Redundancy is fine here.

**Validation is not a setup test.** We do not call `run_connection_setup_tests` or `run_destination_setup_tests` during validate — those are writes-adjacent (they hit the source/target system) and should only happen during execute. Validate reads Fivetran's state; it does not poke source or target systems.

## When validate fails

If status is `fail`, the most common causes:

- User modified the source in Fivetran UI between scope and validate → re-scope
- Source connection or destination was deleted → re-scope (possibly after restoring or adjusting the plan)
- Target destination already has a conflicting group/connection name → re-scope with different names
- A Quickstart package isn't compatible with the target destination → re-scope to remove the package

In all cases, the fix is re-scoping. Validate doesn't try to patch the plan; it just reports.

## Standalone use

A user can invoke validate without having just run scope:

> "I've got a copy plan from last week — what would break if I ran it today?"

In that case:
1. Confirm `.copy-connections/copy_plan.yaml` exists
2. Run all checks as normal
3. Report — the freshness check in step 2 will catch most drift since the plan was written

This is the main reason validate is its own skill instead of folded into execute: it has real standalone value for dry-run-style workflows.

## Anti-patterns to avoid

- **Don't write to Fivetran from validate.** Read-only, always. No setup tests, no connection creation, no patches.
- **Don't try to auto-fix the plan.** If checks fail, report and send the user back to scope. Automated plan patching would mask issues that should be surfaced.
- **Don't treat info findings as warnings.** Info is FYI only — no acknowledgment required. Don't inflate severity.
- **Don't skip the plan checksum.** Execute uses it to detect plan modification after validation.
- **Don't overwrite a passing report with a failing one silently.** If the user runs validate twice and the second run fails, make that obvious — they may have changed something they didn't mean to.

## Example output

For a pass-with-warnings result:

```
Validation: PASS WITH WARNINGS (2 warnings, 3 info)

Destination: ✓ snowflake_test ready

Connections:
  ✓ postgres_app (no issues)
  ✓ stripe_prod (1 info: SOURCE_TABLE_DELETED on charges_legacy — harmless, table is gone at source)
  ⚠ salesforce_prod (1 warning: HASHED_COLUMN_UNPATCHABLE on contacts.ssn — the hashing setting may not carry over to the new connection; verify post-copy)
  ⚠ hubspot_marketing (1 warning: SOURCE_AUTH_BROKEN — source is in auth_token_expired state; you confirmed during scoping that you want to debug in the new env)

Transformations:
  ✓ stripe_analytics Quickstart package — compatible with Snowflake

Execute will require acknowledgment of the 2 warnings. When ready, run execute.
```