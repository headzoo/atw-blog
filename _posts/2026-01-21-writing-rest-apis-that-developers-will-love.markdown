---
layout: post
title: "Writing REST APIs that Developers Will Love"
date: 2026-01-21 12:00:00 +0000
categories: rest-api design
image: /assets/images/writing-rest-apis-developers-love.png
image_alt: "A REST API design diagram with resource paths, HTTP method chips, and a JSON response panel"
---

Developers judge an API quickly. The first request either feels obvious or it does not. A URL that makes sense, a response that matches expectations, and an error message that actually helps — those details add up to an API people want to use.

This guide covers practical design choices that make REST APIs easier to learn, harder to misuse, and more pleasant to integrate with.

## Start with resources, not actions

REST APIs work best when URLs represent **nouns** (resources), not verbs (operations).

Good:

```
GET    /users/42
POST   /users
PATCH  /users/42
DELETE /users/42
```

Avoid:

```
POST /getUser
POST /createUser
POST /deleteUserById
```

Use HTTP methods to express intent. `GET` reads, `POST` creates, `PUT` replaces, `PATCH` updates partially, and `DELETE` removes. When every endpoint is a `POST` with a different action name, clients lose the predictability that makes REST worth using.

Nested resources are fine when the relationship is clear:

```
GET /users/42/posts
GET /users/42/posts/7
```

Keep nesting shallow. If a path grows past two levels, consider flattening:

```
GET /posts?user_id=42
```

## Use HTTP status codes consistently

Status codes communicate outcome before the client parses the body. Use them consistently across the API:

| Code | When to use |
|------|-------------|
| `200 OK` | Successful read or update with a response body |
| `201 Created` | Resource created; include a `Location` header when possible |
| `204 No Content` | Successful delete or update with no body |
| `400 Bad Request` | Invalid input the client can fix |
| `401 Unauthorized` | Missing or invalid authentication |
| `403 Forbidden` | Authenticated but not allowed |
| `404 Not Found` | Resource does not exist (or is not visible to this caller) |
| `409 Conflict` | Duplicate or state conflict |
| `422 Unprocessable Entity` | Valid syntax but business rule failure |
| `429 Too Many Requests` | Rate limit exceeded |
| `500 Internal Server Error` | Unexpected server failure |

Do not return `200 OK` with an error payload. Clients that rely on status codes — and most HTTP libraries do — will treat the request as successful. Pick one error shape and one set of status rules, then apply them everywhere.

## Design errors developers can act on

A useful error response answers three questions:

1. What went wrong?
2. Which field or parameter caused it?
3. What should the client do next?

Example:

```json
{
  "error": {
    "code": "validation_failed",
    "message": "One or more fields are invalid.",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address."
      }
    ]
  }
}
```

Stable `code` values matter more than clever messages. Messages can change; codes should not. Document every error code in your API reference so integrators know what to handle programmatically.

## Keep request and response shapes predictable

Developers build mental models from patterns. Reuse them.

- Use **snake_case** or **camelCase** consistently — pick one and stick with it across every endpoint
- Return the same resource representation from `GET`, `POST`, and `PATCH` where possible
- Include an `id` field on every resource
- Use ISO 8601 for timestamps: `2026-01-21T12:00:00Z`
- Represent enums as strings, not opaque integers

For collections, return a consistent wrapper:

```json
{
  "data": [
    { "id": 1, "title": "First post" },
    { "id": 2, "title": "Second post" }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 142
  }
}
```

Avoid surprising clients with different field names or nesting on similar endpoints. Consistency reduces integration time more than any single clever shortcut.

## Support pagination, filtering, and sorting

List endpoints that return unbounded arrays do not scale — for your database or for the client waiting on a multi-megabyte response.

Common query parameters:

```
GET /posts?page=2&limit=20
GET /posts?sort=-created_at
GET /posts?status=published&author_id=42
```

Cursor-based pagination works better for large or frequently changing datasets:

```
GET /posts?cursor=eyJpZCI6MTIzfQ&limit=20
```

Document defaults (`limit=20`, `sort=created_at`) and maximums (`limit` capped at 100). Clients should not have to guess whether pagination is offset- or cursor-based.

## Make authentication straightforward

Most APIs use bearer tokens, API keys, or OAuth 2.0. Whatever you choose, make the contract obvious:

- Document where credentials go (`Authorization: Bearer …` or a named header)
- Return `401` when credentials are missing or invalid
- Return `403` when credentials are valid but insufficient
- Avoid mixing auth schemes across endpoints without a clear reason

If tokens expire, provide a refresh flow or clear instructions for obtaining a new token. Silent expiry with cryptic `403` responses is a common source of support tickets.

## Version your API deliberately

APIs change. Versioning gives you room to evolve without breaking existing clients.

Common approaches:

- **URL prefix** — `/v1/users` (simple, visible, widely understood)
- **Header** — `Accept: application/vnd.myapp.v1+json` (clean URLs, harder to discover)
- **Query parameter** — `/users?version=1` (works, but easy to forget)

Pick one strategy and apply it consistently. When you ship breaking changes, increment the version and keep the old version running long enough for clients to migrate. Deprecation headers (`Deprecation`, `Sunset`) give integrators a timeline instead of a surprise.

## Write documentation that matches the API

Reference docs should show real requests and real responses — not abstract field lists alone.

Every endpoint page should include:

- Method and path
- Required and optional parameters
- Example request with headers and body
- Example success response
- Example error responses
- Auth requirements

OpenAPI (Swagger) specs are worth the investment. They power interactive docs, client generation, and contract tests. Keep the spec close to the code or generate it from source so it does not drift.

## Provide working examples

Developers learn fastest by sending a request and getting a response. Lower the barrier:

- Publish a **Postman collection** or equivalent for your API
- Include curl examples in documentation
- Offer a **sandbox environment** with test credentials
- Seed sandbox data so list endpoints return meaningful results on the first call

An API with a five-minute quickstart beats one that requires reading thirty pages before the first successful request.

## Design for backwards compatibility

Breaking changes erode trust. Prefer additive changes:

- Add new optional fields instead of renaming existing ones
- Add new endpoints instead of changing semantics on old ones
- Default new behavior off or preserve old behavior when possible

When a change is unavoidable, communicate early:

1. Mark the old behavior as deprecated
2. Document the replacement
3. Set a sunset date
4. Monitor usage of deprecated endpoints before removal

Clients forgive evolution. They do not forgive silent breakage.

## A practical design checklist

Before shipping an endpoint, run through this list:

- [ ] URL represents a resource, not an action
- [ ] HTTP method matches the operation
- [ ] Status codes used consistently
- [ ] Error responses include a stable code and actionable details
- [ ] Request and response field naming matches the rest of the API
- [ ] List endpoints support pagination
- [ ] Auth requirements are documented and enforced consistently
- [ ] Examples in docs match actual behavior
- [ ] Breaking changes are versioned or deprecated with notice

## Closing thoughts

Developers do not love APIs because they are RESTful on paper. They love APIs that are **predictable**, **consistent**, and **easy to debug** when something goes wrong.

Start with clear resource URLs and honest status codes. Add structured errors, stable pagination, and documentation with real examples. Treat backwards compatibility as a feature, not an afterthought. Those choices compound — each good decision makes the next integration faster and each bad one becomes someone else's workaround.

Build the API you would want to integrate with on a Friday afternoon. That is the bar worth hitting.
