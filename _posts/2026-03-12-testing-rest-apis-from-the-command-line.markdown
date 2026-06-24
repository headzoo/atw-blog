---
layout: post
title: "Testing REST APIs from the Command Line"
date: 2026-03-12 19:25:00 +0000
categories: rest-api testing cli
image: /assets/images/testing-rest-apis-from-the-command-line.png
image_alt: "A terminal window sending REST API requests with curl and showing JSON response checks"
---

You do not need a full API client to test every endpoint. The command line is fast, scriptable, and available almost everywhere. For quick checks, CI smoke tests, and debugging from a server, tools like `curl`, `jq`, and HTTPie are hard to beat.

This guide focuses on practical REST API testing from a terminal.

## Start with curl

The simplest request is a `GET`:

```bash
curl https://api.example.com/health
```

Add `-i` when you want response headers:

```bash
curl -i https://api.example.com/health
```

For JSON APIs, include an `Accept` header:

```bash
curl -i \
  -H "Accept: application/json" \
  https://api.example.com/users/123
```

Headers often explain more than the body: content type, cache policy, rate limits, request ids, and auth challenges all live there.

## Send JSON bodies

For `POST`, `PUT`, and `PATCH`, send JSON with `-d` and a content type:

```bash
curl -i \
  -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"email":"avery@example.com","name":"Avery"}' \
  https://api.example.com/users
```

When the body is larger, put it in a file:

```bash
curl -i \
  -X POST \
  -H "Content-Type: application/json" \
  -d @create-user.json \
  https://api.example.com/users
```

Files make requests easier to review and reuse.

## Add authentication

Bearer tokens:

```bash
curl -i \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  https://api.example.com/users/me
```

API keys:

```bash
curl -i \
  -H "X-API-Key: $API_KEY" \
  https://api.example.com/reports
```

Use environment variables for local testing, but avoid printing secrets in shared logs or shell history.

## Format JSON with jq

`jq` makes JSON responses readable and testable.

Pretty-print:

```bash
curl -s https://api.example.com/users/123 | jq .
```

Extract a field:

```bash
curl -s https://api.example.com/users/123 | jq -r '.email'
```

Check a value:

```bash
curl -s https://api.example.com/users/123 | jq -e '.id == "usr_123"'
```

The `-e` flag makes `jq` exit with success or failure, which is useful in scripts and CI.

## Capture status codes

Sometimes you need the HTTP status code separately from the body:

```bash
curl -s -o response.json -w "%{http_code}\n" \
  https://api.example.com/users/123
```

That writes the body to `response.json` and prints the status code.

A shell test can fail on unexpected status:

```bash
status=$(curl -s -o response.json -w "%{http_code}" https://api.example.com/health)
test "$status" = "200"
```

Keep scripts small and obvious. If a test grows complex, it may belong in a real test framework.

## Use HTTPie for readable commands

HTTPie provides a friendlier syntax for many manual tests:

```bash
http GET https://api.example.com/users/123 \
  Authorization:"Bearer $ACCESS_TOKEN"
```

Posting JSON:

```bash
http POST https://api.example.com/users \
  email=avery@example.com \
  name=Avery
```

HTTPie is especially nice for exploratory testing. `curl` is still more universal for scripts because it is installed in more environments.

## Test error cases

Do not only test the happy path. From the command line, it is easy to send bad input:

```bash
curl -i \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"email":"not-an-email"}' \
  https://api.example.com/users
```

Check that the API returns:

- A useful `4xx` status
- A stable error code
- Field-level validation details
- No stack traces or internal secrets

These checks catch many API quality issues early.

## Build a small smoke test

A simple smoke test might verify health, auth, and one read endpoint:

```bash
#!/usr/bin/env bash
set -euo pipefail

base_url="${BASE_URL:-https://api.example.com}"

status=$(curl -s -o /tmp/health.json -w "%{http_code}" "$base_url/health")
test "$status" = "200"

curl -s \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$base_url/users/me" | jq -e '.id and .email' > /dev/null
```

This is not a full test suite, but it is a useful deployment check.

## Closing thoughts

Command-line API testing is fast because it removes ceremony. You can send a request, inspect headers, pipe JSON into checks, and save the command in documentation or CI.

Use `curl` for portability, `jq` for JSON assertions, and HTTPie when readability matters. Together, they cover a surprising amount of day-to-day REST API testing.
