# SDK Generator

A Claude Code skill that reverse-engineers APIs into production-ready, typed client SDKs. Point it at a live website or hand it a HAR file, and it discovers endpoints, designs functions with LLM peer review, generates Python and TypeScript SDKs with full test suites, and keeps them in sync as the API evolves.

## What It Does

SDK Generator runs a five-phase autonomous workflow:

| Phase | What Happens | Output |
|-------|-------------|--------|
| Discovery | Crawls a live site or parses a HAR archive to find API endpoints | `api-surface.json` |
| Planning | Designs SDK functions per endpoint, reviewed by an LLM peer | `sdk-plan.json` |
| Generation | Produces idiomatic Python and TypeScript code with mocks and tests | SDK source tree |
| Testing | Runs unit and e2e tests with an auto-fix loop (up to 5 retries) | Green test suite |
| Git Finalize | Atomic commits, semantic versioning, tags, changelog | Tagged release |

When the upstream API changes, re-run discovery. The skill diffs the old and new surfaces, plans updates, regenerates only what changed, and bumps the version.

## Installation

Copy this skill into your Claude Code skills directory:

```bash
cp -r sdk-generator ~/.claude/skills/sdk-generator
```

Or clone directly:

```bash
git clone <repo-url> ~/.claude/skills/sdk-generator
```

### Prerequisites

The skill needs these tools available on your system:

```bash
# Runtime
python3 --version   # Python 3.10+
node --version      # Node.js 18+
git --version

# Python SDK dependencies
pip install httpx pydantic pytest pytest-asyncio respx

# TypeScript SDK dependencies
npm install -g typescript jest ts-jest @types/jest axios

# Optional: browser-based login for authenticated API discovery
pip install playwright
playwright install chromium
```

## Usage

Invoke the skill through Claude Code:

```
Generate a Python and TypeScript SDK from this HAR file: ./api-traffic.har
```

```
Reverse-engineer the API at https://api.example.com and build typed clients
```

```
Update the SDK in ./my-sdk/ — the API has new endpoints
```

### Authenticated APIs

For APIs that require authentication, provide credentials inline or via environment variables:

```
# Inline credentials (cookie, bearer token, or custom header)
Crawl https://app.example.com using cookie: "session=abc123; csrf_token=xyz789"
Crawl https://api.example.com with bearer token: "eyJhbGciOiJIUzI1NiIs..."
Crawl https://api.example.com with header X-API-Key: "sk-live-abc123"
```

```bash
# Or set environment variables before running
export SDK_GEN_AUTH_TOKEN="eyJhbGciOiJIUzI1NiIs..."
export SDK_GEN_AUTH_COOKIE="session=abc123"
export SDK_GEN_AUTH_HEADER_NAME="X-API-Key"
export SDK_GEN_AUTH_HEADER_VALUE="sk-live-abc123"
```

If you don't have credentials handy, the agent can launch a browser window (via
Playwright or an MCP browser server) for you to log in interactively. For OAuth/SSO
flows, the agent opens a visible browser, guides you through login, and then
automatically extracts the session cookies and tokens.

Claude handles the rest: discovery, planning, review, code generation, testing, and git management.

### What Gets Generated

```
{api-name}-sdk/
├── api-surface.json              # Discovered endpoints (source of truth)
├── sdk-plan.json                 # Reviewed function designs
├── CHANGELOG.md
├── python/
│   ├── src/{package}/
│   │   ├── client.py             # Base HTTP client with auth
│   │   ├── models.py             # Pydantic data classes
│   │   ├── errors.py             # Typed exception hierarchy
│   │   └── resources/            # One module per API resource
│   └── tests/
│       ├── test_*.py             # Unit tests with mocks
│       └── e2e/                  # Live endpoint tests
└── typescript/
    ├── src/
    │   ├── client.ts
    │   ├── models.ts
    │   ├── errors.ts
    │   └── resources/
    └── tests/
        ├── *.test.ts
        └── e2e/
```

## How It Works

The skill coordinates four specialized agents:

- **Discovery agent** — Crawls websites or parses HAR files. Normalizes URLs, infers JSON schemas, detects auth patterns (Bearer, API key, OAuth, cookies). Supports authenticated discovery via inline credentials, environment variables, or browser-based login (Playwright / MCP) with guided OAuth/SSO flows.
- **Reviewer agent** — Reviews the SDK plan for naming consistency, type safety, missing error cases, and pagination patterns. Produces approve/suggest/reject verdicts.
- **Generator agent** — Writes SDK code in dependency order with full docstrings, type hints, and matching test files. Commits each resource module atomically.
- **Test fixer agent** — Parses failures, diagnoses root causes, applies minimal fixes, and re-runs. Gives up after 5 attempts and moves on rather than blocking.

Details for each phase live in the `references/` directory.

## Project Structure

```
sdk-generator/
├── SKILL.md              # Skill definition and full workflow
├── agents/               # Agent instructions
│   ├── discovery.md
│   ├── generator.md
│   ├── reviewer.md
│   └── test-fixer.md
└── references/           # Detailed procedures
    ├── discovery.md
    ├── generation.md
    ├── planning.md
    ├── testing.md
    ├── git-workflow.md
    ├── update-cycle.md
    └── repo-structure.md
```

## Design Principles

- **Never block on a single failure.** If one endpoint can't be fixed after 5 attempts, mark it broken, commit what works, and move on.
- **Every commit leaves tests passing.** No half-finished states on any branch.
- **Surface unknowns explicitly.** Unknown types become `Any`/`unknown` with a `TODO`, not silent assumptions.
- **LLM review is mandatory.** The planning review catches design mistakes before code is written.
- **Updates are first-class.** The diff-and-patch cycle is built in, not bolted on.

## Contributing

Contributions are welcome. The skill is structured so each phase can be improved independently:

- Improve discovery heuristics in `references/discovery.md` and `agents/discovery.md`
- Refine code generation patterns in `references/generation.md` and `agents/generator.md`
- Tighten review criteria in `agents/reviewer.md`
- Add support for new languages by extending `references/generation.md` and `references/repo-structure.md`

## License

MIT
