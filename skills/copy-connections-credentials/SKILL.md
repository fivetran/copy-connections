---
name: copy-connections-credentials
description: Use this skill AFTER copy-connections-execute has successfully created connection shells, to help the user attach credentials to their newly-created connections. For credential-based connections, collects and patches credentials via the API. For OAuth connectors, provides Fivetran UI links for the user to complete authentication. Invoke from the copy-connections coordinator after execute reports success.
---

# Copy Connections — Credentials (post-execute)

This skill runs **after** execute has created the connection shells. It helps the user get credentials attached to their connections — either by collecting and patching them via the API, or by directing the user to the Fivetran UI for connections that require it (OAuth, or user preference).

The connections already exist and are paused. Attaching credentials is a necessary step before schema config can be applied (the schema skill needs a discovered schema, which requires valid credentials).

## What this skill does

For each connection created by execute:

- **Credential-based connections** (password, API key, etc.): collect the credential values from the user and patch the connection via the Fivetran API
- **OAuth connections**: provide the Fivetran UI link where the user completes authentication themselves

This skill does **not** run setup tests or apply schema config — that's the `copy-connections-schema` skill's job, and it runs after credentials are in place.

## What this skill does NOT do

- It does not run setup tests. That happens in the schema skill.
- It does not apply schema config. That happens in the schema skill.
- It does not unpause connections. That remains the user's responsibility.
- It does not retry partial failures automatically. A failed credential patch is reported and the user can re-run this skill later.

## Required checkpoints

1. **Confirm execute completed first.** If `.copy-connections/results.md` doesn't exist, tell the user to run execute first.
2. **Get explicit user opt-in.** Default is ask, not assume. Users who don't respond yes shouldn't have their credentials prompted.
3. **Classify each connection.** Credential-based ones get prompted; OAuth ones get UI links.
4. **Patch per connection.** Don't batch the patch operations — do one at a time so a failure on one doesn't block the next.
5. **Never log, echo, or persist credential values.** Security rules below.
6. **Update results.md** with the per-connection credential attachment outcomes.

## Step-by-step

### 1. Verify we're in a valid state

Read `.copy-connections/results.md` to confirm execute ran. If missing, tell the user to run execute first.

Read `.copy-connections/copy_plan.yaml` to get the list of connections and their auth classifications.

### 2. Present the credential plan

Show the user what needs to happen for each connection:

- **Credential-based connections**: "I can collect your credentials and patch these via the API"
- **OAuth connections**: "These need to be completed in the Fivetran UI — here are the links: ..."

Ask whether they want to proceed with the credential-based ones now, or handle everything in the UI themselves. Don't push — the UI is equally valid.

If they decline, confirm they have results.md for reference, remind them to run the schema skill after credentials are in place, and stop here.

### 3. For credential-based connections

If they accept, ask about collection mode:

- **Inline prompts** — fine for small migrations (<=5 credential-based connections). Lowest friction: you ask, they paste, you move on.
- **Credentials file** — generate a template file they fill in, you read it. Better for 6+ connections, or for users who want to gather credentials in advance without babysitting a session.

Default to inline for <=5, file for 6+. Let the user override.

#### Inline mode

For each credential-based connection, in order:

1. Tell the user which connection you're working on and what fields you need (e.g., "For `postgres_app` — need the database password").
2. Prompt for each masked field recorded in the plan.
3. Patch the connection with the provided values via the appropriate Fivetran API endpoint.
4. Report success/failure for this connection and move to the next.

Do not batch-collect all credentials before patching — stream one at a time. If the user stops mid-flow, the ones already done are done.

#### File mode

Generate `.copy-connections/credentials.yaml.template` in the user's working directory. Content:

