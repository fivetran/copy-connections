---
name: copy-connections-scope
description: Use this skill as part of the Copy Connections workflow to gather the user's intent, fetch their current Fivetran setup (connections, destinations, transformations), and produce a structured copy plan. Invoke this from the copy-connections coordinator, not directly.
---

# Copy Connections — Scope

This skill produces the copy plan. It's the interactive phase where you learn what the user wants to copy, fetch their current state via the Fivetran MCP, and write a structured plan to `.copy-connections/copy_plan.yaml` that the validate and execute skills will consume.

Credentials are **not** part of scoping or execution. Copying a connection and adding credentials to it are separate concerns. After execute finishes, a post-execute flow (the `copy-connections-credentials` skill) offers to help attach credentials, but it's optional. Scope just captures which fields *will* need credentials eventually, so the results file can tell the user what they'll need.

## Required checkpoints

These must happen in order. Skipping any of them will produce an incomplete or wrong plan.

1. **Check for an existing plan first.** If one exists, offer to modify or start over before starting any fresh-scoping work.
2. **Understand the user's intent** (fresh flow only). Are they testing a few connections against a new destination, or migrating everything from one destination to another? The intent shapes the defaults and warnings throughout the rest of the flow.
3. **Resolve source and target destinations before asking about connections.** "Which connections to copy" is meaningless without knowing which source group to look in and which target they're going to.
4. **Create the target destination (if new) before connection-level scoping.** A plan that references a destination that doesn't yet work is worthless.
5. **Fetch full config for every connection in scope** before writing the plan. The plan needs the actual `config` blocks and schema configs so execute can replicate fidelity.
6. **Detect credential fields from the real config responses.** Do not hardcode which fields are credentials per connector — scan for masked values (`"********"`) in what Fivetran returns. This is informational (for the results file later); scope doesn't collect credentials.
7. **Show a summary and get explicit confirmation** before writing `copy_plan.yaml`.

Between these checkpoints, talk naturally. Don't force the conversation through a rigid script.

## Step-by-step

### 0. Check for an existing plan

Before starting the fresh flow, check if `.copy-connections/copy_plan.yaml` already exists.

**If no existing plan:** proceed to step 1 (fresh flow).

**If an existing plan exists:** tell the user what you see and offer two paths:

> "I see a copy plan from [timestamp]. Want to modify it, or start over?"

**Modify path:**

- Load the plan into memory
- Ask what they want to change. Common modifications:
  - Add or remove connections from the scope
  - Change the target destination (existing → different existing, or existing → new)
  - Switch schema config from as-is to reviewed (or vice versa)
  - Update specific connection configs
- Only update the fields the user explicitly changed. Preserve everything else.
- Do NOT re-fetch state or re-ask questions about unchanged parts. The user made their prior decisions deliberately; respect them.
- If they add a new connection, fetch its `get_connection_details` and `get_connection_schema_config` — that's new data that doesn't exist in the plan yet.
- Show a confirmation summary before writing: highlight what changed, then show the full plan. User re-confirms the full plan (to catch anything they forgot), not just the changes.
- Write atomically (temp file + rename) to avoid leaving a half-written plan on disk if something goes wrong.

**Start-over path:**

