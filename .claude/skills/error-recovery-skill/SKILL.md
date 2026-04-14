---
name: error-recovery-skill
description: |
  Teaches the Error Solver agent how to classify, triage, and resolve
  system-level pipeline failures — including agent crashes, MCP tool
  unavailability, missing context files, and workflow sequencing errors.
  Use this skill whenever the Error Solver agent is activated by a new
  unresolved entry in error_log.json. Trigger this skill even for WARNING-level
  errors — the skill defines silent resolution strategies for all severity levels.
  TRIGGER when: agent failure, MCP tool down, context file missing, pipeline
  error, error_log.json has unresolved entry, sequencing error detected.
  DO NOT TRIGGER when: fixing code-level bugs (use testing-skill), generating
  application code, or running standard pipeline steps without errors.
---

# Error Recovery Skill

## Overview
This skill defines the classification logic, resolution strategies, error
memory patterns, and escalation protocol the Error Solver agent must follow.
It covers all three error types — agent failures, MCP tool failures, and
missing context files — with explicit strategies for each. The output is
an updated `error_log.json` and a session `error_memory.json`.

## When to use
- Use when Error Solver agent detects a new unresolved entry in error_log.json
- Use when an agent fails, an MCP tool is unavailable, or a context file is missing
- Use when the pipeline has a workflow sequencing error

Do NOT use when:
- Fixing code-level bugs in generated application files (those go to Tester/QA)
- Running standard pipeline steps with no errors present
- Writing tests or generating application code

---

## Workflow

### Step 1 — Read error memory first
Before taking any action, read `.claude/context/error_memory.json`.
If the file does not exist, start a fresh in-memory record.

Extract:
- `components_failed_twice` — never retry these
- `strategies_tried` — never repeat a failed strategy on the same component

### Step 2 — Read and parse new errors
Read `.claude/context/error_log.json`. Find all entries where
`resolution_status` is `"pending"`.

Process one error at a time in order of severity: CRITICAL first,
WARNING second, INFO last.

### Step 3 — Classify each error before acting
Assign exactly one severity level to every error before attempting any fix.

**CRITICAL** — pipeline cannot continue without resolution:
- Architect agent crash or output file missing
- Filesystem MCP unavailable
- `execution_plan.json` or `architecture.json` missing and unrecoverable
- Database MCP down when database is required in architecture

**WARNING** — pipeline can continue in degraded state:
- Web Search MCP unavailable
- Stitch MCP unavailable
- Sentry MCP unavailable
- Non-essential agent timed out (tester, frontend on API-only app)
- Optional output file missing

**INFO** — no pipeline impact:
- Optional file missing
- Low-confidence assumption logged by an agent
- Non-blocking deprecation warning

### Step 4 — Notify user for CRITICAL errors before acting
For every CRITICAL error, print this to user output BEFORE attempting any fix:

```
🚨 CRITICAL ERROR DETECTED
Component:   <agent or tool name>
Error:       <description>
Action:      Attempting resolution now...
```

Never notify the user for WARNING or INFO errors — resolve or log silently.

### Step 5 — Apply resolution strategy

**For agent failures:**

| Attempt | Action | Wait |
|---------|--------|------|
| 1st retry | Re-spawn agent with original inputs | 30 seconds |
| 2nd retry | Re-spawn agent with reduced/minimal mode flag | 30 seconds |
| Escalate | Mark as escalated, notify Orchestrator | — |

Minimal mode means: agent runs only mandatory tasks, skips all optional ones.

**For MCP tool failures:**

| Tool | Severity | Strategy |
|------|----------|----------|
| Filesystem MCP | CRITICAL | Alert user, halt pipeline, escalate |
| GitHub MCP | CRITICAL | Attempt re-authentication once, then escalate |
| Database MCP | CRITICAL | Attempt reconnection once, then escalate |
| Web Search MCP | WARNING | Log, skip, agent continues without it |
| Stitch MCP | WARNING | Log, frontend generates components manually |
| Sentry MCP | WARNING | Log, QA adds commented setup stubs |

