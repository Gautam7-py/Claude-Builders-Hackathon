# Claude-Builders-Hackathon
# Claude Autonomous App Builder
### Claude Builders Club × APOGEE 2026 Hackathon

> **One prompt. One pipeline. A complete full-stack application.**

Claude Autonomous App Builder is a multi-agent orchestration system built on top of Claude Code. You describe an app in a single natural-language prompt, and the system autonomously plans, scaffolds, codes, tests, and deploys a complete full-stack application — without any further human input.

---

## Table of Contents

- [How It Works](#how-it-works)
- [Architecture Overview](#architecture-overview)
- [Agent Registry](#agent-registry)
- [MCP Tool Registry](#mcp-tool-registry)
- [Skills System](#skills-system)
- [Execution Flow](#execution-flow)
- [Context File Reference](#context-file-reference)
- [Default Tech Stack](#default-tech-stack)
- [Failure Handling](#failure-handling)
- [Setup & Prerequisites](#setup--prerequisites)
- [Usage](#usage)
- [Project Structure](#project-structure)
- [Output](#output)

---

## How It Works

The system is driven by a single master instruction file (`claude.md`) that turns Claude Code into an **Orchestrator** — the top-level coordinator of a network of specialized sub-agents. Each agent has a specific role, reads from a shared context directory, and writes its output back to that same directory so the next agent can pick up where the last one left off.

```
User Prompt → Orchestrator → Executor → Architect → Backend + Frontend
           → Tester → DevOps → QA → [Reconciler if needed]
           → Final Build Summary printed to user
```

The Orchestrator never writes application code itself. It only delegates.

---

## Architecture Overview

```
Claude X APOGEE Hackathon/
├── claude.md                          # Master orchestrator instructions
└── .claude/
    ├── agents/                        # Sub-agent YAML definitions
    │   ├── executor.yaml              # Execution planner
    │   ├── architect.yaml             # System designer
    │   ├── backend.yaml               # API + DB code generator
    │   ├── frontend.yaml              # React UI generator
    │   ├── tester.yaml                # Test suite writer
    │   ├── devops.yaml                # GitHub + environment setup
    │   ├── qa.yaml                    # Final QA + Sentry config
    │   ├── reconciler.yaml            # Architecture drift resolver
    │   └── error_solver.yaml          # System fault handler
    ├── mcp/                           # MCP server configurations
    │   ├── filesystem.json
    │   ├── github.json
    │   ├── git.json
    │   ├── memory.json
    │   ├── fetch.json
    │   └── sequentialthinking.json
    └── skills/                        # Agent knowledge libraries
        ├── architecture-skill/
        ├── backend-skill/
        ├── frontend-skill/
        ├── devops-skill/
        ├── testing-skill/
        ├── qa-skill/
        ├── reconciler-skill/
        ├── executor-skill/
        └── error-recovery-skill/
```

At runtime, the system creates a `.claude/context/` directory to hold all inter-agent shared state as JSON files.

---

## Agent Registry

There are **9 agents** in the system. The Executor decides which ones actually run based on the prompt.

| Agent | Priority | Role | Always Runs? |
|---|---|---|---|
| **Executor** | 1st | Reads the manifest, scores prompt complexity, selects agents, produces the execution plan | Yes |
| **Architect** | 2nd | Designs the complete technical blueprint: stack, folder structure, data models, API contracts, env vars | Yes |
| **Backend** | 3rd | Generates all server-side code — routes, controllers, models, middleware, DB schema | Yes |
| **Frontend** | 4th | Generates React components, pages, API hooks, and routing | Only if UI is needed |
| **Tester** | 5th | Writes backend endpoint tests, frontend hook tests, and integration tests | Only if complexity ≥ medium |
| **DevOps** | 6th | Initialises git, generates README + .gitignore, creates GitHub repo, pushes code | Yes |
| **QA** | 7th | Scans for critical issues, auto-patches, configures Sentry, prints build summary | Yes — always last |
| **Reconciler** | 8th (conditional) | Resolves drift between `architecture.json` and what was actually implemented | Only if QA detects drift |
| **Error Solver** | On-demand | Handles agent crashes, MCP failures, missing context files, sequencing errors | Only on failure |

### Agent Communication

Agents do not call each other directly. They communicate through JSON files in `.claude/context/`:

| File | Written By | Read By |
|---|---|---|
| `manifest.json` | Orchestrator | Executor, all agents |
| `execution_plan.json` | Executor | All agents |
| `architecture.json` | Architect | Backend, Frontend, Tester, DevOps, QA |
| `backend_done.json` | Backend | Frontend, Tester, DevOps, QA, Reconciler |
| `frontend_done.json` | Frontend | Tester, DevOps, QA, Reconciler |
| `test_report.json` | Tester | DevOps, QA |
| `devops_done.json` | DevOps | QA |
| `arch_drift.json` | QA | Reconciler |
| `reconciler_done.json` | Reconciler | QA |
| `qa_report.json` | QA | User (printed as build summary) |
| `error_log.json` | Error Solver | Error Solver (next invocation) |
| `error_memory.json` | Error Solver | Error Solver (next invocation) |
| `skip_log.json` | Orchestrator / QA | QA |

---

## MCP Tool Registry

Six MCP servers are configured. The Executor dynamically selects only those needed per run.

| Tool | Config | Purpose | Required? | Fallback |
|---|---|---|---|---|
| **filesystem** | `mcp/filesystem.json` | Read/write all files — used by every agent | **Yes — critical** | None. Pipeline halts if unavailable. |
| **github** | `mcp/github.json` | Create repos, commit, push. Needs `GITHUB_TOKEN` env var | No | DevOps generates `setup.sh` with manual git commands |
| **git** | `mcp/git.json` | Local git operations before push | No | DevOps writes commands to `setup.sh` |
| **memory** | `mcp/memory.json` | Persistent cross-agent memory store, prevents context drift | No | Agents use `.claude/context/` JSON files |
| **sequentialthinking** | `mcp/sequentialthinking.json` | Forces step-by-step reasoning for Executor, Architect, Reconciler | No | Agents use inline chain-of-thought |
| **fetch** | `mcp/fetch.json` | Read npm docs, package registries, public API specs | No | Agents use known stable versions from skill files |

---

## Skills System

Each agent has a corresponding skill file — a detailed knowledge library that defines exactly how that agent should operate. Agents read their skill file as the first step of their process.

| Skill | Used By | Covers |
|---|---|---|
| `architecture-skill` | Architect | Stack selection logic, folder structure conventions, data model schemas, API contract format |
| `backend-skill` | Backend | Code standards, file generation order, response shapes `{ success, data }`, validation patterns |
| `frontend-skill` | Frontend | React patterns, API hook conventions, Stitch MCP usage, Tailwind standards |
| `devops-skill` | DevOps | GitHub repo setup, git init, README generation, .gitignore rules, DB migration |
| `testing-skill` | Tester | Test modes (smoke/core/full), naming conventions `[subject] [condition] [expected]`, mock patterns |
| `qa-skill` | QA | Critical issue scan rules, auto-patch logic, Sentry configuration, build summary format |
| `reconciler-skill` | Reconciler | Drift classification (missing/extra/type_mismatch/phantom_call), resolution decision tree |
| `executor-skill` | Executor | Complexity scoring rules, MCP tool selection signals, task grouping logic |
| `error-recovery-skill` | Error Solver | Error classification (CRITICAL/WARNING/INFO), resolution strategies, escalation protocol |

---

## Execution Flow

### Startup

1. User provides a single natural-language prompt
2. Orchestrator reads and extracts: app domain, core features, constraints
3. Orchestrator makes reasonable assumptions on ambiguity — never asks for clarification
4. Orchestrator writes `manifest.json`
5. Orchestrator checks MCP tool availability and writes `mcp_status.json`
6. Orchestrator spawns the Executor

### Pipeline

```
manifest.json written
        │
        ▼
   [Executor]
   Scores complexity (low / medium / high)
   Selects agents + MCP tools
   Writes execution_plan.json
        │
        ▼
   [Architect]
   Reads manifest + execution plan
   Designs stack, folder structure, data models, API contracts
   Writes architecture.json
        │
        ▼
   [Backend]  ──────────────────  [Frontend] (parallel if supported)
   Generates API + DB code        Generates React UI
   Writes backend_done.json       Writes frontend_done.json
        │                                  │
        └──────────────┬───────────────────┘
                       ▼
                  [Tester]
                  Writes test suite (smoke / core / full)
                  Writes test_report.json
                       │
                       ▼
                  [DevOps]
                  Git init → GitHub repo → push
                  Writes devops_done.json
                       │
                       ▼
                  [QA] ◄── always last
                  Architecture drift check
                       │
                    drift? ──YES──► [Reconciler]
                       │                   │
                       │◄──────────────────┘
                       │
                  Code scan + auto-patch (max 2 rounds)
                  Sentry configuration
                  Writes qa_report.json
                  Prints build summary to user
```

### Complexity Scoring

| Level | Conditions | Example Prompts | Token Budget |
|---|---|---|---|
| **Low** | < 3 features, no auth, no external APIs, single entity | Todo app, note-taking app, contact list | Conservative — group tasks |
| **Medium** | Multiple entities, basic auth, 2–3 features | Blog with users and comments, inventory tracker | Balanced |
| **High** | Real-time features, payments, multi-role auth, 4+ entities | E-commerce platform, project management tool | Full — no grouping |

---

## Context File Reference

All inter-agent state is stored in `.claude/context/`. The Orchestrator adds `context/` to `.gitignore` — these files are build artifacts, not source files.

### manifest.json (written by Orchestrator)
```json
{
  "app_name": "",
  "app_domain": "",
  "core_features": [],
  "tech_stack": {
    "frontend": "React + Tailwind",
    "backend": "Node.js + Express",
    "database": "PostgreSQL"
  },
  "prompt_complexity": "low | medium | high"
}
```

### execution_plan.json (written by Executor)
```json
{
  "plan_id": "exec_<timestamp>",
  "prompt_complexity": "low | medium | high",
  "estimated_tokens": 0,
  "agents_to_run": [],
  "agents_skipped": [],
  "task_groups": [],
  "mcp_tools_activated": [],
  "optimizations_applied": []
}
```

### architecture.json (written by Architect)
```json
{
  "app_name": "",
  "stack": { "frontend": {}, "backend": {}, "database": {}, "auth": {} },
  "dependencies": { "frontend": [], "backend": [] },
  "folder_structure": {},
  "api_contracts": [],
  "data_models": [],
  "environment_variables": [],
  "architectural_decisions": [],
  "warnings": []
}
```

### qa_report.json (written by QA)
```json
{
  "status": "pass | pass_with_warnings | fail",
  "architecture_drift_detected": false,
  "reconciler_invoked": false,
  "critical_issues_found": 0,
  "critical_issues_fixed": 0,
  "patches_applied": [],
  "sentry": { "status": "configured | fallback | failed" },
  "final_status": "success | success_with_warnings | partial",
  "build_ready": true
}
```

---

## Default Tech Stack

The Architect selects the stack automatically based on the prompt. Defaults:

| Layer | Default | Alternatives |
|---|---|---|
| Frontend | React 18 + Vite + Tailwind CSS | Next.js (SSR/SEO), plain HTML (low complexity) |
| Backend | Node.js + Express | FastAPI (ML/data), Next.js API routes |
| Database | PostgreSQL | MongoDB (unstructured data), SQLite (local tools) |
| Auth | None (skipped if not mentioned) | JWT, OAuth2 |
| Testing | Jest + Supertest (backend), Vitest + RTL (frontend) | — |
| UI Components | Stitch MCP → Tailwind manual fallback | — |

---

## Failure Handling

The system is designed to degrade gracefully. No single agent failure stops the pipeline.

### Agent Failure Protocol

1. Log failure with timestamp and reason
2. Retry up to **2 times** with a 3-second wait
3. If all retries fail → skip agent, continue pipeline
4. Write skip report to `skip_log.json`
5. Print `⚠️ AGENT SKIPPED` notice to user with reason and impact

### MCP Failure Handling

| Tool | On Failure |
|---|---|
| `filesystem` | **CRITICAL** — pipeline halts immediately |
| `github` | DevOps generates `setup.sh` with manual push commands |
| `git` | DevOps writes git commands to `setup.sh` |
| `memory` | Agents use `.claude/context/` JSON files as fallback |
| `sequentialthinking` | Agents use inline reasoning |
| `fetch` | Agents use known stable package versions from skill files |
| `sentry` | QA adds commented Sentry stubs with setup instructions |

### Error Solver Agent

The Error Solver activates automatically whenever `error_log.json` contains an unresolved entry. It classifies errors into three levels:

- **CRITICAL** — notifies user immediately, halts pipeline if unresolvable
- **WARNING** — resolved silently, logs action taken
- **INFO** — logged only, no action needed

It maintains an `error_memory.json` to ensure it never repeats a failed resolution strategy.

---

## Setup & Prerequisites

### Requirements

- [Claude Code](https://claude.ai/code) (latest version)
- Node.js 20+
- `GITHUB_TOKEN` environment variable (optional — for automatic GitHub push)

### Installation

```bash
# Clone or copy the project files
git clone <this-repo>
cd "Claude X APOGEE Hackathon"

# Set your GitHub token (optional)
export GITHUB_TOKEN=your_token_here

# Open in Claude Code
claude
```

### MCP Server Setup

The MCP servers are configured in `.claude/mcp/` and are launched automatically by Claude Code. All use `npx` so no separate installation is needed. On first run, `npx` will download each server package automatically.

To verify MCP tool availability, the Orchestrator runs a lightweight probe on each tool at startup and writes results to `.claude/context/mcp_status.json`.

---

## Usage

Open Claude Code in the project directory and provide a single natural-language prompt describing your app:

```
Build a multi-tenant project management app with workspaces, task boards,
user invitations, and role-based access control. Include a REST API
and a React dashboard.
```

The system takes it from there. No further input is required.

### What the Orchestrator Will NOT Do

- Ask for clarification — it makes reasonable assumptions and documents them
- Write application code itself — it only delegates to agents
- Skip the final build summary — QA always prints it

---

## Project Structure

After the pipeline runs, your generated application lands in `/output/`:

```
/output/
├── backend/
│   ├── src/
│   │   ├── routes/
│   │   ├── controllers/
│   │   ├── models/
│   │   └── middleware/
│   ├── tests/
│   ├── index.js
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── api/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── pages/
│   ├── index.html
│   └── package.json
├── .env.example
├── .gitignore
└── README.md
```

---

## Output

When the pipeline completes, the QA agent prints a build summary:

```
╔══════════════════════════════════════════════════════╗
║           ✅  BUILD COMPLETE                         ║
╠══════════════════════════════════════════════════════╣
║  App:            <app_name>                          ║
║  GitHub:         <repo_url>                          ║
║  Stack:          <frontend> + <backend> + <database> ║
╠══════════════════════════════════════════════════════╣
║  AGENTS                                              ║
║  Ran:            <list of agents that ran>           ║
║  Skipped:        <agent: reason> or None             ║
╠══════════════════════════════════════════════════════╣
║  QUALITY                                             ║
║  Tests:          <passed>/<total> passing            ║
║  Issues fixed:   <count>                             ║
║  Warnings:       <count>                             ║
║  Sentry:         configured | setup instructions     ║
╠══════════════════════════════════════════════════════╣
║  NEXT STEPS                                          ║
║  1. Clone repo and run: cp .env.example .env         ║
║  2. Fill in environment variables                    ║
║  3. Run: npm install && npm run dev                  ║
╚══════════════════════════════════════════════════════╝
```

---

## Rules the Orchestrator Always Follows

- Never ask the user for more input after the initial prompt
- Never write application code — always delegates to agents
- Always run the Executor before any other agent
- Always validate each agent's output file exists before spawning the next
- Retry failed agents exactly 2 times before skipping
- Always notify the user when an agent is skipped with a reason and impact
- Final output must be a working full-stack app in `/output/`

---

## Success Condition

The system has succeeded when:

- `/output/` contains a complete frontend + backend codebase
- A GitHub repo has been created and code has been pushed
- The Tester agent has confirmed core functionality works
- Sentry has been configured (or stubs added with instructions)
- A full build summary has been printed to the user

---

*Built for Claude Builders Club × APOGEE 2026 Hackathon*
