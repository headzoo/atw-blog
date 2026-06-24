---
layout: post
title: "HarborClient Adds Plugin Support"
date: 2026-06-24 18:00:00 +0000
categories: rest-api testing tools
image: /assets/images/harborclient-adds-plugin-support.png
image_alt: "A harbor at sunset with glowing modular cubes connected to a central lighthouse, representing plugin integration"
---

[HarborClient 1.8.0](https://harborclient-blog.com/2026/06/24/harborclient-adds-plugin-support/) now supports installable plugins — extensions that add UI, themes, and HTTP hooks without waiting for a new app release. The full announcement is on the [HarborClient blog](https://harborclient-blog.com/2026/06/24/harborclient-adds-plugin-support/); here are the highlights.

> Starting in HarborClient 1.8.0, you can go further: installable plugins that extend the app without waiting for a new release.

That shift matters for teams who want custom tooling — audit tabs, logging middleware, branded themes — baked into their API client rather than maintained as separate scripts.

> Plugins are packaged as `.hcp` files — HarborClient plugin packages that are really just ZIP archives with a friendly extension.

You can install from a downloaded `.hcp` or `.zip`, clone from a public Git repository, or point HarborClient at an unpacked source folder while you develop.

> Plugins use a permission-gated model similar to VS Code extensions.

HarborClient shows what each plugin is asking for — UI contributions, scoped storage, filesystem access, HTTP hooks — before it activates. You can enable or disable any plugin later without uninstalling it.

Together, these pieces mean HarborClient can grow through community extensions: sidebar panels, request tabs, custom themes, and main-process HTTP hooks for signing or logging traffic. For manifest reference, bundling with esbuild, and example plugins, read the [full announcement](https://harborclient-blog.com/2026/06/24/harborclient-adds-plugin-support/).
