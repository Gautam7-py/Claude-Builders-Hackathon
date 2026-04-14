---
name: architecture-skill
description: |
  Teaches the Architect agent how to design a complete technical foundation
  for an autonomously generated application — covering stack selection, folder
  structure, data models, API contracts, and environment variable definitions.
  Use this skill whenever the Architect agent is activated and needs to produce
  architecture.json. Trigger this skill even if the app type is ambiguous —
  the skill contains decision rules for all cases.
  TRIGGER when: architect agent runs, stack decision needed, folder structure
  design, API contract definition, data model design, system blueprint needed.
  DO NOT TRIGGER when: writing application code, generating UI components,
  running tests, or performing DevOps tasks.
---

# Architecture Skill

## Overview
This skill defines how the Architect agent designs the complete technical
blueprint for a generated application. It covers stack selection logic,
folder structure conventions, data model schemas, API contract formats, and
decision documentation. The output is a single `architecture.json` file that
all downstream agents read and follow exactly.

## When to use
- Use when the Architect agent is activated after execution_plan.json is written
- Use when stack decisions, folder structure, or API contracts must be defined
- Use when data models need to be derived from manifest core_features

Do NOT use when:
- Writing actual application code (HTML, CSS, JS, Python, SQL)
- Generating UI components or frontend pages
- Running test suites or DevOps setup tasks

---

## Workflow

### Step 1 — Read inputs
Read both input files before making any decision:
- `.claude/context/manifest.json` — extract: app_name, app_domain,
  core_features, prompt_complexity
- `.claude/context/execution_plan.json` — extract: agents_to_run,
  agents_skipped, mcp_tools_activated, prompt_complexity

If either file is missing or malformed, write a `warnings` entry in
architecture.json and continue with reasonable defaults based on app_name.

### Step 2 — Select the stack
Apply this decision table in order. Stop at the first matching rule.

**Frontend stack:**

| Condition | Choice |
|-----------|--------|
| frontend in agents_skipped | `"none"` |
| prompt mentions SSR, SEO, or blog | Next.js 14 + Tailwind CSS |
| complexity is low | Plain HTML + CSS + vanilla JS |
| default | React 18 + Vite + Tailwind CSS |

**Backend stack:**

| Condition | Choice |
|-----------|--------|
| no backend mention in features | `"none"` |
| ML, data science, or scientific features | FastAPI (Python 3.11) |
| frontend is Next.js | Next.js API routes |
| default | Node.js 20 + Express 4 |

**Database:**

| Condition | Choice |
|-----------|--------|
| no data persistence in features | `"none"` |
| document-based or schema-less data | MongoDB + Mongoose |
| complexity is low, local tool | SQLite + better-sqlite3 |
| default | PostgreSQL 16 + Prisma ORM |

**Authentication:**

| Condition | Choice |
|-----------|--------|
| auth not mentioned in features | `"none"` |
| "login with Google" or "login with GitHub" | OAuth2 via Passport.js |
| auth mentioned, no OAuth | JWT via jsonwebtoken |

Use Web Search MCP to confirm the latest stable version of each chosen
package. Run a maximum of 3 searches. If Web Search MCP is unavailable,
use the versions listed in this table and add a `warnings` entry.

### Step 3 — Define the folder structure
Produce the complete `/output/` folder tree. Follow this base structure
and extend it based on the chosen stack:

```
/output/
├── frontend/               ← Client-side code (omit if frontend is "none")
│   ├── src/
│   │   ├── components/     ← Reusable UI components
│   │   ├── pages/          ← One file per application page
│   │   ├── hooks/          ← Custom React hooks (one per entity)
│   │   ├── api/            ← API client and axios instance
│   │   └── App.jsx         ← Root component with routing
│   ├── public/             ← Static assets
│   ├── index.html
│   ├── vite.config.js
│   ├── tailwind.config.js
│   └── package.json
├── backend/                ← Server-side code (omit if backend is "none")
│   ├── src/
│   │   ├── routes/         ← Express route files (one per entity)
│   │   ├── controllers/    ← Business logic (one per entity)
│   │   ├── models/         ← ORM models (one per entity)
│   │   ├── middleware/     ← Auth, error handler, logger, CORS
│   │   └── index.js        ← Entry point — mounts all routes
│   ├── prisma/             ← Prisma schema and migrations (if PostgreSQL)
│   │   └── schema.prisma
│   ├── .env.example
│   └── package.json
└── README.md
```

Every directory must have a one-line comment in the architecture.json
`folder_structure` field. Agents use this to know exactly where to write files.

### Step 4 — Design data models
Derive one model per entity mentioned in manifest `core_features`.
Every model must include:
- `id` — primary key (UUID or auto-increment integer)
- `created_at` — timestamp
- `updated_at` — timestamp
- All feature-specific fields with name, type, and constraints
- All relationships to other models with the relationship type