- Overwrite `copy_plan.yaml` with the fresh flow output (don't archive the old one unless the user asks — keeping junk files around clutters `.copy-connections/`).
- Proceed to step 1.

**A note on drift.** The modify path doesn't re-fetch current state for unchanged connections. This is deliberate: if the user made their decisions based on a snapshot from yesterday and wants to tweak one thing today, forcing them to re-confirm everything they already decided wastes their time. Drift detection happens in `copy-connections-validate`, which already re-fetches `list_connections`, `list_destinations`, and per-connection `get_connection_details` to catch anything that changed in Fivetran between scope and validate. That's the right place for it.

### 1. Understand intent

Before any API calls, get a rough sense of what the user is trying to accomplish. Common patterns:

- **Test a few connections against a new destination** — user wants to validate that a target warehouse works with their data before committing to a full migration. Default: small subset.
- **Migrate everything from one destination to another** — full cutover. Default: all connections in the source group.
- **Split one destination into two** — e.g., prod and dev, or by team. Default: user-specified subset.
- **Stand up a mirror for DR or staging** — full copy kept in sync separately. Default: all connections.
- **Something else** — ask them to describe what they're doing.

A short conversational question works — you don't need a form. Something like "What are you trying to accomplish — testing a few connections in a new destination, migrating everything, or something else?" The answer tells you defaults and tone for the rest. Don't force them into a category if their answer is bespoke; just adapt.

Once intent is understood, fetch current state. Run these MCP calls in parallel if possible:

- `list_destinations`
- `list_groups`
- `list_transformations`
- `list_transformation_projects` — to detect custom dbt projects (out of scope for this plugin, but worth flagging)

Don't call `list_connections` at the account level yet — you'll do it filtered to the source group once that's chosen. In large accounts, un-filtered `list_connections` is wasteful.

### 2. Pick the source destination

If the user's intent made this obvious ("my Redshift setup"), confirm it rather than asking. Otherwise list the destinations they have and ask.

Once you have the source destination, call `list_connections` filtered to that group (or use the full list and filter locally). Summarize the connections in that group: service type, sync frequency, health/state. If any are in an error state, flag them — the user may want to fix at source before copying.

Also surface what transformations are attached to connections in this group. Quickstart packages and column config both live at the connection level, so they come along with each connection — no separate "source transformation" decision. But telling the user what's configured helps them know what will come along.

If you detect a custom dbt project (transformation project) tied to the source, tell the user: "You have a custom dbt project — that's out of scope for this copy. You'll need to set it up separately on the new destination after we're done here." Don't try to copy it; don't block on it.

### 3. Figure out the target destination

Ask whether the target destination already exists.

**If it already exists:** confirm it by name/id and move on. Warn if the target is the same as the source — that's almost certainly a mistake, but not strictly invalid (e.g., the user might want to copy-and-rename within the same destination).

**If it doesn't exist yet:** help them create it. Three config options:

- **Copy portable config from an existing destination.** Portable fields (type-agnostic): time zone, processing location, naming conventions (e.g., lowercase), data processing region. Non-portable: warehouse/compute sizing (Snowflake warehouse size, BigQuery slots), networking specifics (PrivateLink IDs, SSH tunnel host/port, proxy agent IDs), role assignments, and anything account-specific (Snowflake account identifier, bucket paths, project IDs).
  - **Same-type copy** (e.g., Snowflake → Snowflake): nearly everything ports except credentials and account-specific identifiers. Default to recommending this path.
  - **Cross-type copy** (e.g., Redshift → Snowflake): only the type-agnostic fields port. Tell them explicitly which fields will carry over and which they'll need to fill in.
- **Start from Fivetran defaults** for the target destination type.
- **Configure from scratch** — user specifies everything.

Once the config path is resolved, actually create the destination:

1. Ask for the target destination credentials (account identifier, user, password/key-pair auth, warehouse, database — whatever's needed for the target type). These are destination credentials, not connector credentials.
2. Call `create_group` for the target group name.
3. Call `create_destination` with the composed config (portable fields from source + user-provided fields + type-specific defaults).
4. Call `run_destination_setup_tests`. If tests fail, surface the errors and stop — fix the destination before proceeding.

`create_destination` is a write operation — only do it after explicit confirmation. Don't create destinations speculatively.

### 4. Pick the connections to copy

Now that source and target are locked, ask which connections from the source to include.

Frame the question by intent (from step 1):

- Test intent → suggest 2–3 representative connections, let them narrow further if they want.
- Full migration intent → default to all connections in the source group, ask them to confirm or exclude.
- Split intent → ask which connections go to this target (vs. staying or going elsewhere).
- Custom intent → open-ended list.

For any connections in an error state at the source, ask explicitly whether to include them. Some users want to fix upstream first; others want to debug in the new environment.

### 5. Schema config handling

Default: copy schema configs as-is. Offer "review each one" as an option for users who want to prune tables or columns before copying.

Most users pick "as-is." Reviewing each connection's schema is tedious; offer it, don't push it.

### 6. Fetch detailed configs

For each connection in scope, call:

- `get_connection_details` — returns the full `config` block with credential fields masked as `"********"`
- `get_connection_schema_config` — returns table/column selections and `sync_mode` per table

For each Quickstart package attached to a connection in scope, call `get_transformation_details` to capture the package name and dependencies.

### 7. Record credential-field inventory

For each connection's `config` block, scan for masked fields (values equal to `"********"`). Those are credential fields that will need to be filled in later (during the optional post-execute credentials flow, or manually by the user in the Fivetran UI).

Record these in the plan per connection. Also classify each connection's auth style based on its config shape:

- **OAuth** — config has OAuth-related fields (e.g., `refresh_token`, `access_token`, `oauth_*`). OAuth connectors can't have their credentials patched via the API — the user must complete auth in the Fivetran UI.
- **Credential-based** — standard credential fields (password, api_key, etc.). These *can* be patched via the API if the user wants to use the post-execute credentials flow.
- **None** — rare; some connectors don't need auth at all.

The plan records this classification. Scope does **not** collect credentials and does **not** ask the user about credential collection mode — credentials happen post-execute, optionally.

### 8. Summarize and confirm

Before writing the plan, show the user a table like this:

```
Copy plan summary:

Intent: Test a few connections before full migration

Source:      prod_redshift (Redshift)
Target:      snowflake_test (Snowflake) — created just now, portable config copied from prod_redshift

Connections (3 of 4 in prod_redshift, hubspot_marketing excluded):
  salesforce_prod    → salesforce_prod    (Salesforce, 47 tables, OAuth — UI needed for auth later)
  postgres_app       → postgres_app       (Postgres, 12 tables, password needed later)
  stripe_prod        → stripe_prod        (Stripe, 23 tables, API key needed later; hash on customers.email)

Transformations:
  stripe_analytics Quickstart package (tied to stripe_prod)

Note: Connections will be created without credentials and left paused. After the copy, you can either:
  — Let me help attach credentials and run setup tests, or
  — Fill in credentials yourself via the Fivetran UI.

Write this plan?
```

If they confirm, write `.copy-connections/copy_plan.yaml`. If they want to change something, adjust and re-summarize.

### 9. Write the plan

Write `.copy-connections/copy_plan.yaml` conforming to the schema in `schemas/copy_plan.schema.yaml` (at the plugin root).

Key fields to capture verbatim from what Fivetran returned:

- Connection `config` block with masked fields replaced by empty strings (don't carry `"********"` through)
- Schema config with every table's `enabled`, `sync_mode`, and every column's `enabled` and `hashed`
- Every connection's `pause_after_trial` should be **overridden to `false`** in the plan, regardless of the source value (see "Key product knowledge" below)
- Per-connection: list of masked field names (the credential inventory) and `auth_style` classification

Tell the user the plan is written and mention the next step: run validate.

## Key product knowledge

**Masked credential fields.** Fivetran's `get_connection_details` returns `"********"` for credential fields and real values for everything else. Never assume which fields are credentials for a given connector — always scan. Different connectors have different credential shapes (Postgres has `password`; Stripe has `api_key`; Salesforce has OAuth tokens that may or may not appear depending on auth method).

**Credentials are not part of copying.** A connection exists once it's created, even without credentials. Credentials make it *sync*, not make it *exist*. This plugin creates connections; adding credentials is a separate, optional post-step. This framing matters for user communication — don't tell them "we need credentials to copy" because we don't.

**`pause_after_trial: true` is a billing-era setting.** It causes the connection to auto-unpause when the account's trial ends. If the user's account is on a paid plan, this field is harmless. If the account is on trial, it's a landmine — copies could start syncing unexpectedly at an arbitrary future date. Override to `false` in the plan for safety. If the user protests, tell them why.

**Destination config portability is type-sensitive.** Same-type: nearly everything ports except credentials and account-specific identifiers. Cross-type: only type-agnostic fields. Be honest with the user about what's carrying over.

**Quickstart packages vs. custom dbt vs. column config.** These are three different things that users conflate:

- **Quickstart packages** are Fivetran-managed dbt transformations available as pre-built packages. Port across destinations if the target type supports them. In scope.
- **Custom dbt projects** (Fivetran calls them "transformation projects") are customer-authored dbt code. Out of scope — flag and tell the user to set up separately.
- **Column config** is hashing and blocking at the column level. Stored in the connection's schema config. Copies automatically with the schema config flow. Never describe this as a "transformation" to the user.

**Source connection state doesn't block the copy.** If a connection is in `auth_token_expired` or another error state, copying is still valid — it replicates the current broken config. The user may want to fix at source first, or they may want to debug in the new environment. Ask, don't assume.

**`enabled_patch_settings.allowed: false` is a validation concern, not a scoping concern.** Some tables/columns have `SYSTEM_TABLE`, `DELETED`, or `OTHER` reason codes that prevent their `enabled` flag from being patched. We capture everything in the plan; validation flags which settings won't actually replicate.

**Intent shapes defaults, not hard rules.** The test-a-few-connections user and the full-migration user can both end up with any scope of connections. Intent is a prior, not a constraint.

## When to re-enter scope from other skills

- **From validate**, when blocking issues require re-scoping (e.g., the user wants to exclude a connection that can't copy cleanly).
- **From execute**, when execution discovers new information that wasn't in the plan (very rare — shouldn't happen if validation passed).
- **From the coordinator**, when the user says "wait, let me change my mind" at any point.

In all cases: the plan in `.copy-connections/copy_plan.yaml` is overwritten, and any existing `validation_report.yaml` is considered stale (validate must re-run before execute).

If the destination was already created in a prior scoping run and the user now wants to change it, don't auto-delete it — surface that the existing destination is orphaned and ask what to do. Destination cleanup is the user's responsibility.

## Anti-patterns to avoid

- **Don't ask connection-level questions before destinations are resolved.**
- **Don't create the target destination before the user explicitly confirms.**
- **Don't fetch detailed configs until scope is locked.** `get_connection_details` per-connection is expensive; only do it after the user has confirmed the connection list.
- **Don't collect credentials during scoping.** Credentials happen post-execute, optionally. Scope just records which fields will need credentials.
- **Don't tell the user "we need your credentials to copy."** We don't.
- **Don't write the plan without confirmation.**
- **Don't conflate "transformations" with "column config" when talking to the user.**
- **Don't speculate about which fields are credentials.** Scan for masked values.
- **Don't try to cover custom dbt.** It's out of scope. Flag and move on.
- **Don't force a user into one of the stock intent categories.** They're a prior for defaults, not a form.

## Example flow fragment

Use this as a reference for the *feel* of the conversation, not a script. The complete end-to-end example with fictional Maya is in the top-level plan document (`COPY_CONNECTIONS_PLAN.md` Section 7).

```
User: I want to copy my Fivetran connections from Redshift to a new Snowflake destination.

Claude: What are you trying to accomplish here — testing a few connections in a new warehouse before committing to a full migration, migrating everything over, or something else?

User: Testing a few first. I want to make sure Snowflake works with our data before I move prod.

[fetch current state — list_destinations, list_groups, list_transformations, list_transformation_projects in parallel]

Claude: Got it. You have one Redshift destination (group `prod_redshift`) and no custom dbt projects, so this should be clean. Is `prod_redshift` the source you want to test from?

User: Yes.

[list_connections filtered to prod_redshift]

Claude: Four connections in prod_redshift: salesforce_prod (healthy), postgres_app (healthy), stripe_prod (healthy), and hubspot_marketing (in auth_token_expired state). The stripe one has a Quickstart package (stripe_analytics) and some column config — customers.email is hashed. Salesforce has contacts.ssn blocked.

For the target — does the Snowflake destination already exist, or do I need to create one?

User: Need to create it.

Claude: Three options for the config — copy portable fields from prod_redshift (that's type-agnostic stuff like time zone, processing location, naming conventions), start from Snowflake defaults, or configure from scratch. For a cross-type copy like Redshift → Snowflake, only a few fields port over, but it saves you some setup.

User: Copy portable from prod_redshift is fine.

[... ask for Snowflake credentials, create_group, create_destination, run_destination_setup_tests ...]

Claude: Destination snowflake_test created and tests passed. Now — which of the four connections do you want to include? For a test, I'd suggest 2–3 representative ones. The HubSpot one is broken — worth skipping for a test, unless you specifically want to see how it behaves in the new env.

[... conversation continues through connection selection and schema choices ...]

Claude: Ready to write the plan. Connections will be created without credentials and left paused — after the copy, I can help attach credentials and run setup tests, or you can do it yourself in the UI. Either way. Sound good?
```