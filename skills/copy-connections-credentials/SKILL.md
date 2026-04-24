---
name: copy-connections-credentials
description: Use this skill AFTER copy-connections-execute has successfully created connection shells, to help the user attach credentials to their newly-created connections. Identifies what auth each connection uses, offers flexible credential collection (bulk file, one-by-one, or UI self-service), and provides Fivetran UI links for anything the user prefers to handle themselves. Invoke from the copy-connections coordinator after execute reports success.
---

# Copy Connections — Credentials (post-execute)

This skill runs **after** execute has created the connection shells. It helps the user get credentials attached to their connections so the schema skill can run next.

## Core approach

Be flexible and user-friendly. Different users will want to handle credentials differently — some will hand you a file with everything, some will want to go connection-by-connection, some will just want links to the Fivetran UI. Adapt to the user rather than forcing a specific flow.

## What you know

During scope, `get_connection_details` returned the full config for each connection. Credential fields came back masked as `"********"`. The plan records which fields are masked and classifies each connection's auth style:

- **OAuth** — has OAuth-related masked fields (refresh_token, access_token, oauth_*). These **cannot** be patched via the API — the user must complete OAuth in the Fivetran UI via a browser redirect. Always provide the UI link for these.
- **Credential-based** — has standard masked fields (password, api_key, api_secret, etc.). These **can** be patched via the API using `modify_connection`.
- **None** — rare; some connectors don't need auth.

Use this information to tell the user what each connection needs and what their options are.

## Guidelines

1. **Start by showing the user what's needed.** For each connection, explain what kind of auth it uses and what fields need to be filled. Group by auth type so the picture is clear.

2. **For OAuth connections, always provide the Fivetran UI link.** There's no API path for OAuth — be upfront about this. Make it easy: give them the direct link to each connection's setup page.

3. **For credential-based connections, let the user choose how to provide them.** Options include:
   - Providing a file with all credentials at once
   - Giving them to you inline (all at once or one-by-one)
   - Doing it themselves in the Fivetran UI (give links)
   - Any combination — some here, some in the UI

4. **If they provide a file or bulk credentials, check for gaps.** If any connections are missing, ask about the remaining ones rather than silently skipping them.

5. **Don't be rigid about the collection method.** If they start giving you credentials inline, don't stop them to ask about a mode. If they say "here's a file," read it. If they say "just give me the links," give links. Match their energy.

6. **Patch credentials via `modify_connection`** with the credential field values. Process and report results.

7. **After patching, report what succeeded and what failed.** If something fails, let them know they can fix it in the UI or retry.

## What this skill does NOT do

- It does not run setup tests. That happens in the schema skill.
- It does not apply schema config. That happens in the schema skill.
- It does not unpause connections.

## Required checkpoints

1. **Confirm execute completed first.** If `.copy-connections/results.md` doesn't exist, tell the user to run execute first.
2. **Never log, echo, or persist credential values.** Security rules below.
3. **Update results.md** with per-connection credential outcomes.

## After credentials are handled

Prompt the user about the next step: running setup tests and applying schema config via the `copy-connections-schema` skill. If some connections still need credentials (OAuth not done yet, some skipped), mention those and suggest coming back after they're handled.

## Security rules

These are non-negotiable.

1. **Never log credential values.** Refer to the *field name* (e.g., "attached password for postgres_app") not the value.
2. **Never echo credential values back to the user.** Say "got password" not "got <value>".
3. **Never write credential values to any artifact.** Results.md, logs, plan — none of these contain credential values.
4. **Never send credential values anywhere but Fivetran.** The only valid destination is the Fivetran API.
5. **If the user provides a credentials file**, check for `.gitignore` protection. If in a git repo and `credentials.yaml` isn't in `.gitignore`, add it and tell the user. Prompt for cleanup when done — don't auto-delete.

## Error handling

- **Credential patch fails for a connection.** Report the error and move on. The user can retry or fix it in the UI.
- **User provides partial credentials.** Handle what's there, ask about the rest.
- **User abandons mid-flow.** Fine. What's done is done. Results.md reflects the current state. They can re-run later.

## Re-running this skill

Users may run this multiple times. On re-run, check results.md to see what still needs credentials and focus on those. Offer to re-patch already-done ones too (e.g., rotated key).

## Key product knowledge

**Credentials don't make a connection exist; they make it connectable.** The connection shells are already real connections in Fivetran — just paused and untested.

**OAuth truly cannot be done from the CLI.** It requires a browser redirect. Provide the link and move on.

**Setup tests are NOT part of this skill.** They happen in the schema skill after credentials are in place.

## Anti-patterns to avoid

- **Don't force a specific collection method.** Let the user drive.
- **Don't echo credential values.**
- **Don't persist credential values to any artifact.**
- **Don't attempt OAuth from this skill.**
- **Don't run setup tests.** That's the schema skill's job.
- **Don't apply schema config.** That's the schema skill's job.
