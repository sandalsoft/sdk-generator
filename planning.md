# SDK Function Planning

This document covers how to transform discovered API endpoints into a reviewed,
approved SDK function plan before any code is written.

## Table of Contents
- [Plan Structure](#plan-structure)
- [Function Design Rules](#function-design-rules)
- [LLM Peer Review Process](#llm-peer-review-process)
- [Review Criteria](#review-criteria)
- [Handling Review Feedback](#handling-review-feedback)

---

## Plan Structure

The `sdk-plan.json` file is the contract between planning and generation:

```json
{
  "metadata": {
    "api_name": "PromptQL",
    "version": "0.1.0",
    "planned_at": "2026-01-15T10:30:00Z",
    "review_status": "approved",
    "languages": ["python", "typescript"]
  },
  "resources": [
    {
      "name": "users",
      "description": "User management operations",
      "functions": [
        {
          "endpoint_id": "get-users-list",
          "names": {
            "python": "list_users",
            "typescript": "listUsers"
          },
          "description": "Retrieve a paginated list of users",
          "parameters": [
            {
              "name": "page",
              "type": "integer",
              "required": false,
              "default": 1,
              "description": "Page number for pagination"
            },
            {
              "name": "limit",
              "type": "integer",
              "required": false,
              "default": 20,
              "description": "Number of results per page"
            }
          ],
          "return_type": {
            "python": "PaginatedResponse[User]",
            "typescript": "PaginatedResponse<User>"
          },
          "errors": [
            { "status": 401, "type": "AuthenticationError" },
            { "status": 403, "type": "PermissionError" }
          ],
          "pagination": {
            "enabled": true,
            "strategy": "offset",
            "auto_paginate_method": "list_users_all"
          },
          "needs_auth": true,
          "http_method": "GET",
          "path_pattern": "/api/v1/users",
          "mock_spec": {
            "factory_name": "mock_users_list",
            "sample_count": 3,
            "edge_cases": ["empty_list", "single_item", "max_page"]
          },
          "review_status": "approved",
          "review_notes": ""
        }
      ]
    }
  ],
  "shared_types": [
    {
      "name": "User",
      "fields": [
        { "name": "id", "type": "string", "required": true },
        { "name": "name", "type": "string", "required": true },
        { "name": "email", "type": "string", "required": false, "nullable": true }
      ]
    },
    {
      "name": "PaginatedResponse",
      "generic": true,
      "type_param": "T",
      "fields": [
        { "name": "data", "type": "array<T>", "required": true },
        { "name": "total", "type": "integer", "required": true },
        { "name": "page", "type": "integer", "required": true }
      ]
    }
  ],
  "error_types": [
    {
      "name": "AuthenticationError",
      "status_code": 401,
      "description": "Request lacks valid authentication credentials"
    },
    {
      "name": "PermissionError",
      "status_code": 403,
      "description": "Authenticated user lacks required permissions"
    },
    {
      "name": "NotFoundError",
      "status_code": 404,
      "description": "Requested resource does not exist"
    },
    {
      "name": "ValidationError",
      "status_code": 422,
      "description": "Request body failed validation"
    },
    {
      "name": "RateLimitError",
      "status_code": 429,
      "description": "Too many requests"
    }
  ]
}
```

---

## Function Design Rules

When converting endpoints to functions, follow these conventions:

### Naming

| Pattern | Python | TypeScript |
|---------|--------|------------|
| List resources | `list_users()` | `listUsers()` |
| Get single resource | `get_user(id)` | `getUser(id)` |
| Create resource | `create_user(data)` | `createUser(data)` |
| Update resource | `update_user(id, data)` | `updateUser(id, data)` |
| Delete resource | `delete_user(id)` | `deleteUser(id)` |
| Custom action | `activate_user(id)` | `activateUser(id)` |

### Parameter Handling

- Path parameters become required positional arguments
- Query parameters become keyword arguments with defaults
- Request bodies become typed data objects (not raw dicts)
- Headers are handled by the client, not passed per-function (except overrides)

### Return Types

- Always return typed objects, never raw JSON
- Paginated endpoints get both a single-page method and an auto-paginate iterator
- Void operations (DELETE 204) return `None` / `void`

### Error Handling

- Map HTTP status codes to typed exception classes
- 4xx errors → client errors (user's fault)
- 5xx errors → server errors (API's fault, may be retriable)
- Include the response body in the exception for debugging

---

## LLM Peer Review Process

After generating the initial plan, it must be reviewed. This is non-negotiable —
the review catches naming inconsistencies, missing edge cases, and design mistakes
that are expensive to fix after code generation.

### How the Review Works

1. **Prepare the review prompt.** Send the full `sdk-plan.json` to a second LLM call
   with the system prompt from `agents/reviewer.md`.

2. **The reviewer evaluates each function** against the criteria below and produces a
   structured review:

```json
{
  "overall_verdict": "approved_with_suggestions",
  "summary": "Solid API surface coverage. A few naming issues and missing retry logic.",
  "function_reviews": [
    {
      "endpoint_id": "get-users-list",
      "verdict": "approve",
      "notes": "Clean design, good pagination support."
    },
    {
      "endpoint_id": "create-user",
      "verdict": "suggest",
      "suggestions": [
        "Add idempotency key parameter for safe retries",
        "Consider accepting partial User type for creation (CreateUserInput)"
      ]
    },
    {
      "endpoint_id": "delete-user-force",
      "verdict": "reject",
      "reason": "Function name 'force_delete_user' is ambiguous. Use 'permanently_delete_user' to be explicit about irreversibility."
    }
  ],
  "global_suggestions": [
    "Add a retry configuration option to the client constructor",
    "Consider adding request/response interceptor hooks",
    "The error types look good — consider adding a TimeoutError"
  ]
}
```

3. **Process the review:**
   - `reject` → must be fixed before proceeding
   - `suggest` → should be addressed; document reason if deferring
   - `approve` → good to go

4. **Iterate if needed.** If there are rejections, update the plan and re-submit
   for review. Maximum 3 review rounds — after that, proceed with any remaining
   suggestions documented as TODOs.

### Review Round Tracking

Track review rounds in the plan metadata:

```json
{
  "review_rounds": [
    {
      "round": 1,
      "verdict": "rejected",
      "rejections": 2,
      "suggestions": 5,
      "approvals": 15
    },
    {
      "round": 2,
      "verdict": "approved_with_suggestions",
      "rejections": 0,
      "suggestions": 2,
      "approvals": 20
    }
  ]
}
```

---

## Review Criteria

The reviewer evaluates against these dimensions:

### 1. Naming Consistency
- Are all functions named consistently within each resource?
- Do names follow language idioms? (no `getUser` in Python, no `get_user` in TypeScript)
- Are resources named logically? (plural nouns for collections)

### 2. Type Safety
- Are all parameters typed? (no `any` / `Any` unless truly unknown)
- Are optional fields marked correctly?
- Are nullable fields handled? (distinct from optional in many languages)
- Do generic types make sense? (e.g., `PaginatedResponse<T>`)

### 3. Error Handling
- Is every observed error status code mapped to an exception type?
- Are retriable errors identified? (429, 503)
- Is there a base exception class for catching all SDK errors?

### 4. Completeness
- Does every discovered endpoint have a corresponding function?
- Are CRUD operations complete where applicable?
- Are pagination helpers provided for list endpoints?

### 5. Authentication
- Is the auth flow correctly represented?
- Can tokens be refreshed automatically?
- Are there clear errors for auth failures?

### 6. Testability
- Does every function have a mock spec?
- Are edge cases identified? (empty responses, max pagination, error states)
- Can the SDK be used entirely offline with mocks?