Use this field type vocabulary consistently:

| Concept | Type string |
|---------|------------|
| Short text | `"string"` |
| Long text | `"text"` |
| Whole number | `"integer"` |
| Decimal | `"float"` |
| True/false | `"boolean"` |
| Date and time | `"timestamp"` |
| File reference | `"string"` (store URL) |
| Foreign key | `"uuid"` or `"integer"` |

### Step 5 — Define API contracts
List every endpoint the backend must implement. Derive endpoints from
`core_features` — one resource per entity, standard CRUD by default.

For every endpoint, define:
- `method` — GET, POST, PUT, DELETE
- `path` — RESTful path, e.g. `/api/users/:id`
- `description` — one sentence
- `auth_required` — true or false
- `request_body` — JSON shape (null for GET/DELETE)
- `response_shape` — always `{ success, data }` or `{ success, error }`

No agent may add or remove endpoints from this list. If an endpoint is
missing, the Orchestrator must re-trigger the Architect — not patch inline.

### Step 6 — Define environment variables
List every environment variable required by the application. For each:
- `key` — the variable name in UPPER_SNAKE_CASE
- `description` — one line explaining what it holds
- `example_value` — a safe placeholder (never a real secret)
- `required` — true or false

### Step 7 — Document all decisions
Write one `architectural_decisions` entry for every non-obvious choice.
Each entry must contain:
- `decision` — what was chosen
- `reason` — why, traced to a manifest field
- `alternatives_considered` — what else was evaluated

Judges will read this section. Make reasoning explicit and traceable.

### Step 8 — Write architecture.json
Write the complete output to `.claude/context/architecture.json`.
Validate that all required fields are present before writing.
After writing, verify the file exists using Filesystem MCP.

---

## Output format

```json
{
  "app_name": "",
  "stack": {
    "frontend": { "framework": "", "version": "", "styling": "", "build_tool": "" },
    "backend": { "runtime": "", "framework": "", "version": "" },
    "database": { "type": "", "name": "", "version": "", "orm": "" },
    "auth": { "strategy": "none | jwt | oauth2", "provider": "" }
  },
  "dependencies": {
    "frontend": ["package@version"],
    "backend": ["package@version"]
  },
  "folder_structure": {
    "/output/frontend/src/components/": "Reusable UI components",
    "/output/backend/src/routes/": "Express route handlers"
  },
  "api_contracts": [
    {
      "method": "POST",
      "path": "/api/users",
      "description": "Create a new user account",
      "auth_required": false,
      "request_body": { "email": "string", "password": "string" },
      "response_shape": { "success": true, "data": { "id": "uuid" } }
    }
  ],
  "data_models": [
    {
      "name": "User",
      "fields": [
        { "name": "id", "type": "uuid", "constraints": "primary key" },
        { "name": "email", "type": "string", "constraints": "unique, not null" },
        { "name": "created_at", "type": "timestamp", "constraints": "not null" }
      ],
      "relationships": [
        { "type": "has_many", "model": "Post", "foreign_key": "user_id" }
      ]
    }
  ],
  "environment_variables": [
    {
      "key": "DATABASE_URL",
      "description": "PostgreSQL connection string",
      "example_value": "postgresql://user:password@localhost:5432/dbname",
      "required": true
    }
  ],
  "architectural_decisions": [
    {
      "decision": "PostgreSQL over MongoDB",
      "reason": "Core features include relational data with foreign keys",
      "alternatives_considered": "MongoDB — rejected because schema is fixed"
    }
  ],
  "warnings": []
}
```

---

## Validation
Before exiting, verify:
- [ ] architecture.json exists at `.claude/context/architecture.json`
- [ ] All `core_features` from manifest are represented in at least one
      data model or API contract
- [ ] Every agent in `agents_to_run` has the folder paths it needs defined
      in `folder_structure`
- [ ] No real secrets appear in `environment_variables.example_value`
- [ ] Every API contract has both `request_body` and `response_shape` defined
- [ ] `architectural_decisions` has at least one entry per major stack choice

If any check fails, fix the issue before writing the file.

---

## Guidelines
- NEVER write application code — no HTML, JS, Python, SQL, or config files
- NEVER invent requirements not present in manifest.json
- ALWAYS document every non-obvious technology decision in `architectural_decisions`
- ALWAYS keep the design as simple as the manifest allows — do not over-engineer
- ALWAYS set `stack.frontend` to `"none"` explicitly if frontend is skipped
- ALWAYS set `stack.backend` to `"none"` explicitly if no backend is needed
- NEVER ask the user for clarification — make a documented assumption instead
- CRITICAL: architecture.json is the single source of truth for all downstream
  agents — it must be complete and correct before exiting
