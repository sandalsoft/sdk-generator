# Update Cycle

This document covers how to detect API changes and update the SDK incrementally
without regenerating everything from scratch.

## Table of Contents
- [When to Trigger an Update](#when-to-trigger-an-update)
- [Diff Algorithm](#diff-algorithm)
- [Change Categories](#change-categories)
- [Incremental Update Workflow](#incremental-update-workflow)
- [Deprecation Strategy](#deprecation-strategy)
- [Breaking Change Handling](#breaking-change-handling)

---

## When to Trigger an Update

Updates can be triggered by:

1. **User request:** "The API has been updated, please re-scan."
2. **New HAR file:** User provides a new HAR archive to diff against.
3. **Scheduled check:** Agent periodically re-crawls the live site.
4. **Failed e2e tests:** If existing e2e tests start failing, it may indicate API drift.

---

## Diff Algorithm

Compare the new `api-surface-new.json` against the existing `api-surface.json`:

```python
def diff_api_surfaces(old: dict, new: dict) -> dict:
    """
    Compare two API surface files and categorize changes.
    """
    old_endpoints = {ep["id"]: ep for ep in old["endpoints"]}
    new_endpoints = {ep["id"]: ep for ep in new["endpoints"]}

    old_ids = set(old_endpoints.keys())
    new_ids = set(new_endpoints.keys())

    diff = {
        "added": [],       # New endpoints not in old
        "removed": [],     # Old endpoints not in new
        "changed": [],     # Endpoints in both but different
        "unchanged": [],   # Identical endpoints
    }

    # New endpoints
    for ep_id in new_ids - old_ids:
        diff["added"].append(new_endpoints[ep_id])

    # Removed endpoints
    for ep_id in old_ids - new_ids:
        diff["removed"].append(old_endpoints[ep_id])

    # Changed vs unchanged
    for ep_id in old_ids & new_ids:
        changes = compare_endpoint(old_endpoints[ep_id], new_endpoints[ep_id])
        if changes:
            diff["changed"].append({
                "endpoint": new_endpoints[ep_id],
                "changes": changes,
            })
        else:
            diff["unchanged"].append(ep_id)

    return diff

def compare_endpoint(old_ep: dict, new_ep: dict) -> list[dict]:
    """Compare two versions of the same endpoint."""
    changes = []

    # Check response schema changes
    old_fields = extract_fields(old_ep.get("response_schema", {}))
    new_fields = extract_fields(new_ep.get("response_schema", {}))

    added_fields = new_fields - old_fields
    removed_fields = old_fields - new_fields

    if added_fields:
        changes.append({
            "type": "response_fields_added",
            "fields": list(added_fields),
            "breaking": False,
        })

    if removed_fields:
        changes.append({
            "type": "response_fields_removed",
            "fields": list(removed_fields),
            "breaking": True,
        })

    # Check query param changes
    old_params = {p["name"] for p in old_ep.get("query_params", [])}
    new_params = {p["name"] for p in new_ep.get("query_params", [])}

    if new_params - old_params:
        changes.append({
            "type": "query_params_added",
            "params": list(new_params - old_params),
            "breaking": False,
        })

    # Check for new required params (breaking!)
    new_required = {
        p["name"] for p in new_ep.get("query_params", [])
        if p.get("required") and p["name"] not in old_params
    }
    if new_required:
        changes.append({
            "type": "required_params_added",
            "params": list(new_required),
            "breaking": True,
        })

    # Check method change (very breaking)
    if old_ep["method"] != new_ep["method"]:
        changes.append({
            "type": "method_changed",
            "old": old_ep["method"],
            "new": new_ep["method"],
            "breaking": True,
        })

    # Check path change
    if old_ep["path_pattern"] != new_ep["path_pattern"]:
        changes.append({
            "type": "path_changed",
            "old": old_ep["path_pattern"],
            "new": new_ep["path_pattern"],
            "breaking": True,
        })

    return changes
```

### Output: `api-surface-diff.json`

```json
{
  "diffed_at": "2026-02-15T10:00:00Z",
  "summary": {
    "added": 3,
    "removed": 1,
    "changed": 2,
    "unchanged": 18,
    "has_breaking_changes": true
  },
  "added": [...],
  "removed": [...],
  "changed": [
    {
      "endpoint": { "id": "get-users-list", "..." },
      "changes": [
        { "type": "response_fields_added", "fields": ["role", "last_login"], "breaking": false },
        { "type": "query_params_added", "params": ["role_filter"], "breaking": false }
      ]
    }
  ]
}
```

---

## Change Categories

| Category | Examples | SDK Impact | Version Bump |
|----------|----------|-----------|-------------|
| **Additive** | New endpoint, new optional field, new optional param | Add function/field, no existing code changes | Minor |
| **Modification** | Field type changed, field became required | Update types, may break existing code | Major if breaking |
| **Removal** | Endpoint removed, field removed | Deprecate then remove | Major |
| **Behavioral** | Different error codes, changed pagination | Update error handling, pagination logic | Patch or Minor |

---

## Incremental Update Workflow

### Step 1: Re-discover

Run the discovery phase again, saving to `api-surface-new.json`.

### Step 2: Diff

```bash
# Compare old and new
python scripts/diff_surfaces.py api-surface.json api-surface-new.json > api-surface-diff.json
```

### Step 3: Update Plan

For each change category, update the `sdk-plan.json`:

**Added endpoints:**
- Create new function entries in the plan
- Run through LLM review (same as initial planning)
- Add mock specs

**Changed endpoints:**
- Update existing function entries
- If return type changed → update models
- If new params added → update function signature
- Quick LLM review of changes only

**Removed endpoints:**
- Mark functions as deprecated (do NOT delete immediately)
- Add `@deprecated` decorator / JSDoc tag
- Plan removal for next major version

### Step 4: Generate Updates

For each affected resource module:

1. Update model types (new fields, changed types)
2. Update function signatures (new params)
3. Add new functions for new endpoints
4. Update mocks to match new schemas
5. Update unit tests
6. Add new e2e tests
7. Run auto-fix loop

### Step 5: Validate

Run the full test suite — existing + new tests:

```bash
# All existing tests must still pass (backward compatibility)
pytest python/tests/ -v

# New tests must pass
pytest python/tests/ -k "new_endpoint" -v

# E2E if available
pytest python/tests/e2e/ -v --e2e
```

### Step 6: Finalize

```bash
# Replace old surface with new
cp api-surface-new.json api-surface.json
rm api-surface-new.json

# Update changelog
# Update version in pyproject.toml / package.json

git add -A
git commit -m "feat: API update {date} — {summary}"
```

---

## Deprecation Strategy

When an endpoint is removed from the API, don't immediately remove it from the SDK.
Instead, follow a deprecation cycle:

### Phase 1: Mark as Deprecated

```python
# Python
import warnings

def get_user_v1(self, user_id: str) -> User:
    """Retrieve a user by ID.

    .. deprecated:: 0.3.0
        Use :meth:`get_user` instead. Will be removed in v1.0.0.
    """
    warnings.warn(
        "get_user_v1 is deprecated, use get_user instead",
        DeprecationWarning,
        stacklevel=2,
    )
    return self.get_user(user_id)
```

```typescript
// TypeScript
/**
 * @deprecated Use `getUser()` instead. Will be removed in v1.0.0.
 */
async getUserV1(userId: string): Promise<User> {
  console.warn("getUserV1 is deprecated, use getUser instead");
  return this.getUser(userId);
}
```

### Phase 2: Remove in Next Major Version

- Remove the deprecated function
- Remove its tests and mocks
- Update the changelog with "BREAKING: removed get_user_v1"
- Bump major version

---

## Breaking Change Handling

When breaking changes are detected:

1. **Document them prominently** in the changelog under a "BREAKING CHANGES" section
2. **Provide migration guidance** in a `MIGRATION.md` file
3. **Consider providing both old and new** for one release cycle
4. **Bump major version** on merge

### Migration Guide Template

```markdown
# Migration Guide: v0.x → v1.0

## Breaking Changes

### User.email is now required
Previously `email` was optional and could be `None`. The API now always returns
an email address.

**Before:**
```python
if user.email:
    send_notification(user.email)
```

**After:**
```python
send_notification(user.email)  # Always present
```

### get_user_v1 removed
Use `get_user()` instead. The function signature is identical.
```
