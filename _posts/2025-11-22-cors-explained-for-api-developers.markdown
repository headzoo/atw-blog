---
layout: post
title: "CORS Explained for API Developers"
date: 2025-11-22 16:05:00 +0000
categories: rest-api http cors
image: /assets/images/cors-explained-for-api-developers.png
image_alt: "A browser sending a cross-origin request to an API with allowed origin and preflight response headers"
---

CORS errors are frustrating because the API can work perfectly in curl while failing in the browser. That is the clue: CORS is a browser security feature, not a general HTTP feature.

If you build APIs that browsers call directly, you need to understand what the browser is asking for and which headers your server must return.

## What CORS protects

Browsers enforce the same-origin policy. A script loaded from one origin cannot freely read responses from another origin.

An origin is the combination of:

- Scheme: `https`
- Host: `app.example.com`
- Port: `443`

These are different origins:

```text
https://app.example.com
https://api.example.com
http://app.example.com
https://app.example.com:8443
```

CORS, short for Cross-Origin Resource Sharing, is the mechanism that lets a server opt in to being called from another origin.

## Simple requests

Some browser requests are considered simple. For example:

```http
GET /users/me
Origin: https://app.example.com
```

If the API allows that origin, it responds with:

```http
Access-Control-Allow-Origin: https://app.example.com
```

The browser checks that header before making the response available to JavaScript. If the header is missing or does not match, the browser blocks the response even if the server returned `200 OK`.

## Preflight requests

Many API calls trigger a preflight request. The browser sends an `OPTIONS` request first to ask whether the real request is allowed.

Example:

```http
OPTIONS /users/me
Origin: https://app.example.com
Access-Control-Request-Method: PATCH
Access-Control-Request-Headers: authorization, content-type
```

The API should respond:

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PATCH, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 600
```

If the preflight succeeds, the browser sends the real `PATCH` request.

## Common CORS headers

The headers you will use most often are:

| Header | Purpose |
|--------|---------|
| `Access-Control-Allow-Origin` | Which origin may read the response |
| `Access-Control-Allow-Methods` | Which methods are allowed for preflighted requests |
| `Access-Control-Allow-Headers` | Which request headers are allowed |
| `Access-Control-Allow-Credentials` | Whether cookies or HTTP auth may be included |
| `Access-Control-Expose-Headers` | Which response headers JavaScript may read |
| `Access-Control-Max-Age` | How long the browser may cache preflight results |

Most CORS bugs come from a missing allowed header, an origin mismatch, or credentials being configured incorrectly.

## Be careful with credentials

Credentialed requests include cookies, client certificates, or HTTP authentication. In `fetch`, that usually means:

```javascript
fetch("https://api.example.com/users/me", {
  credentials: "include"
});
```

For this to work, the server must return:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

You cannot combine credentialed requests with:

```http
Access-Control-Allow-Origin: *
```

The server must echo or select a specific allowed origin.

## Do not use CORS as authentication

CORS limits which browser origins can read responses. It does not prove who the user is, and it does not stop non-browser clients from calling your API.

Your API still needs real authentication and authorization:

- Session cookies
- Bearer tokens
- API keys
- OAuth 2.0 access tokens

CORS is a browser access control layer. It is not a security model for your entire API.

## Configure allowed origins explicitly

For production, prefer an allowlist:

```text
https://app.example.com
https://admin.example.com
```

Avoid reflecting any incoming `Origin` header without checking it. That turns CORS into "allow everyone" while making the configuration look more controlled than it is.

For local development, add explicit local origins:

```text
http://localhost:3000
http://localhost:5173
```

Keep development origins out of production if they are not needed there.

## Debugging checklist

When a browser reports a CORS error, check:

- Is the request coming from the origin you expect?
- Does the response include `Access-Control-Allow-Origin`?
- For preflight, does the `OPTIONS` route return success?
- Are the requested method and headers allowed?
- Are credentials enabled on both client and server if cookies are used?
- Is the API redirecting the preflight request?
- Is a proxy or CDN stripping CORS headers?

Use the browser network panel. The console message is helpful, but the request and response headers tell the real story.

## Closing thoughts

CORS is easier once you separate the browser rule from the API behavior. The server decides which origins, methods, and headers are allowed. The browser enforces that decision before JavaScript can read the response.

Start with a small allowlist, handle `OPTIONS` correctly, and test both simple and credentialed requests. That covers most real-world CORS issues without weakening the API.
