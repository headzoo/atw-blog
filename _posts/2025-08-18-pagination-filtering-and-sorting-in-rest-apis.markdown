---
layout: post
title: "Pagination, Filtering, and Sorting in REST APIs"
date: 2025-08-18 09:15:00 +0000
categories: rest-api design pagination
image: /assets/images/pagination-filtering-and-sorting-in-rest-apis.png
image_alt: "A REST API collection response with pagination controls, filter chips, and a descending sort indicator"
---

List endpoints look simple until they return too much data. A `GET /orders` endpoint that works in development can become slow, expensive, and hard to use once the table has millions of rows.

Pagination, filtering, and sorting turn list endpoints into predictable tools. They help servers protect resources and help clients ask for exactly the slice of data they need.

## Always paginate collections

Unbounded arrays are a common REST API mistake. Even if the list is small today, the endpoint contract should make room for growth.

A basic page-based request might look like this:

```http
GET /orders?page=2&limit=25
```

And the response:

```json
{
  "data": [
    { "id": "ord_101", "status": "paid" }
  ],
  "pagination": {
    "page": 2,
    "limit": 25,
    "total": 413,
    "total_pages": 17
  }
}
```

Page-based pagination is easy to understand and works well for admin screens, reports, and moderately sized datasets.

## Know when to use cursors

Offset or page pagination can become unreliable when records are added or removed while the client is paging. It can also be expensive for deep pages in large tables.

Cursor pagination uses a pointer instead:

```http
GET /orders?limit=25&cursor=eyJjcmVhdGVkX2F0IjoiMjAyNS0wOC0xOFQwOToxNTowMFoifQ
```

Example response:

```json
{
  "data": [
    { "id": "ord_101", "created_at": "2025-08-18T09:15:00Z" }
  ],
  "pagination": {
    "limit": 25,
    "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNS0wOC0xOFQwOTowMDowMFoifQ"
  }
}
```

Cursors are a good fit for activity feeds, event streams, logs, and large tables where clients mostly move forward through results.

## Set defaults and maximums

Document default and maximum page sizes:

- Default `limit`: `25`
- Maximum `limit`: `100`
- Default sort: `created_at desc`

Do not let clients request `limit=100000`. A maximum protects the database, response size, and downstream clients. If a client needs bulk export, create a separate export workflow instead of stretching a list endpoint past its purpose.

## Design filters around real use cases

Filters should map to fields that users actually search by and that the backend can support efficiently.

Good examples:

```http
GET /orders?status=paid
GET /orders?customer_id=cus_123
GET /orders?created_after=2025-08-01T00:00:00Z
GET /orders?status=paid&created_after=2025-08-01T00:00:00Z
```

Avoid inventing a complex query language unless your API truly needs one. Most products can start with explicit query parameters and add advanced search later.

## Make sorting explicit

A compact sorting convention is to use a leading minus sign for descending order:

```http
GET /orders?sort=-created_at
GET /orders?sort=status,-created_at
```

Another readable option is separate parameters:

```http
GET /orders?sort_by=created_at&sort_direction=desc
```

Either style can work. The important part is to document allowed sort fields and reject unsupported ones with a clear error.

## Keep ordering stable

Stable ordering prevents duplicates and missing records between pages. If many rows can share the same sort value, add a tie-breaker such as `id`.

For example, sort by:

```sql
created_at DESC, id DESC
```

Then encode both values into a cursor. This gives the server a precise place to resume from.

## Return useful metadata

Page-based APIs often return totals:

```json
{
  "pagination": {
    "page": 1,
    "limit": 25,
    "total": 413,
    "total_pages": 17
  }
}
```

Cursor-based APIs usually avoid totals because counting can be expensive. Instead, return navigation hints:

```json
{
  "pagination": {
    "limit": 25,
    "next_cursor": "abc123",
    "has_more": true
  }
}
```

Choose metadata that matches the pagination model and can be produced efficiently.

## Validate query parameters

Bad query parameters should fail loudly. If `limit=banana` or `sort=password_hash` is invalid, return a `400` or `422` with a useful error.

Example:

```json
{
  "error": {
    "code": "invalid_query_parameter",
    "message": "The sort field is not supported.",
    "details": [
      { "field": "sort", "message": "Allowed values: created_at, status, total." }
    ]
  }
}
```

Silent fallback makes client bugs harder to find.

## A practical checklist

For every list endpoint, define:

- [ ] Default page size
- [ ] Maximum page size
- [ ] Pagination style: page, offset, or cursor
- [ ] Default sort order
- [ ] Allowed sort fields
- [ ] Supported filters
- [ ] Response metadata
- [ ] Error behavior for invalid query parameters

## Closing thoughts

List endpoints deserve design attention. Pagination protects the system, filtering keeps responses relevant, and sorting makes results predictable.

Start with simple parameters, document defaults, and keep ordering stable. If the dataset grows or changes frequently, move to cursor pagination before clients start depending on brittle offset behavior.
