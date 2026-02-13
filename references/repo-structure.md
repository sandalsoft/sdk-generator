# Repository Structure

The complete directory layout for a generated SDK project.

```
promptql-sdk/
├── .gitignore
├── README.md
├── CHANGELOG.md
├── MIGRATION.md                      # Created when breaking changes occur
│
├── api-surface.json                  # Current API surface (source of truth)
├── sdk-plan.json                     # Approved SDK function plan
│
├── python/
│   ├── pyproject.toml
│   ├── README.md
│   ├── src/
│   │   └── promptql/
│   │       ├── __init__.py           # Public API re-exports
│   │       ├── client.py             # PromptQLClient class
│   │       ├── models.py             # Pydantic models (User, Project, etc.)
│   │       ├── errors.py             # Exception hierarchy
│   │       ├── types.py              # Type aliases, generics, enums
│   │       ├── _pagination.py        # Internal pagination helpers
│   │       └── resources/
│   │           ├── __init__.py
│   │           ├── users.py          # UsersResource
│   │           ├── projects.py       # ProjectsResource
│   │           └── ...               # One file per resource group
│   └── tests/
│       ├── conftest.py               # Shared test fixtures
│       ├── mocks/
│       │   ├── __init__.py
│       │   ├── users.py              # Mock factories for users
│       │   └── projects.py
│       ├── fixtures/
│       │   ├── users_list.json       # Sample response payloads
│       │   ├── user_detail.json
│       │   └── error_401.json
│       ├── test_users.py             # Unit tests
│       ├── test_projects.py
│       └── e2e/
│           ├── conftest.py           # E2E-specific fixtures
│           ├── test_users_e2e.py
│           └── test_projects_e2e.py
│
├── typescript/
│   ├── package.json
│   ├── tsconfig.json
│   ├── jest.config.js
│   ├── README.md
│   ├── src/
│   │   ├── index.ts                  # Public API re-exports
│   │   ├── client.ts                 # PromptQLClient class
│   │   ├── models.ts                 # Interface definitions
│   │   ├── errors.ts                 # Error class hierarchy
│   │   ├── types.ts                  # Type aliases, generics, enums
│   │   ├── pagination.ts             # Pagination helpers
│   │   └── resources/
│   │       ├── index.ts
│   │       ├── users.ts
│   │       ├── projects.ts
│   │       └── ...
│   └── tests/
│       ├── mocks/
│       │   ├── users.ts
│       │   └── projects.ts
│       ├── fixtures/
│       │   ├── users_list.json
│       │   └── ...
│       ├── users.test.ts
│       ├── projects.test.ts
│       └── e2e/
│           ├── users.e2e.test.ts
│           └── projects.e2e.test.ts
│
└── scripts/                          # Build/release automation
    ├── diff_surfaces.py              # Diff two api-surface files
    ├── init_project.sh               # Scaffold a new SDK project
    └── run_all_tests.sh              # Cross-language test runner
```

## Key Files

| File | Purpose | Updated When |
|------|---------|-------------|
| `api-surface.json` | Canonical API surface description | Every discovery/update cycle |
| `sdk-plan.json` | Approved function designs | Every planning/update cycle |
| `CHANGELOG.md` | Human-readable change history | Every merge to main |
| `.gitignore` | Keep repo clean | Project initialization |
| `pyproject.toml` | Python package metadata | Version bumps |
| `package.json` | TypeScript package metadata | Version bumps |
