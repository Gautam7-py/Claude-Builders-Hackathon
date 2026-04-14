---
name: testing-skill
description: |
  Teaches the Tester agent the exact test writing methodology, naming
  conventions, mock patterns, and coverage priorities for generating a
  complete test suite for an autonomously generated application.
  Use this skill whenever the Tester agent is activated to write backend
  endpoint tests, frontend hook tests, or integration tests. Trigger this
  skill even when test mode is smoke-only — it defines all three modes.
  TRIGGER when: test file generation, unit tests, integration tests, endpoint
  testing, hook testing, coverage needed, test suite setup required.
  DO NOT TRIGGER when: writing application code, designing architecture,
  generating frontend or backend files, or running DevOps tasks.
compatibility: Jest 29+, Supertest 6+, Vitest 1+, React Testing Library 14+
---

# Testing Skill

## Overview
This skill defines the test writing methodology, file structure, naming
conventions, and mock patterns the Tester agent must follow. Tests are
generated in one of three modes — smoke, core, or full — determined by
the complexity level in `execution_plan.json`. The output is a set of test
files and a `test_report.json` confirming coverage and results.

## When to use
- Use when the Tester agent generates backend endpoint tests
- Use when frontend hook or component tests are being written
- Use when integration or end-to-end flow tests are needed

Do NOT use when:
- Writing or modifying application code
- Designing architecture or data models
- Running DevOps tasks or Sentry configuration

---

## Workflow

### Step 1 — Determine test mode
Read `.claude/context/execution_plan.json` for `prompt_complexity`:

| Complexity | Test mode | Scope |
|------------|-----------|-------|
| `"low"` | Smoke | Health check + 1 CRUD happy path per entity |
| `"medium"` | Core | All smoke + happy + error cases + 1 integration per entity |
| `"high"` | Full | All core + auth flows + edge cases + E2E user journeys |

### Step 2 — Read inputs
Read before writing any test:
- `.claude/context/architecture.json` — api_contracts, data_models, auth strategy
- `.claude/context/backend_done.json` — `endpoints_implemented` list
- `.claude/context/frontend_done.json` — `hooks_generated` list
- `.claude/context/manifest.json` — core_features for coverage mapping

### Step 3 — Write backend tests first
Priority order: endpoints → validation → error handling → auth.

For every entry in `backend_done.endpoints_implemented` with status
`"implemented"`, write tests in this order:

**Test 1 — Happy path** (every endpoint, all modes)
```javascript
it('GET /api/users returns list of users', async () => {
  const res = await request(app).get('/api/users');
  expect(res.status).toBe(200);
  expect(res.body.success).toBe(true);
  expect(Array.isArray(res.body.data)).toBe(true);
});
```

**Test 2 — Missing required field** (POST/PUT only, core + full modes)
```javascript
it('POST /api/users with missing email returns 400', async () => {
  const res = await request(app).post('/api/users').send({ name: 'Test' });
  expect(res.status).toBe(400);
  expect(res.body.success).toBe(false);
  expect(res.body.error).toBeDefined();
});
```

**Test 3 — Not found** (GET/:id, PUT/:id, DELETE/:id — core + full modes)
```javascript
it('GET /api/users/:id with non-existent id returns 404', async () => {
  const res = await request(app).get('/api/users/nonexistent-id-999');
  expect(res.status).toBe(404);
  expect(res.body.success).toBe(false);
});
```

### Step 4 — Write integration tests (core + full modes only)
One integration test per entity covering the full CRUD lifecycle:

```javascript
describe('User integration — create → read → update → delete', () => {
  let createdId;

  it('creates a user', async () => {
    const res = await request(app)
      .post('/api/users')
      .send({ email: 'test@example.com', name: 'Test User' });
    expect(res.status).toBe(201);
    createdId = res.body.data.id;
  });

  it('reads the created user', async () => {
    const res = await request(app).get(`/api/users/${createdId}`);
    expect(res.status).toBe(200);
    expect(res.body.data.id).toBe(createdId);
  });

  it('updates the user', async () => {
    const res = await request(app)
      .put(`/api/users/${createdId}`)
      .send({ name: 'Updated Name' });
    expect(res.status).toBe(200);
  });

  it('deletes the user', async () => {
    const res = await request(app).delete(`/api/users/${createdId}`);
    expect(res.status).toBe(200);
  });
});
```

