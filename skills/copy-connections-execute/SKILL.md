---
name: copy-connections-execute
description: Use this skill as part of the Copy Connections workflow to execute a validated copy plan. Creates connection shells (without credentials), applies schema and column config, installs Quickstart packages, and produces a results file with next steps. Refuses to run without a passing validation report. Invoke from the copy-connections coordinator.
---

# Copy Connections — Execute

This skill performs the actual copy. It takes a validated plan and produces `.copy-connections/results.md`.

**Execute does not handle credentials.** Connections are created using `run_setup_tests: false`, which lets them be created without valid credentials. The optional `copy-connections-credentials` skill handles credential attachment as a separate, post-execute step.

**Execute does not unpause connections.** Connections are created paused by default. Unpausing is the user's responsibility after they've verified everything is in order.

## Required checkpoints

1. **Verify plan and validation report exist and match.** Execute reads the plan's checksum from the validation report. If the plan has been modified since validation, refuse to run.
2. **Verify validation status is `pass` or `pass-with-warnings`.** If `fail`, refuse. If `pass-with-warnings`, surface them and require explicit user acknowledgment before proceeding.
3. **Confirm user is ready to execute.** Final yes/no before any writes.
4. **Always use `run_setup_tests: false` on `create_connection`.** We don't have credentials at this point; running tests would fail noisily.
5. **Continue on per-connection failures.** Don't abort the whole run if one connection fails. Log the failure and move to the next.
6. **Produce `results.md` at the end regardless of outcome.** Partial successes, full successes, and full failures all produce a results file.

## Step-by-step

### 1. Pre-flight

Read `.copy-connections/copy_plan.yaml` and `.copy-connections/validation_report.yaml`. If either is missing, stop and tell the user which one to generate first (scope for plan, validate for report).

Compute the current checksum of `copy_plan.yaml` and compare to the checksum recorded in `validation_report.yaml`. If they don't match, refuse to run: the plan has been modified since validation. Tell the user to re-run validate.

Check validation status:
- `fail` → stop. Tell the user to re-scope based on the blocking issues.
- `pass-with-warnings` → surface each warning by code with its context, ask for explicit acknowledgment.
- `pass` → proceed without further ceremony.

Ask for final confirmation before any writes.

### 2. Destination setup (only if the target is new and wasn't created during scope)

Scope is responsible for creating the destination if it's new, because we need a working destination before we can fetch per-connection configs. In practice, if the user's flow was scope → validate → execute, the destination already exists.

If for some reason the destination doesn't exist yet (e.g., the user ran scope against an existing target that was later deleted), the validation report should have caught this as `PLAN_STALE`. If we get to execute and the destination is missing anyway, stop and re-scope.

### 3. For each connection in the plan

Process connections sequentially. For each one:

**a. Check for idempotency.** Call `list_connections` once at the start of execution and cache. If a connection with the target name already exists in the target group with a config that matches the plan, skip creation and log `ALREADY_EXISTS`. Don't attempt to update existing connections — too risky in v0.

