---
name: sdk-generator
description: >
  Autonomous SDK generator that discovers API endpoints from a live website or HAR archive,
  plans and builds typed SDKs in multiple languages, and continuously updates them as the
  API evolves. Use this skill whenever the user wants to reverse-engineer an API, generate
  client SDKs from observed traffic, build API wrappers from HAR files, create typed clients
  from web service exploration, or keep an SDK in sync with a changing API. Also trigger when
  the user mentions "SDK generation", "API client from HAR", "reverse engineer API",
  "auto-generate SDK", "crawl API and build client", or any variation of turning observed
  HTTP traffic into usable code libraries.
---

# SDK Generator

An autonomous skill that discovers API surfaces from live websites or HAR archives, plans
SDK functions with LLM-assisted review, generates typed SDKs in multiple languages with
full test suites, validates everything end-to-end, and manages the entire lifecycle through
Git — including continuous updates when the upstream API changes.

## Quick Reference

| Phase | What Happens | Reference |
|-------|-------------|-----------|
| **Discovery** | Crawl site or parse HAR to find endpoints | `references/discovery.md` |
| **Planning** | Design SDK functions, get LLM peer review | `references/planning.md` |
| **Generation** | Implement SDK code per language target | `references/generation.md` |
| **Testing** | Unit tests, mocks, e2e validation, auto-fix | `references/testing.md` |
| **Git Workflow** | Branching, commits, PRs, versioning | `references/git-workflow.md` |
| **Update Cycle** | Re-crawl, diff, patch, re-test | `references/update-cycle.md` |

---

## Prerequisites

Before starting, verify the following tools are available:

```bash
# Required
git --version
python3 --version
node --version
npm --version

# Install if missing
pip install requests httpx pydantic pytest pytest-asyncio respx --break-system-packages
npm install -g typescript jest ts-jest @types/jest axios
```

---

## Workflow Overview

### Phase 0: Initialize Project

1. Create the SDK repository structure (see `references/repo-structure.md`)
2. Initialize Git with `main` branch and proper `.gitignore`
3. Set up language-specific project scaffolding (pyproject.toml, package.json, etc.)
4. Create initial commit: `chore: initialize SDK project scaffold`

### Phase 1: API Discovery

Read `references/discovery.md` for the full procedure.

Two entry paths, same output:

**Path A — Live Website:**
1. Accept a base URL from the user
2. Use browser automation or HTTP crawling to navigate the site
3. Intercept and record all API calls (XHR/fetch requests)
4. Capture: method, URL pattern, headers, query params, request body, response body, status codes
5. Identify authentication patterns (Bearer tokens, API keys, cookies, OAuth flows)

**Path B — HAR Archive:**
1. Accept a .har file from the user
2. Parse all entries, filtering to API-like requests (JSON responses, REST patterns)
3. Extract the same data points as Path A
4. Group by logical endpoint (normalize URL path parameters)

**Output:** An `api-surface.json` file containing every discovered endpoint with:
- Method + URL pattern (with path params identified like `/users/{id}`)
- Request/response schemas (inferred JSON schemas)
- Required headers and auth patterns
- Observed status codes and error shapes
- Pagination patterns if detected

### Phase 2: Function Planning

Read `references/planning.md` for the full procedure.

This is the most critical phase. Every API endpoint becomes a planned SDK function.

1. **Generate the plan:** For each endpoint in `api-surface.json`, produce:
   - Function name (language-idiomatic: `snake_case` for Python, `camelCase` for TS/JS)
   - Parameter list with types (inferred from request schema)
   - Return type (inferred from response schema)
   - Error handling strategy
   - Whether it needs pagination support
   - Mock/stub specification for testing

2. **LLM Peer Review:** The plan is reviewed by a second LLM call acting as a senior
   API design reviewer. The reviewer checks for:
   - Naming consistency and idiomaticness
   - Missing error cases
   - Type safety gaps
   - Pagination/retry patterns that should be included
   - Authentication flow correctness
   - Breaking the plan into logical resource groupings (e.g., `users.*`, `projects.*`)

   The review produces a structured critique with `approve`, `suggest`, or `reject`
   verdicts per function. All `reject` items must be addressed before proceeding.
   `suggest` items should be addressed but can be deferred with justification.

3. **Finalize:** Merge review feedback into the plan. Save as `sdk-plan.json`.

4. **Git:** Commit the plan on a feature branch:
   ```
   git checkout -b feat/initial-sdk-plan
   git add api-surface.json sdk-plan.json
   git commit -m "docs: add API surface discovery and SDK function plan"
   ```

### Phase 3: SDK Generation

Read `references/generation.md` for the full procedure.

For each target language, generate the SDK from the approved plan:

