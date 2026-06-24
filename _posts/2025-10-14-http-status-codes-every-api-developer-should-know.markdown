---
layout: post
title: "HTTP Status Codes Every API Developer Should Know"
date: 2025-10-14 11:20:00 +0000
categories: rest-api http status-codes
image: /assets/images/http-status-codes-every-api-developer-should-know.png
image_alt: "HTTP status code cards grouped into success, client error, and server error categories for REST APIs"
---

HTTP status codes are the first line of communication between a REST API and its clients. Before a client parses JSON, it sees whether the request succeeded, failed because of client input, or failed because of the server.

You do not need to memorize every code. You do need to use the common ones consistently.

## 200 OK

Use `200 OK` when a request succeeds and returns a response body.

Common examples:

```http
GET /users/123
PATCH /users/123
```

If the client expects a resource representation back, `200` is usually the right success code.

## 201 Created

Use `201 Created` when the request created a new resource.

```http
POST /users
```

If possible, include a `Location` header pointing to the new resource:

```http
HTTP/1.1 201 Created
Location: /users/123
```

The response body can include the created resource so the client does not need to immediately fetch it again.

## 202 Accepted

Use `202 Accepted` when the request was accepted but processing is not complete.

This is useful for long-running jobs:

```http
POST /exports
```

Example response:

```json
{
  "job_id": "exp_123",
  "status": "queued",
  "status_url": "/exports/exp_123"
}
```

Do not use `202` if the work already finished. Use it when the client needs to poll, subscribe, or check later.

## 204 No Content

Use `204 No Content` when a request succeeds and there is no response body.

Common examples:

```http
DELETE /users/123
PATCH /settings/email
```

A `204` response should not include a JSON body. If you want to return the updated resource, use `200 OK` instead.

## 400 Bad Request

Use `400 Bad Request` when the request cannot be understood or is structurally invalid.

Examples:

- Invalid JSON
- Missing required query parameter
- Wrong parameter type
- Unsupported sort field

Return a body that explains what the client can fix:

```json
{
  "error": {
    "code": "invalid_query_parameter",
    "message": "limit must be a number between 1 and 100."
  }
}
```

## 401 Unauthorized

Despite the name, `401 Unauthorized` means the client is not authenticated.

Use it when:

- No credentials were provided
- The token is expired
- The API key is invalid

For bearer tokens, include a `WWW-Authenticate` header when useful:

```http
WWW-Authenticate: Bearer error="invalid_token"
```

## 403 Forbidden

Use `403 Forbidden` when the caller is authenticated but not allowed to perform the action.

Example:

```http
DELETE /admin/users/123
```

If the token is valid but lacks the required role or scope, `403` is more accurate than `401`.

## 404 Not Found

Use `404 Not Found` when the resource does not exist or should not be visible to the caller.

Some APIs intentionally return `404` instead of `403` for private resources to avoid revealing that the resource exists. That is fine if it is consistent and documented.

## 409 Conflict

Use `409 Conflict` when the request conflicts with current resource state.

Examples:

- Creating a user with an email that already exists
- Updating a stale version of a document
- Canceling an order that has already shipped

The response should explain the conflict, not just say "conflict".

## 422 Unprocessable Entity

Use `422 Unprocessable Entity` when the request syntax is valid but the content fails validation or business rules.

Example:

```json
{
  "error": {
    "code": "validation_failed",
    "message": "One or more fields are invalid.",
    "details": [
      { "field": "email", "message": "Must be a valid email address." }
    ]
  }
}
```

Some teams use `400` for all validation failures. That can be fine. The key is to choose a rule and apply it consistently.

## 429 Too Many Requests

Use `429 Too Many Requests` when a client exceeds a rate limit.

Include retry information:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
```

This lets clients back off instead of hammering the API.

## 500, 502, 503, and 504

Use 5xx codes for server-side failures.

- `500 Internal Server Error` means an unexpected server error occurred
- `502 Bad Gateway` means an upstream service returned an invalid response
- `503 Service Unavailable` means the service is temporarily unavailable
- `504 Gateway Timeout` means an upstream service timed out

Clients usually cannot fix 5xx errors. Return a safe message and a request id, then log the details internally.

## A practical rule of thumb

Ask three questions:

1. Did the request succeed? Use `2xx`.
2. Did the client send something wrong? Use `4xx`.
3. Did the server or an upstream dependency fail? Use `5xx`.

That simple split prevents many status code mistakes.

## Closing thoughts

Status codes are small, but they shape how clients handle your API. Use them honestly, document your rules, and pair them with useful response bodies.

If every endpoint uses the same status code vocabulary, integrations become easier to write, test, and debug.
