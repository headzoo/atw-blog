---
layout: post
title: "HTTP Caching for REST APIs"
date: 2026-02-07 10:40:00 +0000
categories: rest-api http caching
image: /assets/images/http-caching-for-rest-apis.png
image_alt: "A REST API response flowing through browser, CDN, and origin cache layers with ETag and Cache-Control headers"
---

Caching is one of the most effective ways to make REST APIs faster, but it is also easy to get wrong. A stale public profile is annoying. A stale billing response is a serious bug.

HTTP already gives APIs a useful caching vocabulary. The trick is choosing the right headers for each kind of response.

## Start with Cache-Control

`Cache-Control` tells browsers, proxies, and CDNs how a response may be cached.

Example:

```http
Cache-Control: public, max-age=300
```

This means the response may be stored by shared caches and reused for 300 seconds.

Common directives:

| Directive | Meaning |
|-----------|---------|
| `public` | Shared caches may store the response |
| `private` | Only the end user's browser may store it |
| `no-store` | Do not store the response at all |
| `no-cache` | Store it, but revalidate before reuse |
| `max-age=60` | Fresh for 60 seconds |
| `s-maxage=300` | Fresh for shared caches for 300 seconds |

Use `no-store` for sensitive responses such as tokens, account secrets, or payment details.

## Cache public, stable reads

Good candidates for caching:

- Public documentation metadata
- Product catalogs
- Feature flag definitions that change slowly
- Public user profiles
- Static lookup tables

Example:

```http
GET /countries
Cache-Control: public, max-age=86400
```

If the data changes rarely, a long cache lifetime can remove a lot of repeated traffic.

## Keep personalized responses private

Responses tied to one user should not be stored by shared caches unless you are very sure the cache key includes the user identity.

Safer default:

```http
Cache-Control: private, max-age=60
```

For highly sensitive data:

```http
Cache-Control: no-store
```

Be especially careful with APIs behind CDNs. A missing `private` or `no-store` directive can accidentally cache user-specific data in a shared layer.

## Use ETags for validation

An `ETag` is a version identifier for a response representation.

First response:

```http
HTTP/1.1 200 OK
ETag: "users-123-v7"
Cache-Control: private, no-cache
```

Later request:

```http
GET /users/123
If-None-Match: "users-123-v7"
```

If nothing changed, the server can return:

```http
HTTP/1.1 304 Not Modified
```

The client reuses its cached body, and the API avoids sending the same JSON again.

## Last-Modified also works

`Last-Modified` and `If-Modified-Since` are timestamp-based validation headers.

```http
Last-Modified: Sat, 07 Feb 2026 10:40:00 GMT
```

They are easy to implement when resources already have an `updated_at` field. ETags are usually more precise because they can represent any change in the response, not only timestamp changes.

## Handle unsafe methods carefully

`GET` and `HEAD` are the usual cacheable methods. Mutating requests such as `POST`, `PATCH`, and `DELETE` should generally not be cached.

After a mutation, clients may need to invalidate or refetch cached reads:

```http
PATCH /users/123
GET /users/123
```

If you control both API and client, document which reads are affected by writes. If you use a CDN, make sure your purge or surrogate key strategy matches your resource model.

## Vary when responses depend on headers

The `Vary` header tells caches which request headers affect the response.

Example:

```http
Vary: Accept-Encoding, Accept-Language
```

If an API response differs by authorization or tenant headers, shared caching becomes more complicated. In many cases, `private` or `no-store` is safer than trying to build a perfect shared cache key for personalized data.

## Avoid caching errors blindly

Some error responses can be cached. For example, a public `404` for a missing static resource may be fine.

But be careful with:

- `401` and `403`
- temporary `500` errors
- rate-limit responses
- validation errors tied to one request

If you do cache errors, make the lifetime short and intentional.

## A practical caching checklist

For each `GET` endpoint, decide:

- [ ] Is the response public, private, or sensitive?
- [ ] Can shared caches store it?
- [ ] How long is it fresh?
- [ ] Should clients revalidate with `ETag` or `Last-Modified`?
- [ ] Does the response vary by headers?
- [ ] How are caches invalidated after writes?
- [ ] Are error responses cacheable?

## Closing thoughts

HTTP caching works best when each endpoint has a clear freshness policy. Do not rely on defaults, and do not treat all responses the same.

Cache public, stable data aggressively. Revalidate data that changes. Keep sensitive or personalized responses private or unstored. Those choices can improve performance without turning stale data into a production incident.
