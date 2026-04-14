---
name: executor-skill
description: |
  Teaches the Executor agent how to analyze a user prompt, score its
  complexity, dynamically select MCP tools based on signal detection,
  decide which agents to run or skip, group tasks to minimize token
  usage, and produce an optimized execution plan.
  Use this skill whenever the Executor agent is activated after
  manifest.json is written. Trigger this skill even for simple prompts
  — the skill defines rules for all complexity levels including low.
  TRIGGER when: execution planning needed, agent selection required,
  MCP tool selection, task grouping, token optimization, complexity scoring.
  DO NOT TRIGGER when: writing application code, designing architecture,
  generating UI, running tests, or fixing existing codebases.
---

# Executor Skill

## Overview
This skill defines how the Executor agent scores prompt complexity,
selects MCP tools dynamically from prompt signals, decides which agents
to run or skip, groups tasks for token efficiency, and produces the
execution plan that governs the entire pipeline. The output is a single
`execution_plan.json` that every downstream agent reads before starting.

## When to use
- Use when the Executor agent is activated after manifest.json is written
- Use when complexity scoring, agent selection, or MCP tool mapping is needed
- Use when task grouping decisions must be made for token optimization

Do NOT use when:
- Writing application code of any kind
- The Improve agent is active (improve mode has its own planning logic)
- Any specialist agent (backend, frontend, tester) is already running

---

## Workflow

### Step 1 — Read manifest.json
Read `.claude/context/manifest.json`. Extract:
- `app_name` — for plan identification
- `core_features` — primary signal source for agent and tool selection
- `prompt_complexity` — initial hint from Orchestrator (may be overridden)
- `integrations_implied` — explicit external service signals
- `mode` — must be `"build"` to proceed

If manifest.json is missing or malformed, write a CRITICAL entry to
`error_log.json` and halt — do not guess at inputs.

### Step 2 — Score complexity
Apply these rules in order. Use the first rule that matches:

**Low complexity** — all of the following are true:
- Fewer than 3 core features
- No authentication mentioned
- No external integrations implied
- Single entity or resource
- Examples: todo app, note taking app, contact list

**High complexity** — any of the following are true:
- Real-time features (websockets, live updates)
- Payment processing mentioned
- Multi-role authentication (admin, user, guest)
- 4 or more distinct entities
- External API integrations beyond simple CRUD
- Examples: e-commerce platform, project management tool, SaaS dashboard

**Medium complexity** — everything else:
- 2–3 entities, basic auth, standard CRUD with some extra features
- Examples: blog with comments, inventory tracker, booking system

Override the Orchestrator's initial `prompt_complexity` if your scoring
produces a different result. Document the override reason in the plan.

### Step 3 — Select MCP tools dynamically
Scan `core_features` and `integrations_implied` from manifest.json.
Apply the signal-to-tool mapping table:

| Signal found | Activate |
|-------------|----------|
| Any prompt (always) | Filesystem MCP |
| Any prompt (always) | GitHub MCP |
| "store", "save", "database", "persist", "records", "users", "auth" | Database MCP |
| "latest", "version", "docs", "research", "best practice" | Web Search MCP |
| "UI", "dashboard", "page", "component", "interface", "frontend" | Stitch MCP |
| "monitor", "errors", "production", "bugs", "track" | Sentry MCP |
| "slack", "message", "channel", "pinned", "notify" | Slack MCP |
| "notion", "wiki", "knowledge base", "document", "pages" | Notion MCP |
| "email", "gmail", "send mail", "inbox" | Gmail MCP |
| "calendar", "schedule", "event", "meeting", "reminder" | Calendar MCP |
| "search", "scrape", "browse", "fetch URL", "web" | Web Search MCP |

Activate ONLY the tools with a matching signal. Never activate a tool
with no signal. List all activated tools in `mcp_tools_activated`.

### Step 4 — Decide which agents to run

**Always run — never skip:**
- architect
- backend
- devops
- qa
- error_solver (spawned on demand — not sequential)

