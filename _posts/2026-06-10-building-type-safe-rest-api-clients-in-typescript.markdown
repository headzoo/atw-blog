---
layout: post
title: "Building Type-Safe REST API Clients in TypeScript"
date: 2026-06-10 15:35:00 +0000
categories: rest-api typescript clients
image: /assets/images/building-type-safe-rest-api-clients-in-typescript.png
image_alt: "TypeScript interfaces connected to REST API request and response objects in a client module"
---

TypeScript can make REST API clients easier to use and safer to refactor, but only if the types describe the real API contract. A typed wrapper around `fetch` should prevent common mistakes without hiding how HTTP works.

The goal is not to build a giant SDK on day one. Start with small, typed functions for the endpoints your application uses most.

## Define response types at the boundary

Begin with the shapes your client expects:

```typescript
type User = {
  id: string;
  email: string;
  displayName: string;
  createdAt: string;
};

type ApiError = {
  error: {
    code: string;
    message: string;
    details?: Array<{
      field?: string;
      message: string;
    }>;
  };
};
```

These types are a contract for your application code. Keep them close to the client module or generate them from OpenAPI if your API publishes a reliable spec.

## Wrap fetch once

A small request helper keeps headers, base URLs, JSON parsing, and error handling consistent.

```typescript
type RequestOptions = {
  method?: string;
  path: string;
  token?: string;
  body?: unknown;
};

async function request<T>(options: RequestOptions): Promise<T> {
  const response = await fetch(`${API_BASE_URL}${options.path}`, {
    method: options.method ?? "GET",
    headers: {
      "Accept": "application/json",
      "Content-Type": "application/json",
      ...(options.token ? { "Authorization": `Bearer ${options.token}` } : {})
    },
    body: options.body ? JSON.stringify(options.body) : undefined
  });

  if (!response.ok) {
    throw await response.json();
  }

  return response.json() as Promise<T>;
}
```

The generic `T` gives callers a typed result. The helper still needs runtime care because TypeScript does not validate JSON automatically.

## Add endpoint functions

Endpoint functions give the rest of your app a stable API.

```typescript
type CreateUserInput = {
  email: string;
  displayName: string;
};

export function getUser(id: string, token: string) {
  return request<User>({
    path: `/users/${encodeURIComponent(id)}`,
    token
  });
}

export function createUser(input: CreateUserInput, token: string) {
  return request<User>({
    method: "POST",
    path: "/users",
    token,
    body: input
  });
}
```

Now application code calls `getUser()` instead of constructing URLs and headers everywhere.

## Model pagination explicitly

If the API wraps list responses, type that wrapper once:

```typescript
type Page<T> = {
  data: T[];
  pagination: {
    limit: number;
    nextCursor?: string;
    hasMore: boolean;
  };
};

type ListUsersParams = {
  limit?: number;
  cursor?: string;
};
```

Build query strings from typed params:

```typescript
function listUsers(params: ListUsersParams, token: string) {
  const search = new URLSearchParams();

  if (params.limit) search.set("limit", String(params.limit));
  if (params.cursor) search.set("cursor", params.cursor);

  return request<Page<User>>({
    path: `/users?${search.toString()}`,
    token
  });
}
```

This prevents misspelled parameters from spreading through the codebase.

## Treat errors as part of the contract

Do not leave errors as `any`. Give them a shape:

```typescript
class ApiClientError extends Error {
  constructor(
    message: string,
    public status: number,
    public code?: string,
    public details?: ApiError["error"]["details"]
  ) {
    super(message);
  }
}
```

Then throw a typed error from the request helper. UI and service code can handle `validation_failed`, `token_expired`, or `rate_limit_exceeded` without parsing unknown objects in every caller.

## Validate runtime data when it matters

Type assertions do not prove the server returned the right shape:

```typescript
return response.json() as Promise<User>;
```

That line tells TypeScript what you expect. It does not validate the JSON.

For critical boundaries, add runtime validation with your project's existing validation library or generated client tooling. If the project does not already use a validator, weigh the dependency carefully instead of adding one casually.

## Generate types when the spec is trustworthy

If your API has a maintained OpenAPI spec, generated types can reduce manual drift.

Generation works best when:

- The spec is part of CI
- Examples match real responses
- Breaking changes are reviewed
- Generated files are easy to update
- Custom client code stays thin

Generated clients are not magic. They are only as accurate as the contract they come from.

## Closing thoughts

A type-safe REST client should make correct calls easy and incorrect calls hard. Start with typed endpoint functions, one request helper, explicit pagination types, and a real error model.

As the API grows, consider OpenAPI-generated types or runtime validation for the highest-risk boundaries. Keep the wrapper small enough that developers still understand the HTTP request being made.
