# Hono Master

> An AI skill for building production-grade Hono web servers — from first route to deployment.

---

## What It Does

**Hono Master** equips an AI agent with deep, up-to-date knowledge of the [Hono](https://hono.dev) web framework running on [Bun](https://bun.sh). Rather than relying on generic web framework knowledge, the skill provides concrete patterns, opinionated defaults, and working code for every layer of a real Hono API — authentication, database access, validation, docs, testing, and deployment.

The skill is structured as a main index (`SKILL.md`) that routes to focused reference files, so the agent loads only what's relevant to the task at hand.

---

## Installation

Hono Master is distributed via GitHub and installable into any compatible AI agent (Claude Code, Cursor, Codex, and 50+ others) using the `skills` CLI.

**Install into your project:**

```bash
npx skills add grenish/hono-master
```

**Or install globally:**

```bash
npx skills add grenish/hono-master -g
```

**Preview available skills before installing:**

```bash
npx skills add grenish/hono-master --list
```

**Target a specific agent:**

```bash
npx skills add grenish/hono-master -a claude-code
```

Once installed, no further configuration is needed. The skill activates automatically when a relevant task is detected.

---

## Coverage

| Area | What's Included |
|---|---|
| **Scaffolding** | `bun create hono`, project structure, Zod env validation, `tsconfig`, all core dependencies |
| **Routing** | Sub-apps, chained API, middleware factory, error handling, streaming, SSE, RPC client |
| **Auth** | JWT sign/verify, access + refresh token pair, cookie-based auth, role guards, Better Auth (OAuth) |
| **Database** | Drizzle ORM + Postgres (client, schema, queries, migrations), MongoDB + Mongoose |
| **OpenAPI / Validation** | `@hono/zod-openapi`, `createRoute`, `zValidator`, Scalar docs UI, Drizzle→Zod derivation |
| **Advanced** | WebSockets (Bun), Server-Sent Events, file uploads, rate limiting, IP restriction, timeouts |
| **Testing** | `bun:test` + `app.request()`, auth mocking, DB mocking, multipart uploads, WebSocket tests |
| **Deployment** | Docker + Compose, Railway, Fly.io, Render, migration-on-deploy, health check endpoint |

---

## Design Principles

The skill enforces a consistent set of opinions so generated code is coherent, not a patchwork:

- **Bun-first** — `bun run`, `bun:test`, Bun file I/O, Bun WebSocket handler. Node.js compat is documented but secondary.
- **TypeScript by default** — strict mode, typed bindings (`Bindings`/`Variables`), Zod-inferred types throughout.
- **Zod as the single source of truth** — schemas defined once at the boundary, types derived from them, never duplicated.
- **Drizzle as the ORM** — `drizzle-orm` with `postgres-js`; schema-to-Zod via `createSchemaFactory` (not the deprecated `drizzle-zod`).
- **Lean middleware stack** — Hono ships logger, CORS, JWT, timeout, rate limiter, IP restriction, body limit, and more. Reach for third-party packages only when the built-in falls short.
- **Don't make controllers** — inline handlers preserve path-param type inference. The skill flags this anti-pattern and shows the correct `createFactory` alternative when reuse is genuinely needed.

---

## File Structure

```
hono-master/
├── SKILL.md                    # Entry point — principles, cheatsheet, routing table
└── references/
    ├── scaffolding.md          # Project setup, structure, deps, env
    ├── routing.md              # Routes, middleware, streaming, RPC
    ├── auth.md                 # JWT, token pairs, protected routes, OAuth
    ├── database.md             # Drizzle + Postgres, MongoDB, queries, migrations
    ├── openapi-zod.md          # OpenAPI routes, Zod validation, Scalar UI
    ├── advanced.md             # WebSockets, SSE, uploads, rate limiting
    ├── testing.md              # bun:test patterns, mocking, integration tests
    └── deployment.md           # Docker, Railway, Fly.io, Render
```

---

## Usage

Install the skill into your AI agent environment and it will automatically engage whenever a task involves Hono — whether scaffolding a new project, adding a feature to an existing one, debugging middleware, or shipping to production.

**Trigger keywords:** `hono`, `hono server`, `hono API`, `hono route`, `hono middleware`, `hono auth`, `hono drizzle`, `hono deployment`, `hono websocket`, `hono testing`, `build an API with Bun`, or any backend/API task where the stack includes Bun or Hono.

---

## Requirements

- [Bun](https://bun.sh) `>= 1.0`
- [Hono](https://hono.dev) `>= 4.0`
- TypeScript `>= 5.0` (optional but strongly recommended)
