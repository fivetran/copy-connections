---
name: copy-connections
description: Use this skill when the user wants to copy, clone, migrate, or replicate their Fivetran connections from one destination to another — whether same destination type (e.g., splitting one warehouse into two) or different (e.g., Redshift to Snowflake). Handles the end-to-end workflow including scoping, validation, credential handling, and execution.
---

# Copy Connections (coordinator)

**Purpose.** This is the top-level coordinator for the Copy Connections workflow. It orchestrates four sub-skills:

1. `copy-connections-scope` — gather user intent, fetch current state, produce a structured copy plan
2. `copy-connections-credentials` — decide on the credential collection approach
3. `copy-connections-validate` — check the plan for issues before execution
4. `copy-connections-execute` — run the copy and produce a results artifact

**State lives in `.copy-connections/`** in the user's working directory:

- `copy_plan.yaml` — written by scope, consumed by validate and execute
- `validation_report.yaml` — written by validate, consumed by execute
- `credentials.yaml` (optional) — written by the user if they chose file-based credential collection
- `results.md` — written by execute

**Key constraints this coordinator enforces:**

- Execute refuses to run without a passing `validation_report.yaml`
- Validate refuses to run without a `copy_plan.yaml`
- The coordinator can re-enter scope at any phase if the user wants to change their mind; this invalidates downstream artifacts
- If sub-skills detect inconsistent state (e.g., plan exists but references connections that have since been deleted), the coordinator surfaces this and asks the user how to proceed

**What the coordinator does not do.** It does not implement business logic itself. All product knowledge (what's copyable, what isn't, which config fields are portable across destination types, how to handle OAuth connectors) lives in the sub-skills. The coordinator is flow control.

<!-- TODO(Phase 3): Flesh out the trigger patterns, the state-check logic, and the re-entry behavior. -->
