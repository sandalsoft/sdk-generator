# SDK Code Generation

This document covers how to generate SDK code from the approved plan.
Each language target follows the same logical structure but uses idiomatic patterns.

## Table of Contents
- [Generation Order](#generation-order)
- [Python SDK](#python-sdk)
- [TypeScript SDK](#typescript-sdk)
- [Mock and Stub Generation](#mock-and-stub-generation)
- [Commit Strategy](#commit-strategy)

---

## Generation Order

Generate code in this order for each language — later steps depend on earlier ones:

1. **Error types** — Exception/error classes
2. **Shared models/types** — Data classes / interfaces
3. **Base client** — HTTP client with auth and error handling
4. **Resource modules** — One module per resource group, containing all functions
5. **Mocks and stubs** — Test fixtures for every function
6. **Package metadata** — pyproject.toml, package.json, etc.

---

## Python SDK

### Directory Structure

```
python/
├── pyproject.toml
├── src/
│   └── {package_name}/
│       ├── __init__.py          # Re-exports client + models
│       ├── client.py            # Base HTTP client
│       ├── models.py            # Pydantic data models
│       ├── errors.py            # Exception classes
│       ├── types.py             # Type aliases, generics
│       ├── resources/
│       │   ├── __init__.py
│       │   ├── users.py         # UsersResource class
│       │   └── projects.py      # ProjectsResource class
│       └── _pagination.py       # Pagination helpers
├── tests/
│   ├── conftest.py              # Shared fixtures
│   ├── mocks/
│   │   ├── __init__.py
│   │   ├── users.py             # Mock factories for user responses
│   │   └── projects.py
│   ├── test_users.py            # Unit tests for users resource
│   ├── test_projects.py
│   └── e2e/
│       ├── conftest.py          # E2E fixtures (real client setup)
│       ├── test_users_e2e.py
│       └── test_projects_e2e.py
└── README.md
```

### Models (Pydantic)

```python
# models.py
from __future__ import annotations
from pydantic import BaseModel, Field
from typing import Generic, TypeVar, Optional
from datetime import datetime

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    data: list[T]
    total: int
    page: int

class User(BaseModel):
    id: str
    name: str
    email: Optional[str] = None
    created_at: Optional[datetime] = None
```

### Error Types

```python
# errors.py
class PromptQLError(Exception):
    """Base exception for all SDK errors."""
    def __init__(self, message: str, status_code: int = None, response_body: dict = None):
        super().__init__(message)
        self.status_code = status_code
        self.response_body = response_body

class AuthenticationError(PromptQLError):
    """401 - Invalid or missing authentication."""
    pass

class PermissionError(PromptQLError):
    """403 - Insufficient permissions."""
    pass

class NotFoundError(PromptQLError):
    """404 - Resource not found."""
    pass

class ValidationError(PromptQLError):
    """422 - Request validation failed."""
    pass

class RateLimitError(PromptQLError):
    """429 - Rate limit exceeded."""
    def __init__(self, message, retry_after: int = None, **kwargs):
        super().__init__(message, **kwargs)
        self.retry_after = retry_after

ERROR_MAP = {
    401: AuthenticationError,
    403: PermissionError,
    404: NotFoundError,
    422: ValidationError,
    429: RateLimitError,
}
```

### Base Client

```python
# client.py
import httpx
from .errors import PromptQLError, ERROR_MAP

class PromptQLClient:
    def __init__(self, api_key: str, base_url: str = "https://api.promptql.com"):
        self.base_url = base_url.rstrip("/")
        self._client = httpx.Client(
            base_url=self.base_url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=30.0,
        )
        # Initialize resources
        self.users = UsersResource(self)
        self.projects = ProjectsResource(self)

    def _request(self, method: str, path: str, **kwargs) -> dict:
        response = self._client.request(method, path, **kwargs)
        if response.status_code >= 400:
            error_cls = ERROR_MAP.get(response.status_code, PromptQLError)
            body = response.json() if response.content else {}
            raise error_cls(
                message=body.get("message", f"HTTP {response.status_code}"),
                status_code=response.status_code,
                response_body=body,
            )
        if response.status_code == 204:
            return None
        return response.json()

    def close(self):
        self._client.close()

    def __enter__(self):
        return self

    def __exit__(self, *args):
        self.close()
```

### Resource Module Pattern

```python
# resources/users.py
from ..models import User, PaginatedResponse
from typing import Optional, Iterator

class UsersResource:
    def __init__(self, client):
        self._client = client

    def list(self, page: int = 1, limit: int = 20) -> PaginatedResponse[User]:
        """Retrieve a paginated list of users."""
        data = self._client._request("GET", "/api/v1/users", params={
            "page": page, "limit": limit
        })
        return PaginatedResponse[User](
            data=[User(**u) for u in data["data"]],
            total=data["total"],
            page=data["page"],
        )

    def list_all(self, limit: int = 20) -> Iterator[User]:
        """Auto-paginating iterator over all users."""
        page = 1
        while True:
            result = self.list(page=page, limit=limit)
            yield from result.data
            if page * limit >= result.total:
                break
            page += 1

    def get(self, user_id: str) -> User:
        """Retrieve a single user by ID."""
        data = self._client._request("GET", f"/api/v1/users/{user_id}")
        return User(**data)

    def create(self, name: str, email: Optional[str] = None) -> User:
        """Create a new user."""
        data = self._client._request("POST", "/api/v1/users", json={
            "name": name, "email": email
        })
        return User(**data)
```

---

## TypeScript SDK

### Directory Structure

```
typescript/
├── package.json
├── tsconfig.json
├── jest.config.js
├── src/
│   ├── index.ts                 # Re-exports
│   ├── client.ts                # Base HTTP client
│   ├── models.ts                # Interface definitions
│   ├── errors.ts                # Error classes
│   ├── resources/
│   │   ├── index.ts
│   │   ├── users.ts
│   │   └── projects.ts
│   └── pagination.ts
├── tests/
│   ├── mocks/
│   │   ├── users.ts
│   │   └── projects.ts
│   ├── users.test.ts
│   ├── projects.test.ts
│   └── e2e/
│       ├── users.e2e.test.ts
│       └── projects.e2e.test.ts
└── README.md
```

### Models (Interfaces)

```typescript
// models.ts
export interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
}

export interface User {
  id: string;
  name: string;
  email?: string | null;
  created_at?: string | null;
}

export interface CreateUserInput {
  name: string;
  email?: string;
}
```

### Error Types

```typescript
// errors.ts
export class PromptQLError extends Error {
  constructor(
    message: string,
    public statusCode?: number,
    public responseBody?: Record<string, unknown>
  ) {
    super(message);
    this.name = "PromptQLError";
  }
}

export class AuthenticationError extends PromptQLError {
  constructor(message: string, responseBody?: Record<string, unknown>) {
    super(message, 401, responseBody);
    this.name = "AuthenticationError";
  }
}

// ... same pattern for other error types

export const ERROR_MAP: Record<number, typeof PromptQLError> = {
  401: AuthenticationError,
  403: PermissionError,
  404: NotFoundError,
  422: ValidationError,
  429: RateLimitError,
};
```

### Resource Module Pattern

```typescript
// resources/users.ts
import { PromptQLClient } from "../client";
import { User, PaginatedResponse, CreateUserInput } from "../models";

export class UsersResource {
  constructor(private client: PromptQLClient) {}

  async list(page = 1, limit = 20): Promise<PaginatedResponse<User>> {
    return this.client.request<PaginatedResponse<User>>("GET", "/api/v1/users", {
      params: { page, limit },
    });
  }

  async *listAll(limit = 20): AsyncGenerator<User> {
    let page = 1;
    while (true) {
      const result = await this.list(page, limit);
      for (const item of result.data) yield item;
      if (page * limit >= result.total) break;
      page++;
    }
  }

  async get(userId: string): Promise<User> {
    return this.client.request<User>("GET", `/api/v1/users/${userId}`);
  }

  async create(input: CreateUserInput): Promise<User> {
    return this.client.request<User>("POST", "/api/v1/users", { body: input });
  }
}
```

---

## Mock and Stub Generation

For every SDK function, generate corresponding test infrastructure:

### Mock Factories

Create functions that return realistic fake data matching the schema:

```python
# tests/mocks/users.py
from src.promptql.models import User, PaginatedResponse
import uuid
from datetime import datetime

def mock_user(**overrides) -> dict:
    """Generate a mock user response dict."""
    return {
        "id": overrides.get("id", str(uuid.uuid4())),
        "name": overrides.get("name", "Test User"),
        "email": overrides.get("email", "test@example.com"),
        "created_at": overrides.get("created_at", datetime.now().isoformat()),
        **{k: v for k, v in overrides.items() if k not in ["id", "name", "email", "created_at"]}
    }

def mock_users_list(count: int = 3, page: int = 1, total: int = None) -> dict:
    """Generate a mock paginated users list response."""
    return {
        "data": [mock_user(name=f"User {i}") for i in range(count)],
        "total": total or count,
        "page": page,
    }

def mock_users_list_empty() -> dict:
    return mock_users_list(count=0, total=0)

def mock_users_list_single() -> dict:
    return mock_users_list(count=1, total=1)
```

### Request Stubs

Stubs that intercept HTTP calls and validate the outgoing request:

```python
# In test files, using respx for httpx mocking
import respx

@respx.mock
def test_list_users():
    route = respx.get("https://api.promptql.com/api/v1/users").mock(
        return_value=httpx.Response(200, json=mock_users_list())
    )
    client = PromptQLClient(api_key="test-key")
    result = client.users.list()

    assert route.called
    assert route.calls[0].request.headers["Authorization"] == "Bearer test-key"
    assert len(result.data) == 3
    assert isinstance(result.data[0], User)
```

---

## Commit Strategy

Commit in this order for each language:

1. `feat({lang}): add error types and shared models`
2. `feat({lang}): add base client with auth and error handling`
3. `feat({lang}): add {resource} resource with tests and mocks` (one per resource)
4. `feat({lang}): add pagination helpers`
5. `chore({lang}): add package metadata and README`

Each commit should leave the test suite in a passing state. If a resource's tests
aren't passing yet, don't commit it — fix first, then commit.
