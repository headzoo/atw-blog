---
layout: post
title: "Migrating from JavaScript to Typescript"
date: 2026-04-19 12:00:00 +0000
categories: javascript typescript migration
image: /assets/images/migrating-from-javascript-to-typescript.png
image_alt: "JavaScript code transitioning into TypeScript with type annotations and safer interfaces"
---

Most JavaScript codebases do not need a big-bang rewrite to adopt TypeScript. The migration that works best is incremental: rename a few files, tighten compiler settings over time, and let types catch bugs at the boundaries where they matter most.

This guide walks through a practical path from a running JavaScript project to a TypeScript codebase your team can maintain with confidence.

## Why migrate

TypeScript does not replace JavaScript — it adds a static type layer on top of it. That layer pays off when:

- **Interfaces change often** — types document contracts and break builds when callers drift
- **Teams grow** — autocomplete and inline docs reduce onboarding time
- **Refactors are risky** — the compiler finds renamed fields and missing arguments before runtime
- **API boundaries are messy** — request payloads, database rows, and config objects benefit from explicit shapes

The goal is not 100% type coverage on day one. The goal is fewer production bugs and safer refactors with minimal workflow disruption.

## Prepare the codebase

Before touching file extensions, make the project easy to compile:

1. **Confirm your build pipeline** — webpack, Vite, esbuild, and `tsc` all support TypeScript; pick one source of truth for transpilation
2. **Add a baseline `tsconfig.json`** — start permissive so existing code still passes
3. **Run your test suite** — migration should not change runtime behavior; tests are your safety net
4. **Fix obvious lint issues** — unused variables and implicit globals become harder to ignore once TypeScript is enabled

A starter config for a gradual migration often looks like this:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowJs": true,
    "checkJs": false,
    "strict": false,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src/**/*"]
}
```

`allowJs: true` lets `.js` and `.ts` files coexist while you migrate file by file.

## Enable TypeScript gradually

Avoid renaming every file at once. A proven order:

1. Add TypeScript and a `tsconfig.json`
2. Rename leaf modules first — utilities, helpers, and pure functions with few dependencies
3. Move up the dependency graph toward entry points and framework wiring
4. Rename `.js` to `.ts` (or `.tsx` for React) only when you are ready to fix type errors in that file

Each renamed file should compile cleanly before you move on. Small PRs keep review manageable and reduce merge conflicts.

## Type boundaries first

Not all code needs rich types immediately. Prioritize surfaces where bad data causes the most damage:

- **HTTP handlers** — request bodies, query params, and response payloads
- **Database access** — row shapes returned from queries or ORMs
- **Configuration** — environment variables and feature flags
- **Shared libraries** — functions exported to other packages or teams

Example: a REST handler that accepts JSON benefits from an explicit input type:

```typescript
type CreateUserInput = {
  email: string;
  name: string;
};

function parseCreateUser(body: unknown): CreateUserInput {
  if (
    typeof body !== "object" ||
    body === null ||
    typeof (body as CreateUserInput).email !== "string" ||
    typeof (body as CreateUserInput).name !== "string"
  ) {
    throw new Error("Invalid request body");
  }
  return body as CreateUserInput;
}
```

Once boundaries are typed, internal implementation details can stay loosely typed until you refactor them.

## Handle third-party packages

Most popular npm packages ship TypeScript definitions. Install `@types/*` packages when types are maintained separately:

```bash
pnpm add -D typescript @types/node
```

When a library has no types:

- Check for a DefinitelyTyped package (`@types/lodash`, etc.)
- Write a minimal `.d.ts` module declaration for the parts you use
- Wrap untyped libraries behind your own typed adapter so `@ts-ignore` does not spread through the codebase

`skipLibCheck: true` avoids type errors inside node_modules while you focus on your own code.

## Tighten compiler settings over time

Start permissive, then enable stricter flags as coverage improves:

| Setting | What it catches | When to enable |
|---------|-----------------|----------------|
| `noImplicitAny` | Functions and variables inferred as `any` | After most files are `.ts` |
| `strictNullChecks` | `null` and `undefined` misuse | When core models are typed |
| `strict` | Full strict mode bundle | When the team is comfortable fixing edge cases |
| `noUncheckedIndexedAccess` | Unsafe array/object index access | For data-heavy code paths |

Turn on one flag at a time, fix the resulting errors in a dedicated PR, and document the new standard for the team.

## Common pitfalls

- **Chasing 100% coverage too early** — `any` in a legacy module is acceptable during migration; isolate it and schedule cleanup
- **Over-typing before understanding runtime behavior** — types should reflect what the code actually does, not an idealized API
- **Mixing build tools** — let one tool own transpilation; avoid running Babel and `tsc` with conflicting settings
- **Ignoring test types** — add `@types/jest` or `@types/mocha` so test helpers are typed too
- **Big-bang PRs** — large migrations stall in review; incremental renames merge faster

## A practical migration checklist

- [ ] `tsconfig.json` with `allowJs: true`
- [ ] TypeScript added as a dev dependency
- [ ] Build and test scripts updated
- [ ] Boundary modules typed (API, config, persistence)
- [ ] Third-party types installed or stubbed
- [ ] Stricter compiler flags enabled incrementally
- [ ] Team agrees on conventions for `any`, assertions, and new code

## Closing thoughts

Migrating from JavaScript to TypeScript is a process, not a switch. Rename files in dependency order, type the edges where bugs are expensive, and tighten compiler settings as confidence grows.

Done well, the migration feels invisible to users — same runtime behavior, fewer surprises for developers. Start with one module this week, ship it, and expand from there.
