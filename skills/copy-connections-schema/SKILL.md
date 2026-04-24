---
name: copy-connections-schema
description: Use this skill AFTER credentials have been attached to the newly-created connections (via the credentials skill or manually in the Fivetran UI). Runs setup tests to verify credentials and discover the schema, then applies table/column selections, sync modes, and hashing from the copy plan. Invoke from the copy-connections coordinator after credentials are in place.
---

# Copy Connections — Schema Config (post-credentials)

This skill runs **after** credentials have been attached to the connection shells created by execute. It verifies each connection works (setup tests), then applies the schema config from the plan — which tables and columns are enabled, sync modes, and column hashing.

Schema config can only be applied after a connection has valid credentials and has discovered its schema. That's why this is a separate step from execute.

## What this skill does

For each connection in the plan:

1. Run `run_connection_setup_tests` to verify credentials work and trigger schema discovery
2. Once the schema is available, apply the plan's schema config via `modify_connection_table_config`:
   - Table-level: `enabled`, `sync_mode` (SOFT_DELETE, HISTORY, LIVE)
   - Column-level: `enabled`, `hashed`
3. Report success/failure per connection

## What this skill does NOT do

- It does not attach credentials. That's the credentials skill's job (or the user does it in the UI).
- It does not unpause connections. That remains the user's responsibility.
- It does not retry automatically. A failed test or patch is reported; the user decides what to do.

## Required checkpoints

1. **Confirm execute completed.** If `.copy-connections/results.md` doesn't exist, tell the user to run execute first.
2. **Load the plan.** Read `.copy-connections/copy_plan.yaml` for the schema config to apply per connection.
3. **Process connections one at a time.** Don't batch — if one fails, continue to the next.
4. **Run setup tests before applying schema config.** If tests fail, skip schema config for that connection (there's no schema to configure yet).
5. **Update results.md** with per-connection schema outcomes.

## Step-by-step

### 1. Verify we're in a valid state

Read `.copy-connections/results.md` to confirm execute ran. If missing, tell the user to run execute first.

Read `.copy-connections/copy_plan.yaml` to get the list of connections and their schema configs.

### 2. Determine which connections are ready

Not all connections may have credentials yet. The user may have attached creds to some connections and not others (e.g., they did the API-key ones via the credentials skill but haven't done the OAuth ones in the UI yet).

Ask the user which connections they'd like to run schema config for, or offer to try all of them — setup tests will tell us which ones have working credentials.

### 3. For each connection

**a. Run setup tests.**

Call `run_connection_setup_tests` for the connection.

- If tests **pass**: the connection has valid credentials and the schema has been discovered. Proceed to apply schema config.
- If tests **fail**: report the error. The connection either doesn't have credentials yet or has bad credentials. Skip schema config for this connection — there's nothing to configure. Suggest the user check credentials and re-run this skill later.

**b. Apply schema config from the plan.**

For each table in the plan's schema config for this connection:

- Call `modify_connection_table_config` with:
  - `enabled` (table-level)
  - `sync_mode` (SOFT_DELETE | HISTORY | LIVE)
  - `columns` dict with per-column `enabled` and `hashed`

For tables/columns where the validation report flagged `*_NOT_PATCHABLE` (SYSTEM_TABLE, DELETED, etc.), skip the PATCH call rather than firing it and getting rejected. Log the skip.

**c. Log the result.** Record per-connection: setup tests passed/failed, schema config applied/partial/skipped.

### 4. Write updated results

Update `.copy-connections/results.md` with per-connection schema config results:

- Setup tests: passed / failed (with error)
- Schema config: applied (N tables) / partial (N applied, M skipped) / skipped (setup tests failed)
- Per-table/column skips with reason codes

Example additions to the per-connection section in results.md:

```markdown
### postgres_app
- Created (paused)
- Credentials: attached via API
- Setup tests: passed
- Schema config: applied (12 tables, 45 columns)
  - 1 column config skipped: users.ssn (SYSTEM_COLUMN_NOT_PATCHABLE)

### salesforce_prod
- Created (paused)
- Credentials: OAuth — completed in Fivetran UI
- Setup tests: passed
- Schema config: applied (47 tables, 312 columns)
  - contacts.ssn hashing applied

### hubspot_marketing
- Created (paused)
- Credentials: not yet attached
- Setup tests: failed (no credentials)
- Schema config: skipped — run this skill again after attaching credentials
```

### 5. Summarize to the user

Short verbal summary: how many connections had schema config applied, how many were skipped, and what to do about the skipped ones. Mention that connections are still paused — unpausing is their call.

## Key product knowledge

**Schema discovery happens during setup tests.** When you run `run_connection_setup_tests`, Fivetran connects to the source, discovers available tables and columns, and populates the schema. Without this step, there's no schema to configure.

**Schema config patching is idempotent.** Running `modify_connection_table_config` with the same settings twice produces the same result. Safe to re-run this skill.

**`enabled_patch_settings.allowed: false` means skip, not fail.** Some tables/columns have reasons like SYSTEM_TABLE or DELETED that prevent patching their enabled flag. These were flagged during validation. Skip them cleanly rather than attempting the patch and getting an error.

**Column hashing is a privacy control.** If the plan says `hashed: true` for a column and the patch fails, that's a meaningful warning — the privacy intent didn't carry over. Surface it clearly so the user can apply it manually.

**This skill can be run multiple times.** Users may attach credentials incrementally — do OAuth ones in the UI today, come back tomorrow. Each run processes whatever's ready and skips what isn't.

## Error handling

**Setup tests fail for a connection.** Report the error, skip schema config for that connection, continue to the next. Common causes: no credentials attached yet, wrong credentials, network/firewall blocking.

**Schema config patch fails for a table.** Log the error, continue to the next table. The connection is still usable — just that table's config didn't apply.

**Schema config patch fails for a column within a table.** Log and continue. If it's a hashed column, elevate the visibility in the report.

**User runs this before credentials are attached.** Setup tests will fail for every connection. Report that and suggest they attach credentials first (via the credentials skill or the UI).

## Anti-patterns to avoid

- **Don't apply schema config without running setup tests first.** The schema doesn't exist until tests pass.
- **Don't attach credentials in this skill.** That's the credentials skill's job.
- **Don't unpause connections.** Not this skill's job.
- **Don't abort on per-connection failures.** Continue; log; report.
- **Don't retry setup tests automatically.** If they fail, report and move on.
- **Don't skip the results update.** The results file is the audit trail.
