# API Design Rules

> Backend API design rules for Claude Code. Apply to all Node.js backend projects.

## RESTful Conventions

- Use **nouns** for resource endpoints, **not verbs**:

```ts
// ❌ Incorrect
GET  /getUsers
POST /createUser
PUT  /updateUser/123

// ✅ Correct
GET    /users
POST   /users
PUT    /users/123
PATCH  /users/123
DELETE /users/123
```

- Use **plural** nouns for collection endpoints:

```ts
// ❌ Incorrect
GET /user
GET /user/123

// ✅ Correct
GET /users
GET /users/123
```

## Nested Resources

- Express parent-child relationships through nesting (max 2 levels deep):

```ts
// ✅ Correct — nested resource
GET  /users/123/orders
GET  /users/123/orders/456
POST /users/123/orders

// ❌ Incorrect — too deeply nested
GET /users/123/orders/456/items/789/reviews
```

- For deeply nested resources, use top-level endpoints with query filters:

```ts
// ✅ Correct — flat with filter
GET /reviews?orderId=456
GET /order-items?orderId=456
```

## HTTP Methods

| Method | Purpose | Idempotent | Request Body |
|--------|---------|------------|--------------|
| `GET` | Retrieve resource(s) | Yes | No |
| `POST` | Create a new resource | No | Yes |
| `PUT` | Replace entire resource | Yes | Yes |
| `PATCH` | Partial update | Yes | Yes |
| `DELETE` | Remove a resource | Yes | No |

## HTTP Status Codes

Use appropriate status codes — never return `200` for everything:

| Code | When to Use |
|------|-------------|
| `200 OK` | Successful GET, PUT, PATCH, DELETE |
| `201 Created` | Successful POST that creates a resource |
| `204 No Content` | Successful DELETE with no response body |
| `400 Bad Request` | Validation error, malformed request |
| `401 Unauthorized` | Missing or invalid authentication |
| `403 Forbidden` | Authenticated but insufficient permissions |
| `404 Not Found` | Resource does not exist |
| `409 Conflict` | Duplicate resource, state conflict |
| `422 Unprocessable Entity` | Valid syntax but semantic errors |
| `429 Too Many Requests` | Rate limit exceeded |
| `500 Internal Server Error` | Unhandled server error |

```ts
// ❌ Incorrect
return res.status(200).json({ error: "User not found" });

// ✅ Correct
return res.status(404).json({ message: "User not found" });
```

## Response Format

- Use a **consistent response envelope** across all endpoints:

```ts
// Success response
{
  "data": { "id": 1, "name": "John" },
  "meta": { "timestamp": "2026-01-15T10:30:00Z" }
}

// List response with pagination
{
  "data": [{ "id": 1, "name": "John" }],
  "meta": {
    "total": 100,
    "page": 1,
    "limit": 20,
    "totalPages": 5
  }
}

// Error response
{
  "message": "Validation failed",
  "errors": [
    { "field": "email", "message": "Invalid email format" }
  ]
}
```

## Pagination

- Always paginate list endpoints — never return unbounded collections:

```ts
// ✅ Correct — cursor-based or offset-based pagination
GET /users?page=1&limit=20
GET /users?cursor=abc123&limit=20
```

- Default to `limit=20`, max `limit=100`.
- Return pagination metadata in the response (`total`, `page`, `totalPages` or `nextCursor`).

## Filtering, Sorting, Searching

```ts
// Filtering
GET /users?role=admin&isActive=true

// Sorting
GET /users?sort=createdAt&order=desc

// Searching
GET /users?search=john

// Combined
GET /users?role=admin&sort=name&order=asc&page=2&limit=10
```

## Versioning

- Use URL-based versioning for public APIs:

```ts
// ✅ Correct
GET /api/v1/users
GET /api/v2/users
```

- Internal APIs may skip versioning if clients are tightly coupled.

## Naming Conventions

- Endpoints: **kebab-case** for multi-word resources:

```ts
// ❌ Incorrect
GET /userProfiles
GET /user_profiles

// ✅ Correct
GET /user-profiles
```

- Query parameters: **camelCase**:

```ts
GET /users?sortBy=createdAt&isActive=true
```

- Response fields: **camelCase**:

```json
{ "firstName": "John", "lastName": "Doe", "createdAt": "2026-01-15" }
```

## Rules

- Design APIs from the consumer's perspective — make them intuitive and predictable.
- Be consistent — once you choose a pattern, apply it everywhere.
- Document every endpoint with request/response examples.
- Return only the data the client needs — avoid over-fetching.
- Use proper content types: `application/json` for JSON, `multipart/form-data` for file uploads.
- Never expose internal IDs (auto-increment) if security is a concern — use UUIDs.
