---
layout: post
title: "Introducing HarborClient 2.0"
date: 2026-07-05 20:00:00 +0000
categories: rest-api testing tools
image: /assets/images/introducing-harborclient-2-0.png
image_alt: "HarborClient 2.0 announcement graphic with a lighthouse logo and API feature panels"
---

[HarborClient 2.0](https://harborclient-blog.com/2026/07/05/introducing-harborclient-2-0/) marks a maturity milestone for the local-first HTTP client. The full announcement is on the [HarborClient blog](https://harborclient-blog.com/2026/07/05/introducing-harborclient-2-0/); here are the highlights.

> HarborClient 2.0 brings a mature plugin marketplace, installable themes, global search, request tags, cookie management, richer pre/post scripts, and a smarter AI assistant — still free, local-first, and account-free.

The headline features landed across the 1.9 series; 2.0.0 adds polish like a refreshed splash screen, MDN links on standard HTTP header names, and more reliable script change detection. Together, the release makes the workflow feel complete — extensible through plugins, searchable across your workspace, and organized with tags and cookies the way real API work demands.

> Open **Settings → Plugins → Marketplace** to discover extensions maintained by the HarborClient community.

The curated catalog includes tools for AWS SigV4 signing, cURL and HTTPie import, OpenAPI import, JWT decoding, load testing, and more — 17 entries and growing. Install from the marketplace, a downloaded `.hcp` package, a public Git repository, or a `harborclient://` deep link. Behind the scenes, `@harborclient/sdk` reached 1.0.0, giving plugin authors a stable surface for UI contributions, HTTP hooks, storage, and filesystem access.

> With 2.0, **theme plugins** extend the appearance picker with community color schemes — Catppuccin Latte, Nord, Gruvbox, Dracula, and Solarized among them.

**Global search** spans collections, requests, settings, and installed plugins in one keyboard-first field. **Request tags** let you organize and filter beyond folder hierarchy, and they travel with exports. **Cookie jar management** adds a full UI to view, edit, and clear cookies by domain, with autocomplete for headers, query params, and cookies in the request editor.

> Scripts run in a hardened SES sandbox in the main process — your API keys and collection data never leave your machine.

Pre/post scripts now support **script arrays**, reusable **snippets**, a configurable execution timeout, and AI-assisted inline editing. The AI sidebar uses your own API keys stored locally, with smarter chat titles and tools for inspecting collections and updating scripts. The collection runner can execute a **single request** from the runner modal, and **merge environment down** propagates variables from parent environments to children.

HarborClient 2.0 remains open source under the MIT license. There are no accounts, subscriptions, or usage limits — your collections stay on your machine or storage you control. Existing collections upgrade in place with no migration step. For the full feature list and download links, read the [full announcement](https://harborclient-blog.com/2026/07/05/introducing-harborclient-2-0/).