**b. Create the connection shell.** Call `create_connection` with:
- Full `config` block from the plan (credential fields as empty strings)
- `pause_after_trial: false` (always, regardless of source value)
- `run_setup_tests: false` (always — we don't have credentials)

The connection is created paused by default. This is the Fivetran-side default behavior; we rely on it.

**c. Apply schema config.** For each table in the plan's schema config, call `modify_connection_table_config` with:
- `enabled` (table-level)
- `sync_mode` (SOFT_DELETE | HISTORY | LIVE)
- `columns` dict with per-column `enabled` and `hashed`

For tables/columns where the validation report flagged `*_NOT_PATCHABLE` (SYSTEM_TABLE, DELETED, etc.), skip the PATCH call rather than firing it and getting rejected. Log the skip.

**d. Log the result.** Record per-connection: success, partial success (some schema patches failed), or failure. Continue to the next connection regardless.

### 4. For each Quickstart transformation package in the plan

After all connections are created:

**a. Verify dependencies are present.** The package depends on one or more target connections existing. The validation step already checked this, but sanity-check against the actual execute results — if a dependency connection failed to create, skip the package and log.

**b. Create the transformation.** Call `create_transformation` referencing the package and the target destination.

**c. Log the result.** Success, skipped (missing dependency), or failure.

### 5. Write results.md

Write `.copy-connections/results.md` with:

```markdown
# Copy Connections Results

**Date:** <ISO timestamp>
**Source:** <source group name>
**Target:** <target group name>

## Summary

- ✅ N connections created successfully
- ⚠️ M connections created with schema issues (config applied with skips)
- ❌ K connections failed to create
- ✅ P Quickstart packages installed
- ❌ Q Quickstart packages failed or skipped

## Per-connection results

### salesforce_prod
- ✅ Created (paused)
- ✅ Schema config applied (47 tables)
- ⚠️ 1 column config skipped: contacts.ssn (target schema config patch returned HASHED_COLUMN_UNPATCHABLE — the hashing intent didn't carry over; re-apply manually if needed)
- Auth: OAuth — complete authentication in the Fivetran UI when ready

### postgres_app
- ✅ Created (paused)
- ✅ Schema config applied (12 tables)
- Auth: Password needed — see next steps

### stripe_prod
- ✅ Created (paused)
- ✅ Schema config applied (23 tables, column hash on customers.email)
- Auth: API key needed — see next steps

## Transformations

- ✅ stripe_analytics Quickstart package installed

## Next steps

1. **Attach credentials and run setup tests.** Either:
   - Run the `copy-connections-credentials` skill: I'll help you attach credentials and test each credential-based connection.
   - Or do it yourself in the Fivetran UI for each connection.

2. **For OAuth connectors, complete authentication in the Fivetran UI** when you're ready. OAuth can't be completed from Claude Code.

3. **Unpause connections** when you're ready for them to sync. You can do this one at a time in the Fivetran UI, or via the `sync_connection` MCP tool.

4. **Recreate custom transformations** that were flagged during validation:
   - (list any CUSTOM_TRANSFORMATION_NOT_COPYABLE items from validation report here)

5. **After syncs complete**, consider comparing row counts between source and target connections to verify the copy.

## Cleanup

- Keep `.copy-connections/copy_plan.yaml` and `validation_report.yaml` for audit, or delete if you're done.
- If you used a credentials file during the post-execute credentials flow, delete it when you're done.
```

The results file is written even on failure — it's the source of truth for what happened.

### 6. Summarize to the user

Give a short verbal summary: what was created, what failed, and point at results.md for the full breakdown. Mention the optional next step: "Want me to help attach credentials and run setup tests? Or you can do it yourself in the Fivetran UI."

## Key product knowledge

**`run_setup_tests: false` is the unlock here.** Without it, `create_connection` would run setup tests that would fail because we have no credentials. With it, Fivetran creates the connection in a paused, untested state — which is exactly what we want. This is a first-class Fivetran API parameter, not a workaround.

**Connections created without credentials are real connections.** They appear in the Fivetran UI. They're just paused and untested. The user can see what got created before they add credentials.

**Connections are paused by default after `create_connection`.** Verified from real API responses (`paused: true` in the create response). We rely on this. Do not attempt to pause them explicitly after creation.

**Idempotency is name-based, not config-based.** If a connection with the target name exists in the target group, we skip creation. We don't try to diff configs and patch to match — that's v1+ territory. Users resuming a partial execute get "skipped" for already-created ones and creation for the rest.

**Schema config patches can fail per-table.** `enabled_patch_settings.allowed: false` (caught by validation) or server errors at PATCH time. Fail forward — log and continue; don't abort the connection because one table's config didn't apply.

**Never try to update existing connections.** If a connection already exists with the target name, we skip. Full stop. Updating existing connections mid-execute is a foot-gun — the user might have intentionally set up something different and we'd clobber it.

## Error handling

**Connection creation fails for one connection.** Log the error in results.md with the Fivetran error message. Continue to the next connection.

**Schema config patch fails for one table within a connection.** Log in results.md, continue to the next table. The connection is still usable.

**Destination is somehow missing when execute starts.** This should have been caught by validation's `PLAN_STALE` check. If it slips through anyway, stop and tell the user to re-scope.

**Partial run interrupted (user Ctrl-C, session ends, etc.).** Results.md is written incrementally if possible — i.e., after each connection, not only at the very end. Users who interrupt can see what got done. They can re-run execute; idempotency will skip the already-created ones.

**Quickstart package creation fails.** Log and continue. The connections are still valid.

## Anti-patterns to avoid

- **Don't run setup tests during execute.** We don't have credentials. Setup tests run during the optional credentials skill, after credentials are attached.
- **Don't collect credentials during execute.** Credentials are out of scope for this skill.
- **Don't unpause connections.** Not this skill's job.
- **Don't abort on per-connection failures.** Continue; log; report.
- **Don't attempt to update existing connections.** Skip them.
- **Don't omit results.md on failure.** It's the audit trail.
- **Don't rollback on failure.** If creation fails partway, the partial copy stays. User decides what to do. (v0 has no rollback mechanism; deleting a Fivetran group via the UI is the manual cleanup path.)