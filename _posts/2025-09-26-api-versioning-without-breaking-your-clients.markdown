---
layout: post
title: "API Versioning Without Breaking Your Clients"
date: 2025-09-26 17:45:00 +0000
categories: rest-api versioning design
image: /assets/images/api-versioning-without-breaking-your-clients.png
image_alt: "Two REST API versions running side by side with clients gradually migrating from v1 to v2"
---

APIs change because products change. New fields appear, old assumptions stop holding, and better models replace early decisions. The hard part is evolving the API without breaking every client that already depends on it.

Versioning is not just a URL pattern. It is a promise about compatibility, migration, and communication.

## First, know what counts as breaking

Not every change needs a new version.

Usually safe:

- Adding a new optional response field
- Adding a new endpoint
- Adding a new optional request field
- Adding a new enum value when clients are built to ignore unknown values
- Improving performance without changing behavior

Usually breaking:

- Removing or renaming a field
- Changing a field type
- Making an optional field required
- Changing pagination behavior
- Changing authentication requirements
- Reusing an existing status code for a different meaning
- Changing an endpoint's core semantics

When in doubt, assume clients are less flexible than you hope. Many integrations deserialize into strict models and fail on surprises.

## Prefer additive changes

The easiest versioning strategy is to avoid breaking changes when you can.

Instead of renaming a field:

```json
{
  "id": "usr_123",
  "name": "Avery",
  "display_name": "Avery"
}
```

Add the new field, keep the old one, and document which one new clients should use. You can deprecate the old field later after measuring usage.

Instead of changing behavior behind `GET /reports`, add a new endpoint:

```http
GET /reports
GET /reports/summary
```

Additive changes are not always perfect, but they buy time and preserve trust.

## Pick a versioning style

Common REST API approaches include URL prefixes, headers, and media types.

### URL versioning

```http
GET /v1/users/123
GET /v2/users/123
```

URL versioning is obvious, easy to route, and simple for documentation. It is also the most familiar option for many public APIs.

### Header versioning

```http
GET /users/123
API-Version: 2025-09-26
```

Header versioning keeps URLs clean and can support date-based versions. The trade-off is discoverability. A developer cannot see the version by glancing at a link.

### Media type versioning

```http
Accept: application/vnd.example.v2+json
```

This is precise but more complex. It works best for APIs with strong HTTP content negotiation practices.

For most teams, `/v1` and `/v2` are boring in the best way.

## Version the contract, not every endpoint

Avoid mixing versions endpoint by endpoint unless you have a strong reason. A client using `/v1` should not have to remember that users are v1, invoices are v2, and reports are halfway between.

That does not mean every endpoint must be rewritten for a new major version. It means the public contract should feel coherent. Internally, v2 routes can reuse v1 handlers where behavior is unchanged.

## Use deprecation and sunset headers

When you need to retire an old version, tell clients in responses:

```http
Deprecation: true
Sunset: Wed, 31 Dec 2025 23:59:59 GMT
Link: <https://docs.example.com/migrate/v2>; rel="deprecation"
```

These headers are not a replacement for email, changelogs, dashboard notices, or direct partner outreach. They are an extra signal that automated clients and observability tools can see.

## Give clients a migration guide

A good migration guide is concrete:

- What changed
- Why it changed
- Which endpoints are affected
- Old request and response examples
- New request and response examples
- Deadline for migration
- Contact path for help

Do not make clients diff two reference docs and guess what matters.

## Measure old-version usage

Before removing a version, know who still uses it.

Track:

- Requests by API version
- Requests by endpoint
- Authenticated account or application id
- Error rates after migration
- Deprecated fields still present in responses

Usage data lets you contact the right customers and avoid deleting something that a critical integration still depends on.

## Test versions side by side

Automated tests should cover old and new behavior while both versions are live.

At minimum, test:

- v1 still returns its documented response
- v2 returns the new response
- shared business logic behaves the same where expected
- auth and rate limits apply consistently
- old clients do not accidentally route to new handlers

Versioning often fails at the routing and serialization layer, not in the domain logic.

## A practical versioning policy

Write down a short policy for your team:

- Which changes are breaking
- Which versioning style the API uses
- How long old versions are supported
- How deprecations are announced
- How usage is measured
- Who approves breaking changes

The policy does not need to be long. It just needs to prevent every team from inventing a different rule under deadline pressure.

## Closing thoughts

Versioning is a social contract backed by technical routing. The goal is not to preserve old behavior forever. The goal is to make change predictable.

Prefer additive changes, version only when the contract truly breaks, and give clients clear migration paths. APIs that evolve carefully earn more trust than APIs that never change or change without warning.
