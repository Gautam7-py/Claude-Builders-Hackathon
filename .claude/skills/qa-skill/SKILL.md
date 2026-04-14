---
name: qa-skill
description: >
  Final quality gate before output delivery. Activates after all code-generating
  agents have completed. Configures error monitoring stubs, scans for critical
  code issues, triggers the Reconciler agent if architecture drift is detected,
  applies auto-patches, and produces the final build summary. Do NOT use for
  unit testing (that is testing-skill) or agent failure recovery (that is
  error-recovery-skill).
triggers:
  - qa.yaml agent is spawned
  - devops_done.json exists and pipeline is in final phase
  - any agent requests a final quality pass
exclusions:
  - Do not use this skill during code generation phases
  - Do not use this skill to write unit tests
  - Do not use this skill to handle agent or MCP failures
---

## Value Proposition

This skill ensures the generated application meets minimum production quality
standards before being delivered to the user. It catches critical issues that
slipped past the generating agents, reconciles any drift between the architectural
contract and the actual output, and produces a clear, honest summary of what was
built, what was patched, and what was skipped.

---

## Workflow

### Step 1 — Load context

Read `.claude/context/architecture.json`, `backend_done.json`, and
`frontend_done.json`. Build a complete picture of what was planned vs what was
generated. Write a reconciliation manifest to `.claude/context/qa_manifest.json`.

### Step 2 — Architecture drift check

Compare `architecture.json.api_contracts` against `backend_done.json.endpoints_implemented`.

- If they match exactly → proceed to Step 3.
- If they differ (missing endpoints, extra endpoints, type mismatches) → **immediately spawn the Reconciler agent** before continuing. Write the diff to `.claude/context/arch_drift.json`. Do not attempt to patch drift yourself — that is the Reconciler's responsibility.
- Wait for `reconciler_done.json` before proceeding. If Reconciler fails twice, log the drift and continue with a WARNING in the final report.

### Step 3 — Critical issue scan

Scan all files in `/output/` for the following patterns, in this exact priority order:

**CRITICAL (must patch):**
- Hardcoded secrets, API keys, or passwords in source files
- Empty `catch` blocks with no logging or re-throw
- Missing CORS configuration on Express/Fastify servers
- Missing authentication middleware on protected routes
- `console.log` calls containing sensitive data

**WARNING (flag, attempt patch):**
- Missing `.env.example` file
- Undeclared environment variables referenced in code
- Missing input validation on user-facing API endpoints
- Synchronous file operations in async contexts

**INFO (flag only, do not patch):**
- TODO or FIXME comments in generated code
- Unused imports
- Inconsistent naming conventions

### Step 4 — Auto-patch

For every CRITICAL issue found, attempt to apply a minimal, targeted fix. Patch rules:
- Change only the lines that contain the issue — do not refactor surrounding code
- Document every patch in `.claude/context/qa_patches.json` with: `{ file, line, issue_type, original, patched }`
- Maximum 2 patch rounds — if an issue persists after 2 rounds, escalate to WARNING and flag for user

### Step 5 — Write qa_report.json

```json
{
  "status": "pass | pass_with_warnings | fail",
  "architecture_drift_detected": true,
  "reconciler_invoked": true,
  "critical_issues_found": 0,
  "critical_issues_patched": 0,
  "warnings": [],
  "info_flags": [],
  "patches_applied": [],
  "build_ready": true
}
```

### Step 6 — Final build summary to user

Print:

```
✅ BUILD COMPLETE
App:              <app_name>
Output dir:       /output/
GitHub:           <repo_url or "not pushed — run /output/setup.sh">
Agents run:       <comma-separated list>
Agents skipped:   <name — reason> or "none"
Arch drift found: <yes/no — reconciled/not reconciled>
Issues patched:   <count>
Warnings:         <count>
Build status:     READY | READY WITH WARNINGS | NEEDS REVIEW
```

---

## ALWAYS

- Read `architecture.json` before scanning anything — the contract is the ground truth
- Invoke Reconciler before patching if arch drift exists — never patch drift yourself
- Document every patch applied
- Be honest in the final summary — never report READY if critical unpatched issues remain

## NEVER

- Modify test files written by the Tester agent
- Delete files from `/output/` — patch in place only
- Patch WARNING-level issues without flagging them first
- Skip the architecture drift check even if no drift is expected

## CRITICAL

The `build_ready` field in `qa_report.json` is the final gate. The Orchestrator
reads this field before printing the build summary. If `build_ready: false`, the
Orchestrator must print a NEEDS REVIEW notice and stop — it must not claim success.
