---
name: hono-master
description: >
  Expert skill for building production-grade Hono web servers with Bun (primary runtime), TypeScript or JavaScript.
  Use this skill whenever the user wants to create, scaffold, extend, debug, or optimize a Hono server — including
  REST APIs, middleware stacks, auth systems, database-backed routes, OpenAPI/Zod validation, WebSockets,
  file uploads, rate limiting, testing, and deployment. Trigger on any mention of: Hono, hono server,
  hono API, hono route, hono middleware, hono auth, hono zod, hono drizzle, hono deployment, hono websocket,
  hono testing, hono upload, or "build an API with Bun". Also trigger when the user asks to build a
  backend/API and their stack includes Bun or Hono. Don't wait for the user to say "use hono-master" —
  if the task involves Hono in any way, this skill applies.
---

# hono-master

Expert guidance for building production-grade Hono web servers on Bun (also supports Node.js).

## Quick Reference

| Topic | Reference File |
|---|---|
| Project structure & scaffolding | `references/scaffolding.md` |
| Routing & middleware patterns | `references/routing.md` |
| Auth & JWT | `references/auth.md` |
| Database (Postgres + Drizzle / MongoDB) | `references/database.md` |
| OpenAPI + Zod validation | `references/openapi-zod.md` |
| WebSockets, SSE, file uploads, rate limiting | `references/advanced.md` |
| Testing (bun:test + app.request) | `references/testing.md` |
| Deployment (Docker, Railway, Fly.io, Render) | `references/deployment.md` |

**Always read the relevant reference file(s) before writing code.** For a new full project, read `scaffolding.md` first, then whatever else applies. For tasks spanning multiple topics, read all relevant files.

---

## Core Principles

1. **Bun-first** — scaffold with `bun create hono@latest`, run with `bun run`, test with `bun:test`. Prefer Bun APIs over Node shims.
2. **TypeScript by default** — strict mode, typed env (Zod), typed bindings and variables. JS only if the user explicitly asks.
3. **Zod as single source of truth** — validate at the boundary, derive types from schemas, never duplicate type definitions.
4. **Drizzle as the ORM** — `drizzle-orm` with `postgres-js` for Postgres, or Mongoose for Mongo. Use `createSchemaFactory` from `drizzle-orm/zod` (not the deprecated `drizzle-zod` package) to derive Zod schemas.
5. **`@hono/zod-openapi` + Scalar for documented APIs** — every production API should have auto-generated, interactive OpenAPI docs at `/docs`.
6. **Lean middleware stack** — Hono ships batteries: logger, cors, secureHeaders, compress, jwt, rateLimiter, timeout, requestId, bodyLimit. Only reach for third-party packages when the built-in can't do it.
7. **Don't make controllers** — inline handlers preserve type inference for path params. Use `createFactory`/`createHandlers` only if you genuinely need reusable grouped logic.
8. **Test from the start** — `app.request()` makes HTTP-level tests trivial. Mock the DB layer; test real routes.

---

## Hono Fundamentals (always in context)

```ts
// Bun entry point — src/index.ts
import { Hono } from 'hono'

const app = new Hono()
app.get('/', (c) => c.text('Hello Hono!'))

export default app   // Bun picks this up automatically
```

### Context helpers cheatsheet
| Method | Usage |
|---|---|
| `c.text(str, status?)` | Plain text response |
| `c.json(obj, status?)` | JSON response |
| `c.html(str)` | HTML response |
| `c.redirect(url, status?)` | Redirect (default 302) |
| `c.req.json()` | Parse JSON body |
| `c.req.parseBody()` | Parse multipart / form-data |
| `c.req.param('id')` | Route param |
| `c.req.query('q')` | Query string value |
| `c.req.queries('tag')` | Repeated query params → array |
| `c.req.header('X-Foo')` | Request header |
| `c.req.valid('json')` | Validated + typed body (zod-openapi / zValidator) |
| `c.req.valid('param')` | Validated + typed path params |
| `c.req.valid('query')` | Validated + typed query params |
| `c.get('key')` | Read context variable (set by middleware) |
| `c.set('key', val)` | Write context variable |
| `c.header('X-Foo', 'bar')` | Set response header |
| `c.status(201)` | Set status without body |
| `c.env` | CF Workers env (Bun: use `process.env` directly) |
| `c.var.myVar` | Shorthand for `c.get('myVar')` |

### Typed app bindings
```ts
// Typed environment variables (Bindings) + typed context variables (Variables)
type Bindings = { DATABASE_URL: string; JWT_SECRET: string }
type Variables = { userId: string; role: 'user' | 'admin' }

const app = new Hono<{ Bindings: Bindings; Variables: Variables }>()
// Now c.env, c.get(), c.set() are all fully typed
```

### HTTPException
```ts
import { HTTPException } from 'hono/http-exception'

// Throw anywhere — caught by onError
throw new HTTPException(404, { message: 'User not found' })
throw new HTTPException(401, { message: 'Unauthorized' })
```

---

## Which reference to read

- **Starting a new project** → `scaffolding.md`
- **Routes, sub-apps, middleware, streaming, RPC** → `routing.md`
- **Login/signup, JWT, protected routes, refresh tokens, OAuth** → `auth.md`
- **Drizzle + Postgres, MongoDB, queries, migrations** → `database.md`
- **Zod validation, OpenAPI docs, Scalar UI** → `openapi-zod.md`
- **WebSockets, SSE, file uploads, rate limiting, body limit, timeouts** → `advanced.md`
- **Unit tests, integration tests, bun:test, mocking DB** → `testing.md`
- **Docker, Railway, Fly.io, Render, health checks** → `deployment.md`

For full project generation, read: `scaffolding.md` + relevant feature files, then generate all files at once.