```yaml
# Copy Connections — Credentials Template
#
# Fill in the values below, save as credentials.yaml (without .template),
# then tell me and I'll patch your connections.
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

Check if a `.git` directory exists in or above the working directory. If yes, check whether `credentials.yaml` is in `.gitignore`. If not, add it and tell the user you did.

Tell the user the template is written, where to find it, and to let you know when they've filled it in.

When they confirm it's ready, read `.copy-connections/credentials.yaml`. Validate the file against the plan before patching anything. For each connection entry, patch the connection with the provided credential values.

### 4. For OAuth connections

List each OAuth connection with its Fivetran UI link. Something like:

> "These connections use OAuth and need to be authenticated in the Fivetran UI:
> - `salesforce_prod`: https://fivetran.com/dashboard/connectors/<id>/setup
> - `hubspot_marketing`: https://fivetran.com/dashboard/connectors/<id>/setup
>
> Complete those when you're ready, then we can run the schema skill."

### 5. Report outcomes

Update `.copy-connections/results.md` with per-connection credential results:

- Credentials attached successfully
- Credential attachment failed (include the API error)
- Skipped (user didn't provide credentials for this one)
- OAuth — directed to UI

Do not include credential values in results.md or anywhere else.

### 6. Next step prompt

After credentials are handled, prompt:

> "Credentials are in place for N connections. Ready to run setup tests and apply schema config? (That's the `copy-connections-schema` skill.)"

If some connections still need credentials (OAuth not yet done, or some skipped), mention those and suggest coming back after they're handled.

### 7. Cleanup prompt (file mode only)

If credentials came from a file, prompt:

> "Credentials file is still at `.copy-connections/credentials.yaml`. Want me to delete it now, or keep it for your reference?"

Don't auto-delete. Let the user decide. If they want it kept, remind them once that it has live credentials in it.

## Security rules

These are non-negotiable and apply to both modes.

1. **Never log credential values.** Any logging must redact. Refer to the *field name* (e.g., "attached password for postgres_app") not the value.

2. **Never echo credential values back to the user.** Confirmation messages say "got password" not "got <value>".

3. **Never write credential values to any artifact.** Results.md, logs, plan, validation report — none of these contain credential values.

4. **Never send credential values anywhere but Fivetran.** No telemetry, no logs, nowhere else. The only valid destination is the Fivetran API.

5. **Prompt for cleanup** (file mode only) but don't force.

6. **.gitignore protection** (file mode only) — when writing the template, detect git and add credentials.yaml to .gitignore if missing.

## Error handling

**Credential attachment fails for a connection** (e.g., wrong password, API rejects the patch). Report the error, skip to the next connection. The connection is still created; the user can retry this skill later or fix it in the UI.

**User abandons inline flow mid-stream.** Fine. The ones already processed stay processed. Results.md reflects that. They can re-run the skill later for the remaining ones.

**User provides the wrong credential file** (missing entries, wrong service types, etc.). Validate the file against the plan before patching anything. If entries don't match the plan, surface the mismatch and ask them to fix the file. Don't patch with garbage.

## Re-running this skill

Users may run this skill multiple times — e.g., first pass attaches some credentials but one fails, second pass retries just that one. Support this:

- On re-run, read results.md and identify which connections still need credential attachment
- Offer to process only those, or all (in case the user wants to re-patch one that's already working, e.g., they rotated a key)

## Key product knowledge

**Credentials don't make a connection exist; they make it connectable.** A connection created without credentials is a real connection in Fivetran. It just can't fetch data until it has valid credentials. This framing matters for how you communicate with the user.

**Patching credentials is a write operation.** Treat it with the same confirmation discipline as destination creation — don't do it without the user's explicit go-ahead.

**OAuth connectors truly cannot be helped here.** The OAuth flow requires a browser and a redirect — there's no Claude Code path that completes it. Provide the link and move on.

**Setup tests are NOT part of this skill.** They happen in the schema skill. This skill just gets the credentials attached. Separating these concerns means the user can attach credentials incrementally (some via API, some via UI) and then run the schema skill once everything is ready.

## Anti-patterns to avoid

- **Don't auto-run after execute.** Ask first.
- **Don't push the user to use this skill.** It's a convenience. Manual UI entry is equally valid.
- **Don't batch-collect all credentials before patching any.** Stream one at a time.
- **Don't echo credential values.**
- **Don't persist credential values to any plugin-written artifact.**
- **Don't auto-delete the credentials file.** Prompt.
- **Don't retry failures automatically.** Report them; let the user decide.
- **Don't attempt OAuth from this skill.** It will fail.
- **Don't run setup tests.** That's the schema skill's job.
- **Don't apply schema config.** That's the schema skill's job.
