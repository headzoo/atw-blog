---
layout: post
title: "Quick Summary: Capture Auth Tokens with HarborClient Post-Request Scripts"
date: 2026-07-06 15:45:00 +0000
categories: rest-api authentication tutorial
image: /assets/images/capture-auth-tokens-post-request-scripts-featured.png
image_alt: "HarborClient post-request script workflow for capturing auth tokens"
---

Many APIs require a login or grant request before you can call anything else. HarborClient post-request scripts let you capture a token from that response once, store it in a collection variable, and reuse it automatically on every request in the collection.

This is a quick summary of the full walkthrough: [Capture Auth Tokens with Post-Request Scripts in HarborClient](https://harborclient-blog.com/2026/07/06/capture-auth-tokens-with-post-request-scripts-in-harborclient/).

## The workflow

- Send `POST {% raw %}{{baseUrl}}{% endraw %}/auth/apiGrant` to obtain an `idToken`.
- Add a post-request script on that request to save `idToken` to a collection variable.
- Set collection authorization to **Bearer Token** with value `{% raw %}{{idToken}}{% endraw %}`.
- Send any other request in the collection with auth set to **Inherit from collection**. HarborClient attaches the token for you.

## The script

On the token request, open the **Scripts** tab and add a post-request script:

{% raw %}
```js
const response = hc.response.json();

if (response.idToken) {
  hc.collection.variables.set("idToken", response.idToken);
}
```
{% endraw %}

The script parses the JSON response, writes the token to a collection-scoped variable, and skips the update if the login failed.

## Why this helps

- **No manual copy-paste**: the JWT is captured automatically after the grant request succeeds.
- **Collection-wide auth**: every request that inherits collection auth gets the `Authorization: Bearer` header.
- **Easy refresh**: re-send the token request when you hit a `401` or the token expires.
- **Clean scope**: a request-level script on the login endpoint is usually the simplest approach.

## Read the full tutorial

For step-by-step setup with screenshots, collection variable tables, troubleshooting tips, and ideas for storing expiry or auto-refreshing tokens, read the [full tutorial](https://harborclient-blog.com/2026/07/06/capture-auth-tokens-with-post-request-scripts-in-harborclient/).