**Conditional — skip when signal is absent:**

| Agent | Run when | Skip when |
|-------|----------|-----------|
| frontend | UI, page, dashboard, component in features | API-only, CLI tool, no UI signal |
| tester | complexity is medium or high | complexity is low |

**Improve agent** — only in Improve Mode. Never in Build Mode.

For every skipped agent, write a clear reason in `agents_skipped`.

### Step 5 — Group tasks for token efficiency
Apply grouping only for low and medium complexity. Never group for high
complexity — reliability matters more than token savings at high complexity.

| Group name | Agents included | Apply when |
|------------|----------------|-----------|
| scaffold_group | architect + folder creation | Low complexity |
| data_group | backend models + DB schema | Low or medium |
| deploy_group | devops git init + README + env | Low or medium |

For each group, document the token saving reason in `task_groups`.

### Step 6 — Estimate token usage
Provide a rough token estimate for the full plan:

| Complexity | Estimate |
|------------|---------|
| Low | 8,000 – 15,000 tokens |
| Medium | 20,000 – 45,000 tokens |
| High | 50,000 – 100,000+ tokens |

Adjust upward if many MCP tools are activated or many integrations implied.

### Step 7 — Write execution_plan.json
Write to `.claude/context/execution_plan.json`.
Verify the file exists after writing before exiting.

---

## Output format

```json
{
  "plan_id": "exec_<timestamp>",
  "prompt_complexity": "low | medium | high",
  "complexity_override": false,
  "complexity_override_reason": "",
  "estimated_tokens": 0,
  "agents_to_run": [
    {
      "agent": "architect",
      "priority": 1,
      "grouped_with": null,
      "mcp_tools": ["filesystem", "web_search"]
    },
    {
      "agent": "backend",
      "priority": 2,
      "grouped_with": "data_group",
      "mcp_tools": ["filesystem", "database"]
    },
    {
      "agent": "frontend",
      "priority": 3,
      "grouped_with": null,
      "mcp_tools": ["filesystem", "stitch"]
    },
    {
      "agent": "tester",
      "priority": 4,
      "grouped_with": null,
      "mcp_tools": ["filesystem"]
    },
    {
      "agent": "devops",
      "priority": 5,
      "grouped_with": "deploy_group",
      "mcp_tools": ["filesystem", "github", "database"]
    },
    {
      "agent": "qa",
      "priority": 6,
      "grouped_with": null,
      "mcp_tools": ["filesystem", "sentry"]
    }
  ],
  "agents_skipped": [
    { "agent": "", "reason": "" }
  ],
  "mcp_tools_activated": [],
  "task_groups": [
    {
      "group_name": "",
      "agents_included": [],
      "token_saving_reason": ""
    }
  ],
  "optimizations_applied": []
}
```

---

## Validation
Before exiting, verify:
- [ ] `execution_plan.json` exists at `.claude/context/execution_plan.json`
- [ ] `architect`, `backend`, `devops`, `qa` are always in `agents_to_run`
- [ ] Every skipped agent has a documented reason
- [ ] `mcp_tools_activated` contains at least `filesystem` and `github`
- [ ] No tool is activated without a matching signal in the manifest
- [ ] `estimated_tokens` is a non-zero number

---

## Failure behaviour
If manifest.json is missing:
- Write CRITICAL to `error_log.json`
- Halt and return control to Orchestrator — do not produce a plan

If complexity cannot be determined:
- Default to `"medium"`
- Document the default in `complexity_override_reason`
- Continue — do not block the pipeline

---

## Guidelines
- NEVER activate an MCP tool without a matching signal in the manifest
- NEVER skip architect, backend, devops, or qa — they are mandatory
- NEVER apply task grouping for high complexity prompts
- ALWAYS write execution_plan.json before exiting
- ALWAYS document every skipped agent with a reason
- CRITICAL: execution_plan.json is the governing document for the entire
  pipeline — an incomplete or missing plan causes all downstream agents
  to run with wrong configurations or not at all