1. **Models/Types:** Generate typed data classes / interfaces from response schemas
2. **Client Class:** Base HTTP client with auth, retry, error handling
3. **Resource Modules:** Group functions by resource (users, projects, etc.)
4. **Each Function:** Implement according to the plan — typed params, return types,
   docstrings, error handling, pagination wrappers where needed
5. **Mocks & Stubs:** For every function, generate:
   - A mock response factory (returns realistic fake data matching the schema)
   - A request stub that validates the outgoing request shape
   - A fixture file with sample payloads

6. **Git:** Each resource module is committed separately:
   ```
   git add python/src/promptql/users.py python/tests/test_users.py python/tests/mocks/users.py
   git commit -m "feat(python): add users resource with tests and mocks"
   ```

### Phase 4: Testing

Read `references/testing.md` for the full procedure.

Testing happens in two layers, and bugs are fixed automatically:

**Layer 1 — Unit Tests (offline):**
- Every SDK function has unit tests using mocks/stubs
- Tests validate: correct HTTP method/URL, proper serialization of params,
  correct deserialization of responses, error handling paths
- Run: `pytest python/tests/` and `npx jest` for TypeScript
- If any test fails → read the error → fix the code → re-run → repeat until green

**Layer 2 — End-to-End Tests (if live endpoint available):**
- For each function, make a real API call (if safe — GET endpoints, read-only operations)
- Validate response matches expected schema
- Capture any discrepancies as schema refinements
- Run: `pytest python/tests/e2e/` with `--e2e` flag
- If any e2e test fails → diagnose whether it's an SDK bug or a schema inaccuracy →
  fix accordingly → re-run → repeat until green

**Auto-fix loop:**
```
MAX_FIX_ATTEMPTS = 5
for attempt in range(MAX_FIX_ATTEMPTS):
    result = run_tests()
    if result.all_passed:
        break
    for failure in result.failures:
        diagnose(failure)
        apply_fix(failure)
    git commit -m "fix: {description of what was fixed}"
```

If after MAX_FIX_ATTEMPTS tests still fail, create a GitHub issue or TODO comment
and continue with remaining endpoints. Do not block the entire SDK on one stubborn test.

**Git:** After all tests pass:
```
git add .
git commit -m "test: all unit and e2e tests passing"
git checkout main
git merge feat/initial-sdk-plan --no-ff -m "feat: initial SDK release"
git tag v0.1.0
```

### Phase 5: Update Cycle

Read `references/update-cycle.md` for the full procedure.

When the user triggers an update (or on a schedule):

1. **Re-discover:** Run Phase 1 again, producing `api-surface-new.json`
2. **Diff:** Compare against the existing `api-surface.json`:
   - **New endpoints** → plan + review + generate + test (mini Phase 2-4)
   - **Changed endpoints** (new fields, changed types) → update types + functions + tests
   - **Removed endpoints** → deprecate functions, add `@deprecated` decorators, update tests
3. **Branch:** `feat/api-update-YYYY-MM-DD`
4. **Plan new/changed functions** with LLM review (same as Phase 2)
5. **Generate/update code** with full test suite
6. **Run all tests** (existing + new) — auto-fix loop
7. **Merge + tag:** Bump version appropriately (minor for additions, major for breaking changes)

---

## Multi-Language Support

The skill generates SDKs for these targets by default (user can customize):

| Language | Directory | Package Manager | Test Framework |
|----------|-----------|----------------|----------------|
| Python | `python/` | pip / pyproject.toml | pytest |
| TypeScript | `typescript/` | npm / package.json | jest |

Additional languages can be added by creating a new generation template
in `templates/` and adding the language config to `sdk-plan.json`.

---

## Error Handling Philosophy

- **Never block on a single failure.** If one endpoint's tests can't be fixed after
  MAX_FIX_ATTEMPTS, mark it as `status: broken` in the plan, commit what works,
  and move on.
- **Always commit working state.** Every commit should leave the repo in a state
  where existing tests pass.
- **Surface unknowns.** If the discovery phase can't determine a type or pattern,
  use the most permissive type (`Any` / `unknown`) and add a `TODO` comment.

---

## Coordinator Responsibilities

The agent orchestrating this skill must:

1. **Follow phases in order** — Discovery → Planning → Generation → Testing → Git finalize
2. **Never skip LLM review** — The planning review catches design mistakes early
3. **Commit atomically** — Each logical unit of work gets its own commit
4. **Run the full test suite before merging** — No merge without green tests
5. **Preserve the API surface file** — This is the source of truth for diffing on updates
6. **Use conventional commits** — `feat:`, `fix:`, `test:`, `docs:`, `chore:` prefixes
7. **Tag releases** — SemVer: patch for fixes, minor for new endpoints, major for breaking changes
