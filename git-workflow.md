# Git Workflow

This document defines the Git workflow for the SDK project. Every action in the
skill that produces code must follow these conventions.

## Table of Contents
- [Branch Strategy](#branch-strategy)
- [Commit Conventions](#commit-conventions)
- [Workflow by Phase](#workflow-by-phase)
- [Versioning and Tags](#versioning-and-tags)
- [Merge Rules](#merge-rules)
- [Conflict Resolution](#conflict-resolution)
- [Repository Initialization](#repository-initialization)

---

## Branch Strategy

```
main                          ← Always deployable, tagged releases only
  │
  ├── feat/initial-sdk-plan   ← Discovery + planning phase
  │
  ├── feat/python-sdk         ← Python SDK generation + tests
  │
  ├── feat/typescript-sdk     ← TypeScript SDK generation + tests
  │
  ├── feat/api-update-2026-02-15  ← Update cycle (dated)
  │
  └── fix/users-pagination    ← Bug fixes discovered post-merge
```

### Branch Naming

| Type | Pattern | Example |
|------|---------|---------|
| Feature / new SDK | `feat/{description}` | `feat/python-sdk` |
| API update cycle | `feat/api-update-YYYY-MM-DD` | `feat/api-update-2026-02-15` |
| Bug fix | `fix/{description}` | `fix/users-pagination` |
| Chore / tooling | `chore/{description}` | `chore/update-pytest` |
| Documentation | `docs/{description}` | `docs/add-usage-examples` |

### Rules

1. **Never commit directly to `main`.** All changes go through feature branches.
2. **One logical unit per branch.** Don't mix Python SDK work with TypeScript in the same branch.
3. **Branch from `main`.** Always start from the latest `main`.
4. **Delete branches after merge.** Keep the branch list clean.

---

## Commit Conventions

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

### Types

| Type | When to Use |
|------|-------------|
| `feat` | New SDK function, resource, or capability |
| `fix` | Bug fix in SDK code or tests |
| `test` | Adding or updating tests (no SDK code change) |
| `docs` | Documentation, README, comments |
| `chore` | Build config, dependencies, tooling |
| `refactor` | Code restructuring with no behavior change |
| `style` | Formatting, whitespace (no code change) |

### Scopes

| Scope | What It Covers |
|-------|---------------|
| `python` | Python SDK code and tests |
| `typescript` | TypeScript SDK code and tests |
| `discovery` | API surface discovery |
| `plan` | SDK plan and review |
| `deps` | Dependency changes |
| (none) | Cross-cutting changes |

### Examples

```
feat(python): add users resource with list, get, create operations
feat(typescript): add base client with auth and error handling
fix(python): make User.email optional to match API behavior
test(python): add e2e tests for users resource
docs: add initial README with installation and usage examples
chore(deps): update httpx to 0.27.0
refactor(python): extract pagination logic into shared helper
feat(discovery): add HAR archive parsing support
docs(plan): add API surface and SDK function plan
```

### Commit Message Body

For non-trivial changes, include a body explaining **why**:

```
fix(python): handle 204 No Content for delete operations

The API returns 204 with no body for successful deletes.
The client was trying to parse the empty response as JSON,
causing a JSONDecodeError. Now returns None for 204 responses.
```

### Commit Atomicity

Each commit should be:
- **Self-contained:** The repo should pass all existing tests after this commit
- **Single-purpose:** One logical change per commit
- **Reviewable:** Someone should be able to understand the change from the diff + message

Bad:
```
feat(python): add everything
```

Good:
```
feat(python): add error types and shared models
feat(python): add base client with auth and error handling
feat(python): add users resource with tests and mocks
feat(python): add projects resource with tests and mocks
```

---

## Workflow by Phase

### Phase 0: Initialization

```bash
mkdir promptql-sdk && cd promptql-sdk
git init
git checkout -b main

# Create .gitignore
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
.pytest_cache/
node_modules/
dist/
build/
*.egg-info/
.env
.venv/
coverage/
.coverage
*.log
EOF

git add .gitignore
git commit -m "chore: initialize repository with .gitignore"
```

### Phase 1-2: Discovery + Planning

```bash
git checkout -b feat/initial-sdk-plan

# After discovery
git add api-surface.json
git commit -m "docs(discovery): add API surface from {source_type}"

# After planning + review
git add sdk-plan.json
git commit -m "docs(plan): add reviewed SDK function plan

Reviewed by LLM peer review. {N} functions planned across {M} resources.
Review status: approved with {S} suggestions incorporated."

# Merge to main
git checkout main
git merge feat/initial-sdk-plan --no-ff -m "docs: add API discovery and SDK plan"
git branch -d feat/initial-sdk-plan
```

### Phase 3-4: Generation + Testing (per language)

```bash
git checkout -b feat/python-sdk

# Generate in order, committing after each stable unit
git add python/src/promptql/errors.py python/src/promptql/models.py
git commit -m "feat(python): add error types and shared models"

git add python/src/promptql/client.py
git commit -m "feat(python): add base client with auth and error handling"

# For each resource:
git add python/src/promptql/resources/users.py python/tests/test_users.py python/tests/mocks/users.py
git commit -m "feat(python): add users resource with tests and mocks"

# After all tests pass
git add python/tests/e2e/
git commit -m "test(python): add e2e tests for all resources"

# Package metadata
git add python/pyproject.toml python/README.md
git commit -m "chore(python): add package configuration and README"

# Merge to main
git checkout main
git merge feat/python-sdk --no-ff -m "feat: add Python SDK with full test suite"
git tag v0.1.0-python -m "Python SDK v0.1.0 - initial release"
git branch -d feat/python-sdk
```

### Phase 5: Update Cycle

```bash
git checkout -b feat/api-update-2026-02-15

# Update discovery
git add api-surface.json api-surface-diff.json
git commit -m "docs(discovery): update API surface — {N} new, {M} changed, {K} removed endpoints"

# Update plan
git add sdk-plan.json
git commit -m "docs(plan): update SDK plan for new/changed endpoints"

# Update code (per language, per resource)
git add python/src/promptql/resources/users.py python/tests/test_users.py
git commit -m "feat(python): add new fields to User model and list_users_filtered function"

# Deprecations
git add python/src/promptql/resources/legacy.py
git commit -m "deprecate(python): mark get_user_v1 as deprecated — use get_user instead"

# After all tests pass
git checkout main
git merge feat/api-update-2026-02-15 --no-ff \
  -m "feat: API update 2026-02-15 — {summary of changes}"

# Version bump
git tag v0.2.0 -m "v0.2.0 - API update: {N} new endpoints, {M} changes"
git branch -d feat/api-update-2026-02-15
```

---

## Versioning and Tags

Follow [Semantic Versioning](https://semver.org/):

| Change Type | Version Bump | Example |
|------------|-------------|---------|
| New endpoints added (backward compatible) | Minor | 0.1.0 → 0.2.0 |
| Bug fixes, type corrections | Patch | 0.2.0 → 0.2.1 |
| Removed endpoints, breaking type changes | Major | 0.2.1 → 1.0.0 |
| Initial release | | 0.1.0 |

### Tag Format

```
v{major}.{minor}.{patch}            # Single-language SDK
v{major}.{minor}.{patch}-python     # Language-specific tag
v{major}.{minor}.{patch}-typescript # Language-specific tag
```

### Changelog

Maintain a `CHANGELOG.md` at the repo root, updated on every merge to main:

```markdown
# Changelog

## [0.2.0] - 2026-02-15
### Added
- `list_users_filtered()` — filter users by role and status
- `UserRole` enum type

### Changed
- `User.email` is now always present (no longer optional)

### Deprecated
- `get_user_v1()` — use `get_user()` instead, will be removed in v1.0.0

## [0.1.0] - 2026-01-15
### Added
- Initial SDK release with users and projects resources
- Python and TypeScript targets
- Full unit and e2e test suites
```

---

## Merge Rules

Before merging any branch to `main`:

1. **All unit tests pass** — `pytest python/tests/` and `npx jest` must be green
2. **No unresolved TODOs from auto-fix** — review any skipped tests
3. **Commit history is clean** — squash fixup commits if needed
4. **Use `--no-ff`** — always create a merge commit for visibility

```bash
# Pre-merge checklist
cd python && pytest tests/ && cd ..
cd typescript && npx jest && cd ..

# Merge
git checkout main
git merge feat/my-branch --no-ff -m "feat: descriptive merge message"
```

---

## Conflict Resolution

If `main` has advanced while a feature branch was in progress:

```bash
git checkout feat/my-branch
git rebase main

# Resolve any conflicts, then:
git rebase --continue

# Verify tests still pass after rebase
pytest python/tests/
npx jest
```

Prefer rebase over merge for feature branches to keep history linear.

---

## Repository Initialization

Full initialization script for a new SDK project:

```bash
#!/bin/bash
set -e

PROJECT_NAME="${1:-promptql-sdk}"
mkdir -p "$PROJECT_NAME" && cd "$PROJECT_NAME"

git init
git checkout -b main

# .gitignore
cat > .gitignore << 'GITIGNORE'
# Python
__pycache__/
*.pyc
.pytest_cache/
*.egg-info/
dist/
build/
.venv/
.coverage

# TypeScript
node_modules/
dist/
coverage/

# IDE
.idea/
.vscode/
*.swp

# Environment
.env
.env.local

# OS
.DS_Store
Thumbs.db

# SDK specific
api-surface-new.json
api-surface-diff.json
GITIGNORE

# README
cat > README.md << 'README'
# PromptQL SDK

Auto-generated SDK for the PromptQL API.

## Installation

### Python
```bash
pip install promptql
```

### TypeScript
```bash
npm install promptql
```

## Usage

See language-specific README files in `python/` and `typescript/` directories.
README

# Directory structure
mkdir -p python/src python/tests/{mocks,e2e,fixtures}
mkdir -p typescript/src typescript/tests/{mocks,e2e}

git add -A
git commit -m "chore: initialize SDK project scaffold"

echo "Repository initialized at $(pwd)"
```