### Step 5 — Write frontend hook tests (core + full modes only)
For every hook in `frontend_done.hooks_generated`, mock the API client:

```javascript
// tests/useUsers.test.js
import { renderHook, act } from '@testing-library/react';
import { vi } from 'vitest';
import { useUsers } from '../src/hooks/useUsers';
import client from '../src/api/client';

vi.mock('../src/api/client');

it('useUsers fetchAll calls GET /api/users', async () => {
  client.get.mockResolvedValue({ data: { data: [] } });
  const { result } = renderHook(() => useUsers());
  await act(async () => { await result.current.fetchAll(); });
  expect(client.get).toHaveBeenCalledWith('/users');
  expect(result.current.loading).toBe(false);
});

it('useUsers fetchAll sets error on failure', async () => {
  client.get.mockRejectedValue({ response: { data: { error: 'Server error' } } });
  const { result } = renderHook(() => useUsers());
  await act(async () => { await result.current.fetchAll(); });
  expect(result.current.error).toBe('Server error');
});
```

### Step 6 — Write auth tests (full mode only)
If `architecture.auth.strategy` is not `"none"`:

```javascript
it('protected route without token returns 401', async () => {
  const res = await request(app).put('/api/users/some-id').send({ name: 'X' });
  expect(res.status).toBe(401);
  expect(res.body.success).toBe(false);
});

it('protected route with valid token succeeds', async () => {
  const token = generateTestToken(); // helper that creates a valid JWT
  const res = await request(app)
    .put('/api/users/some-id')
    .set('Authorization', `Bearer ${token}`)
    .send({ name: 'Updated' });
  expect(res.status).not.toBe(401);
});
```

### Step 7 — Name every test correctly
Pattern: `"[subject] [condition] [expected result]"`

| Good | Bad |
|------|-----|
| `GET /api/users returns list of users` | `test 1` |
| `POST /api/tasks with missing title returns 400` | `it works` |
| `useUsers fetchAll sets loading to false after response` | `should fetch` |

Never use: "test 1", "it works", "should work", "test case".

### Step 8 — Document bugs found
If application code has a bug discovered while writing tests:
- Do NOT fix the application code
- Add an entry to `known_gaps` in test_report.json:
  ```json
  { "description": "PUT /api/users returns 500 instead of 404", "affected_file": "users.controller.js", "type": "bug" }
  ```
- Continue writing remaining tests

### Step 9 — Write test_report.json
Write to `.claude/context/test_report.json` after all test files are generated.

---

## Output format

```json
{
  "test_mode": "smoke | core | full",
  "summary": {
    "total": 0,
    "passed": 0,
    "failed": 0,
    "skipped": 0
  },
  "test_files": [
    {
      "file": "tests/users.smoke.test.js",
      "tests_count": 5,
      "tests": [
        { "name": "GET /api/users returns list of users", "status": "pass", "error": null, "duration_ms": 12 }
      ]
    }
  ],
  "core_features_covered": ["user registration", "post creation"],
  "known_gaps": [
    { "description": "", "affected_file": "", "type": "bug | missing_coverage | untestable" }
  ],
  "recommendations_for_qa": [],
  "timestamp": ""
}
```

---

## Validation
Before writing `test_report.json`, verify:
- [ ] Every implemented endpoint has at least one test
- [ ] Every `core_features` item from manifest has at least one test
- [ ] All mocks are properly reset between tests (use `beforeEach` with `vi.clearAllMocks()`)
- [ ] No test imports a live database or real server — all external dependencies mocked
- [ ] All test names follow the `[subject] [condition] [expected result]` pattern

---

## Guidelines
- NEVER modify application code — only write test files
- NEVER write tests that depend on each other's state or execution order
- ALWAYS mock external dependencies — never hit a real server or database
- ALWAYS test at least one failure case per endpoint — happy path alone is not enough
- ALWAYS write `test_report.json` before exiting — even if all tests fail
- NEVER use generic test names — always follow the naming pattern
- CRITICAL: If you cannot write a test for something, document it in `known_gaps`
  with type `"untestable"` and a clear reason — never silently skip coverage
