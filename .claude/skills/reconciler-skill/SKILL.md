---
name: reconciler-skill
description: >
  Targeted architecture-to-implementation drift correction. Use only when the QA
  agent has confirmed a mismatch between architecture.json and backend_done.json
  or frontend_done.json. Provides decision rules for resolving each discrepancy
  type without over-engineering the fix.
triggers:
  - reconciler.yaml agent is active
  - arch_drift.json exists with unresolved entries
exclusions:
  - Do not use during normal code generation
  - Do not use to add features
  - Do not use to refactor passing code
---

## Value Proposition

When agents generate code independently, small mismatches emerge — the Architect
specifies an endpoint the Backend forgets, the Frontend calls a route that was
renamed, a schema field changes between planning and implementation. This skill
gives the Reconciler agent a clear, decision-tree approach to resolving these
diffs without touching anything beyond the specific gap.

---

## Resolution Decision Tree

```
Is the discrepancy a missing endpoint?
  → YES: Add a minimal stub implementation to the Backend.
         A stub is: correct route path, correct HTTP method, returns
         { success: true, data: null, message: "not yet implemented" }
         Update backend_done.json.

Is the discrepancy an extra endpoint (not in architecture.json)?
  → YES: Add it to architecture.json as an addendum section.
         Do NOT remove it from the Backend — removal breaks things.
         Update architecture.json with: { "addendum_endpoints": [...] }

Is the discrepancy a type mismatch (schema differs)?
  → YES: Read both versions. The architecture.json schema is authoritative.
         Update the implementation to match the contract, not the reverse.
         Affects: Backend handler + Frontend call + test assertions.

Is the discrepancy a Frontend phantom call?
  → YES: Update the Frontend call to use the correct endpoint path
         from backend_done.endpoints_implemented.
         Do NOT remove the functionality — reroute it correctly.
```

---

## Patch Format

Every change must be documented as:

```json
{
  "file": "relative/path/from/output",
  "line_range": [1, 10],
  "change_type": "missing_endpoint | extra_endpoint | type_mismatch | phantom_call",
  "before": "original content summary",
  "after": "replacement content summary"
}
```

---

## ALWAYS

- Read both sides of every discrepancy before deciding — never assume
- Use sequentialthinking MCP to reason through resolution before acting
- Write reconciler_done.json with full change log
- Leave architecture.json as the single source of truth

## NEVER

- Delete implemented endpoints — always stub or redirect
- Modify the Architect's core design decisions
- Patch more than the specific discrepancy
- Skip writing reconciler_done.json

## CRITICAL

The QA agent waits for `reconciler_done.json` before continuing. If you exit
without writing it, the QA agent will hang. Always write it — even if the
status is "failed" or "nothing_to_reconcile".
