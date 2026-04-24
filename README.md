# Copy Connections

Copy existing Fivetran connections to a new destination with full config fidelity.  Connections are created paused and without credentials; you decide how to handle credentials afterward (let the skills help, or finish in the Fivetran UI).

Five skills that orchestrate a copy workflow against the Fivetran REST API. Packaged as a Claude Code plugin; also runs in Codex. Depends on the [Fivetran MCP server](https://github.com/fivetran/fivetran-mcp).

## Status

**v0.1.1 — implementation complete, pre-testing.** Skills are written.  They've been tested again copying into a destination of the same type.  Have not yet been tested by copying into a destination of a different type.

## What this does

Given an existing Fivetran account, Copy Connections helps you create a duplicate set of connections pointing to a different destination. Typical use cases:

- Testing a few connections against a new warehouse before committing to a full migration (e.g., Redshift → Snowflake)
- Migrating everything from one destination to another
- Splitting a single destination into prod and dev
- Standing up a staging or DR mirror

**What copies:** connection configs, schema configs (enabled tables/columns, sync mode), column config (hashing, blocking), connection-level settings (sync frequency, data delay, networking method), and Quickstart transformation packages.

**What doesn't copy:** credentials (can be added post-copy via the credentials skill or manually in the UI), OAuth tokens (re-auth required in the UI — the skills can't do browser OAuth), custom dbt projects (detected and flagged, require separate setup), historical data (new connections start fresh from the time you unpause them).

**Connections are created paused.** The skills never unpause connections — you verify and unpause when you're ready.

## The workflow

Four phases. The first three are the copy itself; the fourth is an optional convenience for attaching credentials afterward.

1. **Scope** — you describe what you want, the scope skill fetches current state, and produces a structured copy plan. Intent-first: "testing a few connections" and "migrating everything" get different defaults.
2. **Validate** — checks the plan for issues (naming collisions, type compatibility, schema portability, source state). Produces a report; execute refuses to run if validation fails.
3. **Execute** — creates the connection shells, applies schema and column config, installs Quickstart packages. Uses `run_setup_tests: false` so no credentials are needed. Produces a results file.
4. **Credentials** *(optional, post-execute)* — helps attach credentials and run setup tests on the created connections. You can skip this entirely and handle credentials yourself in the Fivetran UI.

Between phases, the coordinator asks before continuing. State is persisted in `.copy-connections/` in your working directory, so you can close your session mid-workflow and resume later.

## Requirements

* Claude Code or Codex CLI installed
* Fivetran MCP server configured in your working directory — see [setup instructions](https://github.com/fivetran/fivetran-mcp)
- These MCP tools enabled (uncomment in `server.py` if needed):
  - **Reads:** `list_connections`, `get_connection_details`, `get_connection_schema_config`, `list_destinations`, `get_destination_details`, `list_groups`, `list_transformations`, `get_transformation_details`, `list_transformation_projects`, Quickstart package metadata endpoints
  - **Writes:** `create_group`, `create_destination`, `run_destination_setup_tests`, `create_connection`, `modify_connection_table_config`, `run_connection_setup_tests`, `create_transformation`, and the credential-patch endpoint (for the optional post-execute credentials flow)

## Installation

From the directory where [fivetran-mcp](https://github.com/fivetran/fivetran-mcp) is configured:

**Claude Code:**

```
claude --plugin-dir /path/to/copy-connections
```

**Codex:**

Copy the skills into Codex's skills directory, then restart Codex:

```
cp -r /path/to/copy-connections/skills/* ~/.codex/skills/
```

Alternatively, symlink them if you want updates to flow through automatically:

```
for skill in /path/to/copy-connections/skills/*/; do
  ln -s "$skill" ~/.codex/skills/
done
```

Restart Codex after installing so it picks up the new skills.

## Usage

Start a session (Claude Code or Codex) in a working directory that has your Fivetran MCP configured. Then say something like:

> "I want to copy my Fivetran connections from Redshift to a new Snowflake destination."

The coordinator will figure out where you are in the workflow (no `.copy-connections/` → start scoping; existing plan → offer to validate or modify it; etc.) and walk you through from there.

### Modifying an existing plan

If you already have a copy plan from a previous session and want to tweak it — exclude a connection, change the target, whatever — invoke scope again. It'll detect the existing plan and ask whether to modify or start over. The modify path preserves your prior decisions and only re-asks about what's changing.

### Running a phase directly

Each skill can be invoked directly if you know what you want. For example:

> "Validate the copy plan I have."

This invokes `copy-connections-validate` without the coordinator ceremony. Useful for standalone dry-runs of an existing plan.

## Workflow files

Everything the skills write lives in `.copy-connections/`:

- `copy_plan.yaml` — the structured plan produced by scoping
- `validation_report.yaml` — issues found during validation (with a checksum of the plan, so execute can detect if the plan changed after validation)
- `results.md` — what execute did, with next steps
- `credentials.yaml.template` — generated by the credentials skill if you choose the file-based path
- `credentials.yaml` — your filled-in template (do not commit)

The credentials skill adds `credentials.yaml` to `.gitignore` if it detects a git repository. Check before pushing anyway.

## Skills

- **`copy-connections`** — coordinator. Detects current state and routes to the right sub-skill.
- **`copy-connections-scope`** — interactive scoping. Asks about intent, fetches Fivetran state, produces the plan. Supports both fresh scoping and modifying an existing plan.
- **`copy-connections-validate`** — validates the plan against current Fivetran state. Produces a report with blocking issues, warnings, and info-level findings.
- **`copy-connections-execute`** — runs the copy. Creates connection shells with `run_setup_tests: false`, applies schema and column config, installs Quickstart packages, leaves everything paused.
- **`copy-connections-credentials`** — *optional, post-execute.* Helps attach credentials (inline prompts or a template file) and run setup tests on credential-based connections. Does nothing for OAuth connectors — those need UI completion.

## What's out of scope for v0

- **Automatic unpause** — connections are created paused and stay paused. You verify and unpause when ready.
- **Cross-account copy** — one Fivetran account only.
- **Historical backfill** — copies start fresh from their unpause point.
- **Custom dbt projects** — detected and flagged during scoping, require separate setup on the new destination.
- **Row-count verification** — no post-copy data comparison.
* **OAuth completion** — can't be done from the CLI. You complete OAuth in the Fivetran UI for any OAuth connectors.
* **Rollback** — if execute fails partway through, the partial copy stays. Clean up via the Fivetran UI (delete the target group).
