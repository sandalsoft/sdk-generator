# API Discovery

This document covers the two discovery paths in detail: live website crawling and HAR
archive parsing. Both produce the same output: an `api-surface.json` file.

## Table of Contents
- [Common Output Format](#common-output-format)
- [Authenticated Browsing](#authenticated-browsing)
- [Path A: Live Website Crawling](#path-a-live-website-crawling)
- [Path B: HAR Archive Parsing](#path-b-har-archive-parsing)
- [Endpoint Normalization](#endpoint-normalization)
- [Schema Inference](#schema-inference)
- [Auth Detection](#auth-detection)

---

## Common Output Format

The `api-surface.json` file has this structure:

```json
{
  "metadata": {
    "source": "har" | "live",
    "source_url": "https://example.com",
    "discovered_at": "2026-01-15T10:30:00Z",
    "total_endpoints": 42
  },
  "auth": {
    "type": "bearer" | "api_key" | "cookie" | "oauth2" | "custom",
    "header_name": "Authorization",
    "token_prefix": "Bearer",
    "notes": "Token obtained via POST /auth/login"
  },
  "endpoints": [
    {
      "id": "get-users-list",
      "method": "GET",
      "path_pattern": "/api/v1/users",
      "path_params": [],
      "query_params": [
        { "name": "page", "type": "integer", "required": false, "default": 1 },
        { "name": "limit", "type": "integer", "required": false, "default": 20 }
      ],
      "request_headers": {
        "Authorization": "Bearer {token}",
        "Content-Type": "application/json"
      },
      "request_body_schema": null,
      "response_schema": {
        "type": "object",
        "properties": {
          "data": { "type": "array", "items": { "$ref": "#/models/User" } },
          "total": { "type": "integer" },
          "page": { "type": "integer" }
        }
      },
      "response_status_codes": [200, 401, 403],
      "error_schema": {
        "type": "object",
        "properties": {
          "error": { "type": "string" },
          "message": { "type": "string" }
        }
      },
      "pagination": {
        "type": "offset",
        "page_param": "page",
        "limit_param": "limit",
        "total_field": "total"
      },
      "tags": ["users"],
      "observations": 3,
      "sample_responses": ["...truncated..."]
    }
  ],
  "models": {
    "User": {
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "name": { "type": "string" },
        "email": { "type": "string", "format": "email" }
      },
      "required": ["id", "name"]
    }
  }
}
```

---

## Authenticated Browsing

Many websites and APIs require authentication before their endpoints are accessible.
The discovery agent supports multiple strategies for obtaining and using credentials
during crawling, from direct credential input to automated browser-based login.

### Credential Input Methods

Credentials can be provided in two ways (inline takes precedence over env vars):

**1. Inline in the prompt:**
```
Crawl https://app.example.com using cookie: "session=abc123; csrf_token=xyz789"
Crawl https://api.example.com with bearer token: "eyJhbGciOiJIUzI1NiIs..."
Crawl https://api.example.com with header X-API-Key: "sk-live-abc123"
```

**2. Environment variables:**

| Variable | Purpose | Example |
|----------|---------|---------|
| `SDK_GEN_AUTH_TOKEN` | Bearer token value | `eyJhbGciOiJIUzI1NiIs...` |
| `SDK_GEN_AUTH_COOKIE` | Cookie header string | `session=abc123; csrf_token=xyz` |
| `SDK_GEN_AUTH_HEADER_NAME` | Custom header name | `X-API-Key` |
| `SDK_GEN_AUTH_HEADER_VALUE` | Custom header value | `sk-live-abc123` |

When both inline and env var credentials are present, inline values take precedence.
Multiple auth types can be combined (e.g., a cookie for the website plus a bearer
token for its API).

### Supported Auth Types

| Type | How to Provide | Applied As |
|------|---------------|------------|
| **Cookie** | Inline or `SDK_GEN_AUTH_COOKIE` | `Cookie` header on all requests |
| **Bearer token** | Inline or `SDK_GEN_AUTH_TOKEN` | `Authorization: Bearer {token}` header |
| **Custom header** | Inline or `SDK_GEN_AUTH_HEADER_NAME` + `SDK_GEN_AUTH_HEADER_VALUE` | `{name}: {value}` header |

### Browser-Based Login (Playwright)

For sites where credentials aren't readily available — or where login produces
short-lived session tokens — the discovery agent can use Playwright to automate
a browser login flow and extract the resulting cookies and tokens.

#### When to Use Browser Login

- The user doesn't have a raw cookie or token to provide
- The site uses form-based login that sets session cookies
- Tokens are issued during login and stored in `localStorage` or `sessionStorage`
- The user wants to authenticate against a staging or development environment

#### Login Flow

```python
# Pseudocode for browser-based credential extraction
from playwright.sync_api import sync_playwright

def extract_credentials_via_browser(
    login_url: str,
    credentials: dict = None,  # {"username": "...", "password": "..."}
    manual: bool = False,
) -> dict:
    """
    Launch a browser, log in, and extract session cookies + tokens.

    If credentials are provided, attempt automated form login.
    If manual=True or automated login fails, pause for user interaction.

    Returns a dict with extracted auth data:
      {"cookies": "...", "token": "...", "headers": {...}}
    """
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=not manual)
        context = browser.new_context()
        page = context.new_page()

        page.goto(login_url)

        if credentials and not manual:
            # Attempt automated login
            fill_login_form(page, credentials)
        else:
            # Pause and let the user log in manually
            # This opens a visible browser window
            print("Please log in via the browser window.")
            print("Press Enter here once you have logged in...")
            page.pause()  # Opens Playwright Inspector for manual interaction

        # Wait for navigation after login
        page.wait_for_load_state("networkidle")

        # Extract cookies from the browser context
        cookies = context.cookies()
        cookie_header = "; ".join(
            f"{c['name']}={c['value']}" for c in cookies
            if c['domain'] in login_url
        )

        # Extract tokens from localStorage / sessionStorage
        token = page.evaluate("""() => {
            return localStorage.getItem('token')
                || localStorage.getItem('access_token')
                || localStorage.getItem('auth_token')
                || sessionStorage.getItem('token')
                || sessionStorage.getItem('access_token')
                || null;
        }""")

        # Extract any auth headers from intercepted API calls
        auth_headers = extract_auth_headers_from_requests(page)

        browser.close()

        return {
            "cookies": cookie_header or None,
            "token": token,
            "headers": auth_headers,
        }

def fill_login_form(page, credentials):
    """Attempt to fill and submit a login form automatically."""
    # Common selectors for login forms
    username_selectors = [
        'input[name="username"]', 'input[name="email"]',
        'input[type="email"]', 'input[id="username"]',
        'input[id="email"]', 'input[name="login"]',
    ]
    password_selectors = [
        'input[name="password"]', 'input[type="password"]',
        'input[id="password"]',
    ]
    submit_selectors = [
        'button[type="submit"]', 'input[type="submit"]',
        'button:has-text("Log in")', 'button:has-text("Sign in")',
    ]

    for selector in username_selectors:
        if page.query_selector(selector):
            page.fill(selector, credentials["username"])
            break

    for selector in password_selectors:
        if page.query_selector(selector):
            page.fill(selector, credentials["password"])
            break

    for selector in submit_selectors:
        if page.query_selector(selector):
            page.click(selector)
            break
```

#### MCP Browser Alternative

If a browser automation MCP server is available (e.g., `@anthropic/browser-mcp`),
prefer it over spawning Playwright directly. The MCP server provides the same
capabilities through the tool interface:

1. Use the MCP `browser_navigate` tool to open the login page
2. Use `browser_fill` / `browser_click` to complete login forms
3. Use `browser_evaluate` to extract tokens from `localStorage` / `sessionStorage`
4. Use `browser_cookies` to extract session cookies
5. Use the extracted credentials for subsequent API crawling

The MCP approach integrates naturally with the agent's tool-calling flow and avoids
needing Playwright installed locally.

### OAuth / SSO Flows

OAuth and SSO flows typically require user interaction (consent screens, MFA prompts,
IdP redirects) that cannot be fully automated. Use the following strategy:

#### Strategy: Guided Manual Login

1. **Detect the flow:** If navigating to the login page redirects to a third-party
   IdP (Google, Okta, Auth0, Azure AD, etc.), identify it as OAuth/SSO.

2. **Launch a visible browser:**
   ```python
   browser = p.chromium.launch(headless=False)  # Must be visible
   ```

3. **Inform the user:**
   ```
   This site uses SSO/OAuth authentication (detected: {provider}).
   A browser window has been opened to the login page.

   Please:
   1. Complete the login flow in the browser (including any MFA prompts)
   2. Wait until you see the authenticated application page
   3. Return here and press Enter to continue

   The agent will then extract your session cookies and tokens automatically.
   ```

4. **Wait for user completion:** Use `page.pause()` or poll for a known
   post-login indicator (a specific URL pattern, a DOM element, or cookies being set).

5. **Extract credentials:** After the user confirms login is complete, extract
   cookies and tokens as shown in the browser login flow above.

6. **Handle token refresh:** If OAuth refresh tokens are observed, record them
   in the auth section of `api-surface.json` so the generated SDK can implement
   automatic token refresh:
   ```json
   {
     "auth": {
       "type": "oauth2",
       "token_endpoint": "/oauth/token",
       "refresh_token_observed": true,
       "grant_type": "refresh_token",
       "notes": "Access token expires in 3600s. Refresh via POST /oauth/token."
     }
   }
   ```

### Credential Resolution Order

When multiple credential sources are available, resolve in this order:

1. **Inline prompt credentials** (highest priority)
2. **Environment variables** (`SDK_GEN_AUTH_TOKEN`, `SDK_GEN_AUTH_COOKIE`, etc.)
3. **Browser-extracted credentials** (from Playwright or MCP login)
4. **No auth** (attempt unauthenticated crawling)

If auth is required but no credentials are available and browser login is not
feasible, prompt the user:

```
The site at https://app.example.com requires authentication.
How would you like to provide credentials?

1. Provide a cookie string or auth token directly
2. Set SDK_GEN_AUTH_COOKIE or SDK_GEN_AUTH_TOKEN environment variables
3. Let me open a browser window so you can log in manually
```

### Credential Security

- Never write raw tokens or cookies to `api-surface.json` — use placeholders like `{token}`
- Never log credential values in agent output
- Redact `Authorization`, `Cookie`, and custom auth headers in all committed files
- If credentials are extracted via browser login, discard them from memory after
  the crawl completes — do not persist them to disk

---

## Path A: Live Website Crawling

### Strategy

Use a headless browser or HTTP client to navigate the target website while intercepting
network requests. The goal is to trigger as many API calls as possible by:

1. Loading the main page and all linked pages
2. Interacting with UI elements (buttons, forms, dropdowns)
3. Navigating through authenticated flows if credentials are provided
4. Following pagination links and loading more data

### Resolving Credentials

Before crawling, resolve authentication credentials using the
[Credential Resolution Order](#credential-resolution-order) defined above.
Apply resolved credentials to the HTTP client or browser context:

```python
def apply_auth(client_or_context, auth: dict):
    """Apply resolved credentials to the HTTP client or Playwright context."""
    headers = {}
    if auth.get("token"):
        headers["Authorization"] = f"Bearer {auth['token']}"
    if auth.get("cookies"):
        headers["Cookie"] = auth["cookies"]
    if auth.get("custom_header_name") and auth.get("custom_header_value"):
        headers[auth["custom_header_name"]] = auth["custom_header_value"]

    if isinstance(client_or_context, httpx.Client):
        client_or_context.headers.update(headers)
    else:
        # Playwright: set extra HTTP headers on the browser context
        client_or_context.set_extra_http_headers(headers)
```

### Implementation Steps

```python
# Pseudocode for live discovery
import httpx
import os
from urllib.parse import urlparse

def resolve_credentials(inline_auth: dict = None) -> dict:
    """
    Resolve credentials from inline input and/or environment variables.
    Inline values take precedence over env vars.
    """
    auth = {}

    # Bearer token
    auth["token"] = (
        (inline_auth or {}).get("token")
        or os.environ.get("SDK_GEN_AUTH_TOKEN")
    )

    # Cookie string
    auth["cookies"] = (
        (inline_auth or {}).get("cookies")
        or os.environ.get("SDK_GEN_AUTH_COOKIE")
    )

    # Custom header
    auth["custom_header_name"] = (
        (inline_auth or {}).get("custom_header_name")
        or os.environ.get("SDK_GEN_AUTH_HEADER_NAME")
    )
    auth["custom_header_value"] = (
        (inline_auth or {}).get("custom_header_value")
        or os.environ.get("SDK_GEN_AUTH_HEADER_VALUE")
    )

    return {k: v for k, v in auth.items() if v}

async def discover_live(base_url: str, inline_auth: dict = None):
    """
    Crawl the site and collect API calls.
    Start with the base URL, follow links, and record all XHR-like requests.

    Args:
        base_url: The website URL to crawl.
        inline_auth: Optional dict with auth credentials provided inline:
            {"token": "...", "cookies": "...",
             "custom_header_name": "...", "custom_header_value": "..."}
    """
    auth = resolve_credentials(inline_auth)
    discovered = []
    visited_pages = set()
    api_calls = []

    # 0. If no credentials resolved and site requires auth, attempt browser login
    # 1. Apply resolved credentials to the HTTP client or browser context
    # 2. Fetch the main page, extract JS bundles
    # 3. Parse JS for API base URLs and endpoint patterns
    # 4. Navigate pages, recording fetch/XHR calls
    # 5. Group and normalize collected calls

    return build_api_surface(api_calls)
```

### Tips for Live Crawling

- Look for API base URLs in JavaScript bundles (common patterns: `/api/v1`, `/graphql`)
- Check for OpenAPI/Swagger specs at common paths: `/api/docs`, `/swagger.json`, `/openapi.json`
- If an OpenAPI spec exists, prefer it over crawling — it's more complete
- Inspect `<script>` tags for embedded configuration objects with API URLs
- Watch for WebSocket connections (note these separately — they need different SDK patterns)
- If the initial crawl returns 401/403 responses and no credentials were provided, prompt
  the user for credentials or offer browser-based login before retrying

---

## Path B: HAR Archive Parsing

### Strategy

Parse the HAR (HTTP Archive) file and extract API-like requests. HAR files contain
every network request the browser made, so filtering is critical.

HAR files captured from authenticated browser sessions contain live auth tokens and
cookies. These are useful for detecting auth patterns but must be redacted before
any data is committed.

### Implementation Steps

```python
import json
import re

# Headers that contain sensitive auth data
SENSITIVE_HEADERS = {
    "authorization", "cookie", "set-cookie",
    "x-api-key", "x-auth-token", "x-csrf-token",
    "x-session-id", "proxy-authorization",
}

def redact_header_value(name: str, value: str) -> str:
    """Replace sensitive header values with a placeholder."""
    if name.lower() in SENSITIVE_HEADERS:
        # Preserve the auth scheme (e.g., "Bearer") but redact the token
        if name.lower() == "authorization" and " " in value:
            scheme, _ = value.split(" ", 1)
            return f"{scheme} {{token}}"
        if name.lower() == "cookie":
            # Redact cookie values but preserve names for pattern detection
            return re.sub(r'=([^;]+)', '={redacted}', value)
        return "{redacted}"
    return value

def extract_auth_from_har(entries: list) -> dict:
    """
    Analyze HAR entries to detect the auth mechanism used during capture.
    Returns an auth descriptor for api-surface.json.
    """
    auth_headers_seen = {}

    for entry in entries:
        for header in entry["request"]["headers"]:
            name = header["name"].lower()
            if name in SENSITIVE_HEADERS:
                auth_headers_seen.setdefault(name, []).append(header["value"])

    if "authorization" in auth_headers_seen:
        sample = auth_headers_seen["authorization"][0]
        if sample.startswith("Bearer "):
            return {"type": "bearer", "header_name": "Authorization",
                    "token_prefix": "Bearer"}
        return {"type": "custom", "header_name": "Authorization",
                "notes": f"Scheme: {sample.split(' ', 1)[0]}"}

    if "cookie" in auth_headers_seen:
        return {"type": "cookie", "header_name": "Cookie",
                "notes": "Session cookies used for authentication"}

    for name in auth_headers_seen:
        if name.startswith("x-"):
            return {"type": "api_key", "header_name": name,
                    "notes": f"Custom auth header: {name}"}

    return {"type": "none", "notes": "No auth headers detected in HAR"}

def parse_har(har_path: str) -> dict:
    with open(har_path) as f:
        har = json.load(f)

    api_calls = []
    all_entries = har["log"]["entries"]

    # Extract auth pattern before filtering
    auth_info = extract_auth_from_har(all_entries)

    for entry in all_entries:
        request = entry["request"]
        response = entry["response"]

        # Filter: only keep API-like requests
        if not is_api_request(request, response):
            continue

        # Redact sensitive headers before storing
        redacted_headers = {
            h["name"]: redact_header_value(h["name"], h["value"])
            for h in request["headers"]
        }

        api_calls.append({
            "method": request["method"],
            "url": request["url"],
            "request_headers": redacted_headers,
            "request_body": parse_body(request.get("postData")),
            "response_status": response["status"],
            "response_headers": {h["name"]: h["value"] for h in response["headers"]},
            "response_body": parse_response_body(response["content"]),
        })

    surface = build_api_surface(api_calls)
    surface["auth"] = auth_info
    return surface

def is_api_request(request, response):
    """Heuristics for identifying API calls vs static assets."""
    url = request["url"]
    content_type = get_header(response["headers"], "content-type", "")

    # Include if:
    # - Response is JSON
    # - URL contains /api/, /v1/, /graphql, etc.
    # - Method is POST/PUT/PATCH/DELETE (usually not static assets)
    # Exclude if:
    # - URL ends in .js, .css, .png, .jpg, .svg, .woff, etc.
    # - Content-Type is text/html, text/css, image/*, font/*
    # - URL matches known CDN patterns

    if any(ext in url for ext in ['.js', '.css', '.png', '.jpg', '.svg', '.woff']):
        return False
    if 'application/json' in content_type:
        return True
    if any(pattern in url for pattern in ['/api/', '/v1/', '/v2/', '/graphql']):
        return True
    if request["method"] in ['POST', 'PUT', 'PATCH', 'DELETE']:
        return True
    return False
```

### HAR Parsing Gotchas

- HAR files can be very large — stream if possible
- Response bodies may be base64-encoded (check `content.encoding`)
- Some tools export HAR with truncated bodies — handle missing data gracefully
- Cookies in HAR may contain sensitive tokens — always redact values before committing
- Multiple observations of the same endpoint give better schema inference
- HAR files captured from authenticated sessions are the best source for auth pattern
  detection — use `extract_auth_from_har()` to identify the mechanism, then redact
  all token values before writing to `api-surface.json`

---

## Endpoint Normalization

After collecting raw API calls, normalize them into endpoint patterns:

1. **Path parameter detection:** If you see `/users/123` and `/users/456`, normalize to
   `/users/{user_id}`. Heuristics: UUIDs, numeric IDs, slugs in the same path position.

2. **Grouping:** Merge all observations of the same method + path pattern. This gives
   you multiple sample requests/responses to infer schemas from.

3. **Query param analysis:** Across observations, identify which query params appear,
   their types, and whether they're always present (required) or sometimes absent (optional).

```python
def normalize_path(url: str) -> tuple[str, list[str]]:
    """Convert a concrete URL to a pattern with named path params."""
    parts = urlparse(url).path.split('/')
    params = []
    normalized = []
    for part in parts:
        if looks_like_id(part):
            param_name = infer_param_name(normalized[-1] if normalized else "id")
            params.append(param_name)
            normalized.append(f"{{{param_name}}}")
        else:
            normalized.append(part)
    return '/'.join(normalized), params

def looks_like_id(segment: str) -> bool:
    """Heuristic: is this path segment a dynamic ID?"""
    import re
    if re.match(r'^\d+$', segment):  # Numeric ID
        return True
    if re.match(r'^[0-9a-f]{8}-', segment):  # UUID
        return True
    if len(segment) > 20 and re.match(r'^[a-zA-Z0-9]+$', segment):  # Hash/token
        return True
    return False
```

---

## Schema Inference

Given multiple observations of the same endpoint's response, infer a JSON schema:

1. Merge all observed response bodies
2. For each field, determine: type (string/number/boolean/array/object/null),
   whether it's always present (required), and if it's nullable
3. For array items, infer the item schema from all observed items
4. For nested objects, recurse
5. Extract shared shapes into the `models` section and use `$ref` pointers

**Type inference rules:**
- If a field appears in all observations → required
- If a field is sometimes `null` → nullable
- If a field has values `"active"`, `"inactive"`, `"pending"` → enum
- If a field looks like ISO 8601 → `string` with `format: date-time`
- If a field looks like an email → `string` with `format: email`
- If a field looks like a URL → `string` with `format: uri`

---

## Auth Detection

Analyze request headers across all endpoints to identify auth patterns:

| Pattern | Detection | Notes |
|---------|-----------|-------|
| Bearer token | `Authorization: Bearer ...` header | Most common for SPAs |
| API key | Consistent custom header like `X-API-Key` | Sometimes in query params |
| Cookie auth | `Cookie` header with session token | Common for server-rendered apps |
| OAuth2 | Token endpoint + refresh flow observed | Look for `/oauth/token` calls |
| No auth | No consistent auth headers | Public API |

Record the auth pattern in the API surface file so the SDK can implement the correct
auth flow. If multiple auth methods are observed, document all of them.
