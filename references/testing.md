# Testing Strategy

This document covers the two-layer testing approach and the autonomous bug-fix loop.

## Table of Contents
- [Testing Philosophy](#testing-philosophy)
- [Layer 1: Unit Tests](#layer-1-unit-tests)
- [Layer 2: End-to-End Tests](#layer-2-end-to-end-tests)
- [Auto-Fix Loop](#auto-fix-loop)
- [Test Coverage Requirements](#test-coverage-requirements)
- [Fixture Management](#fixture-management)

---

## Testing Philosophy

Every SDK function must have tests. The goal is confidence that:

1. The SDK correctly serializes requests (method, URL, headers, body, query params)
2. The SDK correctly deserializes responses into typed objects
3. Error responses are caught and raised as typed exceptions
4. Pagination works correctly across pages
5. Auth tokens are sent correctly
6. The actual live API returns what the SDK expects (if testable)

Tests are not optional. They are generated alongside the SDK code and are part of
the same commit. If a function doesn't have tests, it's not done.

---

## Layer 1: Unit Tests

Unit tests run entirely offline using mocks and stubs. They validate the SDK's
logic without making any network calls.

### What Unit Tests Cover

For each SDK function, test:

1. **Happy path:** Call with valid params → correct request sent, correct response parsed
2. **Parameter handling:** Optional params omitted → not sent in request
3. **Error handling:** Mock a 401/404/422/429/500 → correct exception type raised
4. **Edge cases:** Empty responses, null fields, max-length strings
5. **Type validation:** Response fields map to correct Python/TS types

### Python Unit Test Template

```python
# tests/test_users.py
import pytest
import httpx
import respx
from promptql import PromptQLClient
from promptql.models import User, PaginatedResponse
from promptql.errors import AuthenticationError, NotFoundError
from tests.mocks.users import mock_user, mock_users_list, mock_users_list_empty

BASE_URL = "https://api.promptql.com"

@pytest.fixture
def client():
    return PromptQLClient(api_key="test-key", base_url=BASE_URL)

class TestListUsers:
    @respx.mock
    def test_returns_paginated_users(self, client):
        mock_data = mock_users_list(count=3, total=10, page=1)
        respx.get(f"{BASE_URL}/api/v1/users").mock(
            return_value=httpx.Response(200, json=mock_data)
        )
        result = client.users.list(page=1, limit=20)

        assert isinstance(result, PaginatedResponse)
        assert len(result.data) == 3
        assert result.total == 10
        assert all(isinstance(u, User) for u in result.data)

    @respx.mock
    def test_sends_correct_query_params(self, client):
        respx.get(f"{BASE_URL}/api/v1/users").mock(
            return_value=httpx.Response(200, json=mock_users_list())
        )
        client.users.list(page=2, limit=50)

        request = respx.calls[0].request
        assert "page=2" in str(request.url)
        assert "limit=50" in str(request.url)

    @respx.mock
    def test_sends_auth_header(self, client):
        respx.get(f"{BASE_URL}/api/v1/users").mock(
            return_value=httpx.Response(200, json=mock_users_list())
        )
        client.users.list()

        assert respx.calls[0].request.headers["Authorization"] == "Bearer test-key"

    @respx.mock
    def test_empty_list(self, client):
        respx.get(f"{BASE_URL}/api/v1/users").mock(
            return_value=httpx.Response(200, json=mock_users_list_empty())
        )
        result = client.users.list()
        assert result.data == []
        assert result.total == 0

    @respx.mock
    def test_401_raises_auth_error(self, client):
        respx.get(f"{BASE_URL}/api/v1/users").mock(
            return_value=httpx.Response(401, json={"message": "Invalid token"})
        )
        with pytest.raises(AuthenticationError) as exc_info:
            client.users.list()
        assert exc_info.value.status_code == 401

class TestGetUser:
    @respx.mock
    def test_returns_user(self, client):
        user_data = mock_user(id="usr_123", name="Alice")
        respx.get(f"{BASE_URL}/api/v1/users/usr_123").mock(
            return_value=httpx.Response(200, json=user_data)
        )
        user = client.users.get("usr_123")

        assert isinstance(user, User)
        assert user.id == "usr_123"
        assert user.name == "Alice"

    @respx.mock
    def test_404_raises_not_found(self, client):
        respx.get(f"{BASE_URL}/api/v1/users/nonexistent").mock(
            return_value=httpx.Response(404, json={"message": "User not found"})
        )
        with pytest.raises(NotFoundError):
            client.users.get("nonexistent")
```

### TypeScript Unit Test Template

```typescript
// tests/users.test.ts
import axios from "axios";
import MockAdapter from "axios-mock-adapter";
import { PromptQLClient } from "../src/client";
import { mockUser, mockUsersList } from "./mocks/users";
import { AuthenticationError, NotFoundError } from "../src/errors";

let mock: MockAdapter;
let client: PromptQLClient;

beforeEach(() => {
  client = new PromptQLClient({ apiKey: "test-key" });
  mock = new MockAdapter(client.httpClient);
});

afterEach(() => mock.restore());

describe("users.list", () => {
  it("returns paginated users", async () => {
    mock.onGet("/api/v1/users").reply(200, mockUsersList(3));
    const result = await client.users.list();

    expect(result.data).toHaveLength(3);
    expect(result.total).toBe(3);
    expect(result.data[0]).toHaveProperty("id");
  });

  it("sends auth header", async () => {
    mock.onGet("/api/v1/users").reply(200, mockUsersList());
    await client.users.list();

    expect(mock.history.get[0].headers?.Authorization).toBe("Bearer test-key");
  });

  it("throws AuthenticationError on 401", async () => {
    mock.onGet("/api/v1/users").reply(401, { message: "Invalid token" });
    await expect(client.users.list()).rejects.toThrow(AuthenticationError);
  });
});
```

---

## Layer 2: End-to-End Tests

E2E tests make real API calls against the live service. They validate that the SDK
works against the actual API and that schemas haven't drifted.

### Safety Rules

Only run E2E tests against endpoints that are safe:
- **Always safe:** GET endpoints, read-only operations
- **Usually safe:** POST to idempotent endpoints (with unique test data)
- **Never run automatically:** DELETE, destructive mutations, billing operations

Mark unsafe tests with a skip decorator and a comment explaining why.

### E2E Test Structure

```python
# tests/e2e/test_users_e2e.py
import pytest
import os

# Skip entire module if no API key is set
pytestmark = pytest.mark.skipif(
    not os.getenv("PROMPTQL_API_KEY"),
    reason="PROMPTQL_API_KEY not set — skipping e2e tests"
)

@pytest.fixture
def live_client():
    return PromptQLClient(
        api_key=os.environ["PROMPTQL_API_KEY"],
        base_url=os.getenv("PROMPTQL_BASE_URL", "https://api.promptql.com"),
    )

class TestUsersE2E:
    def test_list_users_returns_valid_response(self, live_client):
        """Verify the live API returns data matching our schema."""
        result = live_client.users.list(limit=5)

        assert hasattr(result, "data")
        assert hasattr(result, "total")
        assert isinstance(result.data, list)
        if result.data:
            user = result.data[0]
            assert hasattr(user, "id")
            assert hasattr(user, "name")

    def test_get_user_with_valid_id(self, live_client):
        """Fetch the first user from the list, then get by ID."""
        listing = live_client.users.list(limit=1)
        if not listing.data:
            pytest.skip("No users available for testing")

        user = live_client.users.get(listing.data[0].id)
        assert user.id == listing.data[0].id
```

### Schema Drift Detection

E2E tests serve double duty: they validate the SDK AND detect when the API has changed.

```python
def test_response_schema_matches_model(self, live_client):
    """Check that the live response has all expected fields."""
    import httpx

    # Make a raw request to see the actual response shape
    raw = httpx.get(
        f"{live_client.base_url}/api/v1/users",
        headers={"Authorization": f"Bearer {live_client.api_key}"},
        params={"limit": 1}
    )
    body = raw.json()

    # Check for new fields we don't know about
    if body.get("data"):
        known_fields = {"id", "name", "email", "created_at"}
        actual_fields = set(body["data"][0].keys())
        new_fields = actual_fields - known_fields
        if new_fields:
            pytest.fail(
                f"API returned unknown fields: {new_fields}. "
                f"The SDK models may need updating."
            )
```

---

## Auto-Fix Loop

When tests fail, the agent should attempt to fix them automatically before
requesting human intervention.

### Algorithm

```
MAX_ATTEMPTS = 5

function auto_fix(test_suite):
    for attempt in 1..MAX_ATTEMPTS:
        result = run_tests(test_suite)

        if result.all_passed:
            log(f"All tests passing after {attempt} attempt(s)")
            git_commit(f"fix: resolve test failures (attempt {attempt})")
            return SUCCESS

        for failure in result.failures:
            diagnosis = diagnose_failure(failure)

            match diagnosis.category:
                case "wrong_url":
                    # SDK is calling wrong endpoint path
                    fix_endpoint_path(diagnosis)

                case "wrong_type":
                    # Response field has different type than expected
                    fix_model_type(diagnosis)

                case "missing_field":
                    # Response has field we didn't model
                    add_field_to_model(diagnosis)

                case "wrong_status_handling":
                    # Error status code not handled correctly
                    fix_error_mapping(diagnosis)

                case "mock_mismatch":
                    # Mock doesn't match actual response shape
                    fix_mock(diagnosis)

                case "serialization_error":
                    # Request body not serialized correctly
                    fix_serialization(diagnosis)

                case "unknown":
                    # Can't auto-diagnose — leave as TODO
                    add_todo_comment(failure)
                    skip_test(failure)

    log(f"Could not fix all tests after {MAX_ATTEMPTS} attempts")
    log(f"Remaining failures: {result.failures}")
    return PARTIAL_SUCCESS
```

### Diagnosis Heuristics

| Error Pattern | Likely Cause | Fix Strategy |
|--------------|-------------|-------------|
| `AssertionError: expected 'GET' got 'POST'` | Wrong HTTP method in SDK | Fix method in resource module |
| `ValidationError: field 'x' is required` | Model marks field required but API doesn't send it | Make field optional |
| `KeyError: 'data'` | Response shape different than expected | Update deserialization logic |
| `TypeError: 'NoneType' is not subscriptable` | Null response not handled | Add null check |
| `ConnectionError` in unit test | Mock not set up correctly | Fix mock URL/pattern |
| `422 Unprocessable Entity` in e2e | Request body wrong shape | Fix serialization |

### Commit After Fix

Every successful fix gets its own commit with a descriptive message:

```
fix(python): make User.email optional — API returns null for some users
fix(typescript): correct endpoint path for deleteProject
fix(python): handle 204 No Content response for delete operations
```

---

## Test Coverage Requirements

Minimum coverage per SDK function:

| Test Type | Required | Notes |
|-----------|----------|-------|
| Happy path | Yes | Basic call with valid params |
| Auth header sent | Yes | Verify Bearer token present |
| Error handling (at least one) | Yes | 401 or 404 |
| Optional params | Yes if function has them | Omitted params not sent |
| Empty response | Yes for list endpoints | Empty array case |
| E2E happy path | Yes if endpoint is safe | Real API call |

---

## Fixture Management

### Shared Fixtures (conftest.py)

```python
# tests/conftest.py
import pytest
from promptql import PromptQLClient

@pytest.fixture
def client():
    """Offline client for unit tests."""
    return PromptQLClient(api_key="test-key-unit")

@pytest.fixture
def mock_base_url():
    return "https://api.promptql.com"
```

### Sample Payload Files

For complex responses, store sample payloads as JSON files:

```
tests/
├── fixtures/
│   ├── users_list_response.json
│   ├── users_list_empty.json
│   ├── user_detail_response.json
│   └── error_401_response.json
```

Load them in mocks:

```python
import json
from pathlib import Path

FIXTURES = Path(__file__).parent / "fixtures"

def load_fixture(name: str) -> dict:
    return json.loads((FIXTURES / name).read_text())
```