**For missing context files:**

| File | Strategy |
|------|----------|
| `manifest.json` | Attempt reconstruction from prompt history |
| `execution_plan.json` | Reconstruct with full agent list, medium complexity |
| `architecture.json` | CRITICAL — cannot reconstruct safely, escalate |
| `backend_done.json` | Reconstruct as skipped, continue |
| `frontend_done.json` | Reconstruct as skipped, continue |

If reconstruction produces a file, add `"reconstructed": true` to the file.
If reconstruction fails for a CRITICAL file, escalate immediately.

**For workflow sequencing errors:**
1. Re-read `execution_plan.json` to understand intended order
2. If the error is an out-of-order spawn, re-queue the affected agent
3. If reordering would corrupt existing outputs, escalate to Orchestrator

### Step 6 — Update error_log.json after every action
After each resolution attempt — success or failure — update the entry:
- Set `resolution_status` to `"resolved"`, `"degraded"`, or `"escalated"`
- Add the strategy attempted to `strategies_attempted`
- Set `notified_user` to `true` if the user was notified

Never leave an entry in `"pending"` status after acting on it.

### Step 7 — Escalate unresolvable errors
If all strategies are exhausted and the error remains:

1. Set `resolution_status` to `"escalated"` in error_log.json
2. Print to user output:
```
🚨 UNRESOLVED ERROR — ESCALATING TO ORCHESTRATOR
Component:   <agent or tool name>
Error Type:  <classification>
Severity:    CRITICAL
Attempted:   <list of strategies tried>
Resolution:  FAILED
Action:      Pipeline halted. Orchestrator notified.
```
3. Halt the pipeline — do not allow subsequent agents to run
4. Return control to the Orchestrator

### Step 8 — Write error_memory.json
Write after every invocation — not just at the end of the run — so that
subsequent invocations can read the up-to-date session history.

---

## Output format

### error_log.json entry
```json
{
  "error_id": "err_001",
  "timestamp": "2026-04-11T10:00:00Z",
  "component": "backend-agent",
  "error_type": "agent_failure",
  "severity": "CRITICAL",
  "description": "Backend agent failed to write backend_done.json",
  "strategies_attempted": [
    { "strategy": "retry_1", "outcome": "failed", "timestamp": "" },
    { "strategy": "minimal_mode", "outcome": "failed", "timestamp": "" }
  ],
  "resolution_status": "escalated",
  "action_taken": "Escalated to Orchestrator after 2 failed retries",
  "notified_user": true
}
```

### error_memory.json
```json
{
  "run_id": "",
  "errors_encountered": 0,
  "errors_resolved": 0,
  "errors_escalated": 0,
  "components_failed_twice": ["backend-agent"],
  "strategies_tried": [
    { "component": "", "strategy": "", "outcome": "success | failed", "timestamp": "" }
  ],
  "session_summary": ""
}
```

---

## Validation
Before writing `error_memory.json`, verify:
- [ ] Every entry in `error_log.json` has `resolution_status` that is not `"pending"`
- [ ] Every CRITICAL error has `notified_user: true`
- [ ] `components_failed_twice` is accurate — agents listed here must not be retried
- [ ] `error_memory.json` is written to `.claude/context/` not any other path

---

## Guidelines
- NEVER attempt to fix code-level bugs — redirect those entries to the Tester agent
- NEVER silently skip a CRITICAL error — always notify user before acting
- NEVER retry a strategy that already failed for the same component this run
- ALWAYS update `error_log.json` after every action — success or failure
- ALWAYS write `error_memory.json` before exiting each invocation
- ALWAYS classify severity before acting — never act on an unclassified error
- CRITICAL: If escalating, halt the pipeline completely — never continue
  silently after escalation. Control must return to the Orchestrator.
