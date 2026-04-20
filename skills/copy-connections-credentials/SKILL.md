---
name: copy-connections-credentials
description: Use this skill AFTER copy-connections-execute has successfully created connection shells, to optionally help the user attach credentials to their newly-created connections and run setup tests. This is an optional post-execute flow — users can decline and add credentials manually via the Fivetran UI instead. Invoke from the copy-connections coordinator after execute reports success.
---

# Copy Connections — Credentials (post-execute, optional)

This skill runs **after** execute has created the connection shells. It's entirely optional — the user can decline and handle credentials themselves in the Fivetran UI. The connections already exist and are paused; attaching credentials is a separate, deferrable step.

## What this skill does

For each credential-based connection created by execute:

1. Collect the credential values from the user (inline prompts or via a template file)
2. Patch the connection via the Fivetran API with those values
3. Run `run_connection_setup_tests` to verify
4. Report success/failure per connection

For OAuth connectors, this skill does not patch anything — OAuth requires UI completion. It just provides the setup link for the user.

## What this skill does NOT do

- It does not unpause connections. That remains the user's responsibility after they've verified the copy.
- It does not run if the user declines. The results file already tells them what they'd need to do manually in the UI.
- It does not retry partial failures automatically. A failed credential/test is reported and the user can re-run this skill later.

## Required checkpoints

1. **Confirm execute completed first.** If `.copy-connections/results.md` doesn't exist, tell the user to run execute first.
2. **Get explicit user opt-in.** Default is ask, not assume. Users who don't respond yes shouldn't have their credentials prompted.
3. **Collect credentials for credential-based connections only.** OAuth connectors are listed with their UI links but not prompted for.
4. **Patch and test per connection.** Don't batch the patch operations — do one at a time so a failure on one doesn't block the next.
5. **Never log, echo, or persist credential values.** Security rules below.
6. **Update results.md** with the per-connection credential attachment outcomes.

## Step-by-step

### 1. Verify we're in a valid state

Read `.copy-connections/results.md` to confirm execute ran. If missing, tell the user to run execute first.

Read `.copy-connections/copy_plan.yaml` to get the list of connections and their auth classifications. Filter to the ones that are:
- `auth_style: credentials` (eligible for API-based credential attachment)

OAuth connectors are out of scope for this skill's active work — they're listed in results.md with UI links and the user handles them separately.

### 2. Ask the user whether to proceed

Frame it as an optional convenience. Something like:

> "Execute finished successfully. The connections are created and paused. Want me to help attach credentials and run setup tests now? If not, you can add credentials yourself in the Fivetran UI — it's the same amount of work either way, just a matter of where you'd rather do it."

Don't push. If they decline, confirm they have results.md for reference and stop here.

If they accept, ask about collection mode:

- **Inline prompts** — fine for small migrations (≤5 credential-based connections). Lowest friction: you ask, they paste, you move on.
- **Credentials file** — generate a template file they fill in, you read it. Better for 6+ connections, or for users who want to gather credentials in advance without babysitting a session.

Default to inline for ≤5, file for 6+. Let the user override.

### 3. For inline mode

For each credential-based connection, in order:

1. Tell the user which connection you're working on and what fields you need (e.g., "For `postgres_app` — need the database password").
2. Prompt for each masked field recorded in the plan.
3. Patch the connection with the provided values via the appropriate Fivetran API endpoint. (Patching connection config is typically done via `modify_connection` or the connector-specific patch endpoint — use whatever the MCP exposes for this.)
4. Run `run_connection_setup_tests` for that connection.
5. Report the result for this connection and move to the next.

Do not batch-collect all credentials before patching — stream one at a time. If the user stops mid-flow, the ones already done are done.

### 4. For file mode

Generate `.copy-connections/credentials.yaml.template` in the user's working directory. Content:

```yaml
# Copy Connections — Credentials Template
#
# Fill in the values below, save as credentials.yaml (without .template),
# then tell me and I'll patch your connections and run setup tests.
#
# DO NOT COMMIT THIS FILE TO VERSION CONTROL.
# If you're in a git repo, credentials.yaml should already be in .gitignore.
#
# OAuth connectors are NOT in this file — complete those in the Fivetran UI
# using the links in results.md.

connections:
  - source_id: dissented_field
    target_name: postgres_app
    service: postgres
    credentials:
      password: ""  # Database password for app_db on prod-host.example.com

  - source_id: stripe_prod_xyz
    target_name: stripe_prod
    service: stripe
    credentials:
      api_key: ""  # Stripe API key (restricted with read access is fine)
```

