# API Design (2025)

> **Last updated**: January 2025
> **Scope**: REST, GraphQL, versioning, error handling

---

## Philosophy

APIs are contracts. Design for the consumer, not the implementation. Be consistent, predictable, and well-documented.

---

## REST Conventions

### URL Structure

```
/<resource>                 # Collection
/<resource>/:id             # Single item
/<resource>/:id/<sub>       # Sub-resource
```

### HTTP Methods

| Method | Use | Idempotent |
|--------|-----|------------|
| GET | Read | Yes |
| POST | Create | No |
| PUT | Replace | Yes |
| PATCH | Partial update | Yes |
| DELETE | Delete | Yes |

### Examples

```
GET    /users              # List users
GET    /users/123          # Get user 123
POST   /users              # Create user
PATCH  /users/123          # Update user 123
DELETE /users/123          # Delete user 123

GET    /users/123/orders   # Get user's orders
POST   /users/123/orders   # Create order for user
```

---

## Response Format

### Success

```json
{
  "data": { },
  "meta": {
    "page": 1,
    "totalPages": 10,
    "totalCount": 100
  }
}
```

### Error

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address"
      }
    ]
  }
}
```

---

## Status Codes

| Code | When to use |
|------|-------------|
| 200 | Success (GET, PATCH) |
| 201 | Created (POST) |
| 204 | No content (DELETE) |
| 400 | Bad request (validation error) |
| 401 | Unauthorized (no auth) |
| 403 | Forbidden (no permission) |
| 404 | Not found |
| 409 | Conflict (duplicate, etc.) |
| 422 | Unprocessable entity |
| 429 | Too many requests |
| 500 | Server error |

---

## Versioning

### URL Versioning (Recommended)

```
/api/v1/users
/api/v2/users
```

### Header Versioning

```
Accept: application/vnd.api+json; version=1
```

---

## Pagination

### Offset-based

```
GET /users?page=2&limit=20
```

### Cursor-based (Recommended for large datasets)

```
GET /users?cursor=abc123&limit=20
```

---

## Filtering & Sorting

```
GET /users?status=active&role=admin    # Filter
GET /users?sort=created_at:desc        # Sort
GET /users?fields=id,name,email        # Sparse fields
```

---

## Quick Reference

| Pattern | Example |
|---------|---------|
| List | `GET /resources` |
| Get one | `GET /resources/:id` |
| Create | `POST /resources` |
| Update | `PATCH /resources/:id` |
| Delete | `DELETE /resources/:id` |
| Sub-resource | `GET /resources/:id/items` |
| Search | `GET /resources?q=term` |
| Filter | `GET /resources?status=active` |
| Sort | `GET /resources?sort=name:asc` |
| Paginate | `GET /resources?page=1&limit=20` |
