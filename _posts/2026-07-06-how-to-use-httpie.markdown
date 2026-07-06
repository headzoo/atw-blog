---
layout: post
title: "How to Use HTTPie for REST API Requests"
date: 2026-07-06 14:30:00 +0000
categories: rest-api testing cli
image: /assets/images/how-to-use-httpie.png
image_alt: "A terminal running an HTTPie request against an echo API with GET, POST, PUT, and DELETE method labels"
---

HTTPie turns HTTP requests into readable one-liners. Instead of juggling flags, you write the method, URL, headers, query parameters, and JSON fields in order — and the response comes back formatted and colorized in your terminal.

This guide is a hands-on walkthrough of the four core REST methods: `GET`, `POST`, `PUT`, and `DELETE`. Every example uses [https://echo.harborclient.com](https://echo.harborclient.com), a public echo service that reflects your request back as JSON. You can run each command as-is and inspect exactly what was sent.

If you want a broader look at why HTTPie feels different from `curl`, see [The Beauty and Simplicity of HTTPie](/rest-api/testing/cli/2026/06/24/the-beauty-and-simplicity-of-httpie.html).

## Install HTTPie

Install through your package manager:

```bash
brew install httpie      # macOS
apt install httpie       # Debian/Ubuntu
pip install httpie       # Python environments
```

Verify the install:

```bash
http --version
```

The command name is `http`. Some distributions also ship `https` as a shortcut that defaults to TLS.

## How HTTPie commands are structured

Every request follows the same shape:

```bash
http [OPTIONS] METHOD URL [REQUEST_ITEMS...]
```

**Request items** are the tokens after the URL. The separator tells HTTPie what each item means:

| Separator | Meaning | Example |
|-----------|---------|---------|
| `Header:value` | HTTP header | `Accept:application/json` |
| `field=value` | JSON or form field | `email=avery@example.com` |
| `param==value` | Query string parameter | `page==2 limit==10` |
| `field:=value` | Raw JSON value | `active:=true` |

HTTPie defaults to `GET` when you omit the method and send no body fields. When you include field data without a method, it defaults to `POST`.

## GET — read a resource

`GET` retrieves data. It should not change server state.

A basic request to the echo service:

```bash
http GET https://echo.harborclient.com/
```

The response is JSON showing what the server received — query args, headers, and the request URL. Look at the `headers` object to confirm your `User-Agent` and other metadata arrived intact.

### Add query parameters

Append filters or pagination with `==`:

```bash
http GET https://echo.harborclient.com/users \
  page==2 \
  limit==10 \
  sort==created_at
```

HTTPie builds `?page=2&limit=10&sort=created_at` for you. In the echo response, check the `args` field:

```json
"args": {
  "page": "2",
  "limit": "10",
  "sort": "created_at"
}
```

### Send headers on a GET

Headers use a single colon:

```bash
http GET https://echo.harborclient.com/users/me \
  Authorization:"Bearer $ACCESS_TOKEN" \
  Accept:application/json
```

The echo response lists every header under `headers`, so you can confirm auth and content negotiation before pointing the same command at a real API.

### Read only the response body

When piping output to another tool, skip headers and status:

```bash
http --print=b GET https://echo.harborclient.com/users/42
```

## POST — create a resource

`POST` sends data to the server, usually to create something new.

Field arguments after the URL become a JSON object automatically — no `Content-Type` flag required:

```bash
http POST https://echo.harborclient.com/users \
  email=avery@example.com \
  name=Avery
```

In the echo response, look at `json`:

```json
"json": {
  "email": "avery@example.com",
  "name": "Avery"
}
```

HTTPie set `Content-Type: application/json` and serialized the body for you.

### Nested JSON fields

Use bracket notation for nested objects:

```bash
http POST https://echo.harborclient.com/users \
  name=Avery \
  address[city]=Portland \
  address[state]=OR
```

The echo response shows the nested structure under `json`.

### Raw JSON values

When a value is not a string — booleans, numbers, arrays — use `:=`:

```bash
http POST https://echo.harborclient.com/users \
  name=Avery \
  active:=true \
  tags:='["admin","beta"]'
```

Without `:=`, HTTPie treats every value as a string. The echo response confirms the types landed correctly in `json`.

### POST with custom headers

```bash
http POST https://echo.harborclient.com/users \
  X-Request-Id:demo-001 \
  email=avery@example.com \
  name=Avery
```

Check both `headers` and `json` in the response to verify the full request.

## PUT — replace a resource

`PUT` sends a complete representation of a resource. In REST APIs, it typically replaces the existing record at the given URL.

Update user `42` with a full payload:

```bash
http PUT https://echo.harborclient.com/users/42 \
  email=avery@example.com \
  name=Avery \
  role=editor \
  active:=true
```

The echo response shows the method reflected in the request metadata and the full JSON body under `json`. On a real API, you would send every field the resource expects — partial updates usually belong on `PATCH` instead.

### PUT a body from a file

For larger payloads, read JSON from a file:

```bash
http PUT https://echo.harborclient.com/users/42 < update-user.json
```

Files keep requests reviewable and easy to reuse in scripts.

## DELETE — remove a resource

`DELETE` removes a resource at the given URL.

```bash
http DELETE https://echo.harborclient.com/users/42
```

The echo service still returns JSON describing the request. On production APIs, `DELETE` often responds with `204 No Content` and an empty body — that is normal. Use `--print=Hh` to inspect status and headers when the body is empty:

```bash
http --print=Hh DELETE https://echo.harborclient.com/users/42
```

### DELETE with auth

Protected endpoints expect credentials on every method, including `DELETE`:

```bash
http DELETE https://echo.harborclient.com/users/42 \
  Authorization:"Bearer $ACCESS_TOKEN"
```

The echo response confirms the header was sent even though there is no request body.

## Quick reference

| Goal | Example |
|------|---------|
| Simple GET | `http GET https://echo.harborclient.com/users/42` |
| Query params | `http GET https://echo.harborclient.com/users page==2` |
| POST JSON | `http POST https://echo.harborclient.com/users name=Avery` |
| PUT JSON | `http PUT https://echo.harborclient.com/users/42 name=Avery role=editor` |
| DELETE | `http DELETE https://echo.harborclient.com/users/42` |
| Custom header | `Authorization:"Bearer $TOKEN"` |
| Body only in output | `--print=b` |
| Fail on 4xx/5xx | `--check-status` |

## Tips for day-to-day use

**Preview before sending.** Use `--offline` to print the request without hitting the network:

```bash
http --offline POST https://echo.harborclient.com/users name=Avery role=editor
```

**Reuse login state.** Named sessions persist cookies between commands — useful for APIs with session auth:

```bash
http --session=logged-in POST https://api.example.com/login \
  email=avery@example.com password="$PASSWORD"

http --session=logged-in GET https://api.example.com/users/me
```

**Script-friendly exits.** Combine `--check-status` with `--print=b` in shell scripts so HTTPie exits non-zero on error responses while printing only the body on success.

**When to reach for curl instead.** HTTPie shines for interactive exploration. In minimal CI images or when you need maximum portability, `curl` with `jq` is still the safer default. See [Testing REST APIs from the Command Line](/rest-api/testing/cli/2026/03/12/testing-rest-apis-from-the-command-line.html) for that workflow.

## Try it yourself

Pick one endpoint you work with regularly and rewrite your usual `curl` one-liner as `http`. Start with a `GET`, then try a `POST` with a JSON body. The echo service at `https://echo.harborclient.com` is a safe sandbox when you want to experiment without touching production data.

For the full CLI reference, see [httpie.io/docs/cli](https://httpie.io/docs/cli).