Per-connection:
- `source_id` — from the plan
- `target_name` — from the plan
- `service` — connector type (helps the user orient)
- `credentials:` — one entry per masked field from the plan, with a brief comment explaining what it is (where possible — Postgres password vs Stripe API key are both discoverable from the service; for exotic connectors, a best-effort description is fine)

Check if a `.git` directory exists in or above the working directory. If yes, check whether `credentials.yaml` is in `.gitignore`. If not, add it and tell the user you did. This is the belt-and-suspenders protection against accidentally committing credentials.

Tell the user the template is written, where to find it, and to let you know when they've filled it in.

When they confirm it's ready, read `.copy-connections/credentials.yaml`. For each connection entry:
1. Patch the connection with the provided credential values
2. Run `run_connection_setup_tests`
3. Report the result

### 5. Report outcomes

Update `.copy-connections/results.md` with per-connection credential attachment results. For each credential-based connection:

- ✅ Credentials attached, setup tests passed
- ⚠️ Credentials attached, setup tests failed (include the test error message)
- ❌ Credential attachment failed (include the API error)
- ⏭️ Skipped (user didn't provide credentials for this one)

Do not include credential values in results.md or anywhere else.

### 6. Cleanup prompt (file mode only)

After processing is complete, if credentials came from a file, prompt the user:

> "Credentials file is still at `.copy-connections/credentials.yaml`. Want me to delete it now, or keep it for your reference?"

Don't auto-delete. Let the user decide. If they want it kept, remind them once that it has live credentials in it.

## Security rules

These are non-negotiable and apply to both modes.

1. **Never log credential values.** Any logging must redact. If you have to describe what happened in prose, refer to the *field name* (e.g., "attached password for postgres_app") not the value.

2. **Never echo credential values back to the user.** Confirmation messages say "got password" not "got <value>". Users often check what they typed; don't reinforce that by echoing.

3. **Never write credential values to any artifact.** Results.md, logs, plan, validation report — none of these contain credential values. The credentials file is user-owned; we read it, we don't write credential values to it.

4. **Never send credential values anywhere but Fivetran.** No telemetry, no logs, nowhere else. The only valid destination is the Fivetran API.

5. **Prompt for cleanup** (file mode only) but don't force. Users may want to keep the file; that's their call.

6. **.gitignore protection** (file mode only) — when writing the template, detect git and add credentials.yaml to .gitignore if missing.

## Error handling

**Credential attachment fails for a connection** (e.g., wrong password, API rejects the patch). Report the error, skip to the next connection. The connection is still created; the user can retry this skill later or fix it in the UI.

**Setup tests fail after successful credential attachment** (e.g., wrong password at the database level vs. API level, network blocking, etc.). Report the test error. The credentials are attached (the patch succeeded); only the tests failed. The user can debug in the Fivetran UI from here — the shell + credentials are in place.

**User abandons inline flow mid-stream.** Fine. The ones already processed stay processed. Results.md reflects that. They can re-run the skill later for the remaining ones.

**User provides the wrong credential file** (missing entries, wrong service types, etc.). Validate the file against the plan before patching anything. If entries don't match the plan, surface the mismatch and ask them to fix the file. Don't patch with garbage.

## Re-running this skill

Users may run this skill multiple times — e.g., first pass attaches some credentials but one fails, second pass retries just that one. Support this:

- On re-run, read results.md and identify which connections still need credential attachment
- Offer to process only those, or all (in case the user wants to re-patch one that's already working, e.g., they rotated a key)

## Key product knowledge

**Credentials don't make a connection exist; they make it sync.** A connection created without credentials is a real connection in Fivetran. It just can't fetch data until it has valid credentials and setup tests pass. This framing matters for how you communicate with the user.

**Patching credentials is a write operation.** Treat it with the same confirmation discipline as destination creation — don't do it without the user's explicit go-ahead for each connection (or an up-front "yes, proceed through all of them" in inline mode).

**OAuth connectors truly cannot be helped here.** The OAuth flow requires a browser and a redirect — there's no Claude Code path that completes it. Don't pretend. Provide the link and move on.

**Setup test failure ≠ credential failure.** Sometimes the credential is right but the test fails for other reasons (database not yet reachable, firewall, etc.). Report the test error faithfully — don't assume it means bad credentials.

## Anti-patterns to avoid

- **Don't auto-run after execute.** Ask first.
- **Don't push the user to use this skill.** It's a convenience. Manual UI entry is equally valid.
- **Don't batch-collect all credentials before patching any.** Stream one at a time.
- **Don't echo credential values.**
- **Don't persist credential values to any plugin-written artifact.**
- **Don't auto-delete the credentials file.** Prompt.
- **Don't retry failures automatically.** Report them; let the user decide.
- **Don't attempt OAuth from this skill.** It will fail.