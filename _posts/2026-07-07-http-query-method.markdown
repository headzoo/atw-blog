---
layout: post
title: "HTTP QUERY: The New Method for Safe Requests with a Body"
date: 2026-07-07 12:00:00 +0000
categories: rest-api http design
image: /assets/images/http-query-method.png
image_alt: "An HTTP QUERY request carrying a structured body with safe, idempotent, and cacheable method semantics"
---

For decades, HTTP left API designers with an awkward choice for complex read operations: squeeze everything into a `GET` URL, or use `POST` and hope clients, caches, and intermediaries treat the request as read-only.

That gap is now closed. In June 2026, the IETF published [RFC 10008](https://datatracker.ietf.org/doc/html/rfc10008), which defines a new HTTP method: **QUERY**. It combines the request body support of `POST` with the safe, idempotent semantics of `GET`.

If you build search APIs, analytics dashboards, reporting endpoints, or any read operation with structured filters too large for a query string, QUERY is worth understanding.

## The problem QUERY solves

A simple read looks like this:

```http
GET /feed?q=foo&limit=10&sort=-published HTTP/1.1
Host: example.org
```

That works until the query gets complicated:

- URL length limits vary across proxies, gateways, and servers
- Encoding structured data in a query string is verbose and error-prone
- Request URIs are more likely to be logged, bookmarked, or cached as distinct resources
- Every combination of filter inputs effectively becomes its own resource URL

Many teams reach for `POST` instead:

```http
POST /feed HTTP/1.1
Host: example.org
Content-Type: application/x-www-form-urlencoded

q=foo&limit=10&sort=-published
```

The body solves the payload problem, but the method does not promise safety or idempotency. A client cannot know from the method alone that repeating the request will not change server state. Caches and intermediaries cannot treat it like a read. Automatic retries become risky.

QUERY sits between those two patterns:

```http
QUERY /feed HTTP/1.1
Host: example.org
Content-Type: application/x-www-form-urlencoded

q=foo&limit=10&sort=-published
```

Same body as `POST`. Same safe, idempotent semantics as `GET`.

## GET vs QUERY vs POST

| | GET | QUERY | POST |
|---|-----|-------|------|
| **Safe** | yes | yes | potentially no |
| **Idempotent** | yes | yes | potentially no |
| **Request body** | no defined semantics | expected | expected |
| **Cacheable** | yes | yes | yes, but mainly for future GET or HEAD |
| **URI for the query itself** | yes (by definition) | optional (`Location`) | no |

Use `GET` when the query fits comfortably in the URL. Use `POST` when the operation may change state. Use `QUERY` when you need a body for a read-only operation and want HTTP semantics to reflect that.

## What QUERY actually means

A QUERY request asks the target resource to perform a **query operation within the scope of that resource**. The request content and its `Content-Type` define the query. The server decides how to interpret it.

Important rules from RFC 10008:

- **`Content-Type` is required.** If it is missing or inconsistent with the body, the server must reject the request.
- **QUERY is safe.** It must not change the state of the target resource.
- **QUERY is idempotent.** Clients and intermediaries can retry it after connection failures without worrying about side effects.
- **The URI query string still matters.** Parameters in the URL can still affect which resource is queried.

A direct response looks like any other successful read:

```http
QUERY /contacts HTTP/1.1
Host: example.org
Content-Type: application/x-www-form-urlencoded
Accept: application/json

select=surname,givenname,email&limit=10&match=%22email=*@example.*%22
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  { "surname": "Smith", "givenname": "John", "email": "smith@example.org" },
  { "surname": "Jones", "givenname": "Sally", "email": "sally.jones@example.com" }
]
```

For larger payloads, JSON works just as well:

```http
QUERY /reports/sales HTTP/1.1
Host: api.example.com
Content-Type: application/json
Accept: application/json

{
  "filters": {
    "region": ["US", "CA"],
    "status": "active"
  },
  "groupBy": ["month", "product"],
  "limit": 100
}
```

## Discovering QUERY support with Accept-Query

RFC 10008 introduces a response header that tells clients which query formats a resource accepts:

```http
Accept-Query: "application/jsonpath", application/sql;charset="UTF-8"
```

Servers can return `Accept-Query` on any response. Clients can also send `OPTIONS` or `HEAD` first to discover support before sending a full QUERY request:

```http
OPTIONS /contacts HTTP/1.1
Host: example.org
```

```http
HTTP/1.1 200 OK
Allow: GET, HEAD, OPTIONS, QUERY
Accept-Query: "application/x-www-form-urlencoded"
```

If the client sends an unsupported query format, the server should respond with `415 Unsupported Media Type` and can include `Accept-Query` to guide the client toward supported types.

Other useful status codes for QUERY:

| Code | When to use |
|------|-------------|
| `400 Bad Request` | Missing or inconsistent `Content-Type` |
| `415 Unsupported Media Type` | Query format not supported by the resource |
| `422 Unprocessable Content` | Valid syntax, but the query cannot be processed |
| `406 Not Acceptable` | Requested response format in `Accept` is not supported |

## Caching QUERY responses

QUERY responses are cacheable, but caching them is more involved than caching `GET`.

The cache key must incorporate the request body and related metadata — not just the URL. That means intermediaries need to read the full request content before deciding whether a cached response applies.

Caches may normalize request content to improve hit rates — for example, by removing insignificant JSON formatting differences — but incorrect normalization can produce false cache hits. Clients can send `Cache-Control: no-transform` to advise against that.

Because caching QUERY is inherently heavier than caching GET, RFC 10008 provides a useful escape hatch: servers can assign URIs to equivalent resources or query results.

## Location and Content-Location

Two response headers help clients avoid resending large query bodies:

**`Content-Location`** identifies a resource corresponding to the query results. A client can later `GET` that URI to retrieve the same result set.

**`Location`** identifies an equivalent resource for the QUERY request itself. A client can `GET` that URI to repeat the same query without resubmitting the body.

Example flow:

```http
QUERY /reports HTTP/1.1
Host: example.org
Content-Type: application/json

{ "filter": { "year": 2025 }, "groupBy": "month" }
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Location: /stored-queries/4815162342

{ "rows": [ ... ] }
```

Later:

```http
GET /stored-queries/4815162342 HTTP/1.1
Host: example.org
Accept: application/json
```

That pattern makes CDN caching and conditional requests much simpler. A client can also use `If-Modified-Since` or `ETag` on the stored resource instead of repeating the full QUERY payload.

## Conditional requests and redirects

Conditional QUERY requests work like conditional GET requests against the equivalent resource. If nothing changed, the server can return `304 Not Modified` instead of the full result body.

Redirection semantics mostly mirror other methods, with one notable difference: the POST-specific rule that turns some redirects into GET requests does **not** apply to QUERY. A `303 See Other` response means the client should follow up with GET to the URI in `Location`.

## Security and browser behavior

QUERY inherits the usual HTTP security considerations, with a few practical notes:

- Putting sensitive filter data in the body instead of the URL can reduce exposure in access logs and referrer headers
- If a server creates a temporary URI for query results, that URI should not embed sensitive portions of the original request content
- In browsers, QUERY requires a **CORS preflight** because it is not a CORS-safelisted method

That last point matters for browser-based API clients. A cross-origin QUERY call is not as lightweight as GET.

## Why the method is called QUERY

The HTTP method registry already contained safe, idempotent methods like `SEARCH`, `REPORT`, and `PROPFIND` from the WebDAV era. Early drafts of this specification used `SEARCH`, but the working group chose `QUERY` instead because:

- The WebDAV methods were tied to XML-centric semantics
- `QUERY` maps cleanly to the idea of a server-side query against a resource
- The name describes a general read operation, not just search

RFC 10008 was authored by Julian Reschke, James M. Snell, and Mike Bishop — contributors with deep history in HTTP standardization and CDN infrastructure.

## Where QUERY fits in real APIs

QUERY is most useful when a read operation needs structured input that does not belong in a URL:

- **Complex search filters** with nested conditions, arrays, or long expressions
- **Analytics and reporting** endpoints with group-by, aggregation, and date-range parameters
- **GraphQL-style reads** where the query document is too large for GET
- **SQL, JSONPath, or XSLT queries** against a resource-backed dataset

It is not a replacement for everyday CRUD. `GET /users/42` is still the right call for fetching a resource by ID. QUERY is for when the *query itself* is the payload.

## Adoption in 2026

RFC 10008 is a Proposed Standard, and ecosystem support is still rolling out. HTTP methods are just tokens, so much of the stack can experiment early — proxies, API gateways, and custom clients can send QUERY today if the server understands it.

Watch for support in:

- Server frameworks and reverse proxies
- API clients such as curl, HTTPie, Postman, and Bruno
- Browser `fetch()` implementations and CORS handling
- CDN and edge caching layers

Until your stack supports QUERY natively, the specification still gives API designers a clearer semantic model. Even if you continue using `POST /search` internally, documenting that endpoint as a safe, idempotent read helps clients, caches, and observability tools reason about it correctly.

## Practical guidance for API designers

If you are designing a new read endpoint with a complex body, consider these guidelines:

1. **Prefer GET when the query fits in the URL.** QUERY is not an excuse to abandon simple reads.
2. **Use QUERY when the body carries the query and the operation is read-only.** Do not use it for creates, updates, or deletes.
3. **Require and validate `Content-Type`.** Treat missing or mismatched types as client errors.
4. **Advertise supported formats with `Accept-Query`.** Help clients discover capabilities before sending large payloads.
5. **Return `Location` for expensive queries.** Give clients and caches a GET-friendly URI for repeat access.
6. **Document retry behavior.** Idempotency makes QUERY safe to retry, but only if the server honors the semantics.

## Bottom line

HTTP finally has a first-class method for safe requests with a body. QUERY does not replace GET or POST — it fills the space between them.

For API developers, that means fewer workarounds (`POST` endpoints that are secretly reads), clearer semantics for clients and caches, and a standard way to express complex queries without stuffing them into URLs.

For related reading on HTTP fundamentals in this blog, see [HTTP Status Codes Every API Developer Should Know](/rest-api/http/status-codes/2025/10/14/http-status-codes-every-api-developer-should-know.html), [HTTP Caching for REST APIs](/rest-api/http/caching/2026/02/07/http-caching-for-rest-apis.html), and [The Beauty and Simplicity of HTTPie](/rest-api/testing/cli/2026/06/24/the-beauty-and-simplicity-of-httpie.html).
