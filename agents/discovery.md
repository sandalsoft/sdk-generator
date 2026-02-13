# API Discovery

This document covers the two discovery paths in detail: live website crawling and HAR
archive parsing. Both produce the same output: an `api-surface.json` file.

## Table of Contents
- [Common Output Format](#common-output-format)
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

## Path A: Live Website Crawling

### Strategy

Use a headless browser or HTTP client to navigate the target website while intercepting
network requests. The goal is to trigger as many API calls as possible by:

1. Loading the main page and all linked pages
2. Interacting with UI elements (buttons, forms, dropdowns)
3. Navigating through authenticated flows if credentials are provided
4. Following pagination links and loading more data

### Implementation Steps

```python
# Pseudocode for live discovery
import httpx
from urllib.parse import urlparse

async def discover_live(base_url: str, auth_token: str = None):
    """
    Crawl the site and collect API calls.
    Start with the base URL, follow links, and record all XHR-like requests.
    """
    discovered = []
    visited_pages = set()
    api_calls = []

    # 1. Fetch the main page, extract JS bundles
    # 2. Parse JS for API base URLs and endpoint patterns
    # 3. Navigate pages, recording fetch/XHR calls
    # 4. Group and normalize collected calls

    return build_api_surface(api_calls)
```

### Tips for Live Crawling

- Look for API base URLs in JavaScript bundles (common patterns: `/api/v1`, `/graphql`)
- Check for OpenAPI/Swagger specs at common paths: `/api/docs`, `/swagger.json`, `/openapi.json`
- If an OpenAPI spec exists, prefer it over crawling — it's more complete
- Inspect `<script>` tags for embedded configuration objects with API URLs
- Watch for WebSocket connections (note these separately — they need different SDK patterns)

---

## Path B: HAR Archive Parsing

### Strategy

Parse the HAR (HTTP Archive) file and extract API-like requests. HAR files contain
every network request the browser made, so filtering is critical.

### Implementation Steps

```python
import json

def parse_har(har_path: str) -> dict:
    with open(har_path) as f:
        har = json.load(f)

    api_calls = []
    for entry in har["log"]["entries"]:
        request = entry["request"]
        response = entry["response"]

        # Filter: only keep API-like requests
        if not is_api_request(request, response):
            continue

        api_calls.append({
            "method": request["method"],
            "url": request["url"],
            "request_headers": {h["name"]: h["value"] for h in request["headers"]},
            "request_body": parse_body(request.get("postData")),
            "response_status": response["status"],
            "response_headers": {h["name"]: h["value"] for h in response["headers"]},
            "response_body": parse_response_body(response["content"]),
        })

    return build_api_surface(api_calls)

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
- Cookies in HAR may contain sensitive tokens — redact in output
- Multiple observations of the same endpoint give better schema inference

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
