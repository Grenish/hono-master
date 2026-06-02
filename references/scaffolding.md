# Hono Project Scaffolding

## Quickstart (Bun)

```bash
bun create hono@latest my-app
# Select: bun template
cd my-app && bun install
bun run dev   # http://localhost:3000
```

## Recommended Project Structure

```
my-app/
├── src/
│   ├── index.ts           # Entry point — creates app, mounts routers
│   ├── app.ts             # App factory (useful for testing)
│   ├── env.ts             # Zod-validated env
│   ├── db/
│   │   ├── index.ts       # Drizzle client
│   │   └── schema.ts      # Table definitions
│   ├── routes/
│   │   ├── index.ts       # Re-exports all routers
│   │   ├── auth.ts
│   │   └── users.ts
│   └── middleware/
│       ├── auth.ts        # JWT verification middleware
│       └── error.ts       # Global error handler
├── drizzle/               # Migration files (drizzle-kit generate)
├── .env
├── tsconfig.json
└── package.json
```

## tsconfig.json (strict, Bun)

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "skipLibCheck": true,
    "types": ["bun-types"]
  },
  "include": ["src"]
}
```

## Zod-validated env (src/env.ts)

```ts
import { z } from 'zod'

const EnvSchema = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  PORT: z.coerce.number().default(3000),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
})

export const env = EnvSchema.parse(process.env)
export type Env = z.infer<typeof EnvSchema>
```

## App factory pattern (src/app.ts)

```ts
import { Hono } from 'hono'
import { logger } from 'hono/logger'
import { cors } from 'hono/cors'
import { authRoutes } from './routes/auth'
import { userRoutes } from './routes/users'

export function createApp() {
  const app = new Hono()

  // Global middleware
  app.use('*', logger())
  app.use('*', cors())

  // Routes
  app.route('/auth', authRoutes)
  app.route('/users', userRoutes)

  // 404
  app.notFound((c) => c.json({ error: 'Not found' }, 404))

  // Global error handler
  app.onError((err, c) => {
    console.error(err)
    return c.json({ error: err.message }, 500)
  })

  return app
}
```

## Entry point (src/index.ts)

```ts
import { createApp } from './app'
import { env } from './env'

const app = createApp()

console.log(`Server running on http://localhost:${env.PORT}`)

export default {
  port: env.PORT,
  fetch: app.fetch,
}
```

## package.json scripts

```json
{
  "scripts": {
    "dev": "bun run --hot src/index.ts",
    "start": "bun run src/index.ts",
    "test": "bun test",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "db:studio": "drizzle-kit studio"
  }
}
```

## Core dependencies

```bash
# Runtime
bun add hono zod

# Database (Postgres + Drizzle)
bun add drizzle-orm postgres
bun add -d drizzle-kit

# OpenAPI / validation
bun add @hono/zod-openapi @scalar/hono-api-reference

# Auth
bun add hono/jwt   # built into hono — no separate install
bun add bcryptjs
bun add -d @types/bcryptjs

# Optional utils
bun add hono-rate-limiter
```

## JavaScript variant

For plain JS, omit `tsconfig.json`, change file extensions to `.js`, and remove type annotations. Bun runs JS natively. The `bun create hono` template defaults to TS — choose "No" when prompted about TypeScript to get a JS scaffold.

---

## Full production dependency list

```bash
# Core
bun add hono zod

# OpenAPI + docs
bun add @hono/zod-openapi @hono/zod-validator @scalar/hono-api-reference

# Auth
bun add bcryptjs
bun add -d @types/bcryptjs

# Database — Postgres
bun add drizzle-orm postgres
bun add -d drizzle-kit

# Rate limiting
bun add hono-rate-limiter

# Optional (OAuth/social)
bun add better-auth
```

## .env.example

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/myapp
JWT_SECRET=change-me-to-at-least-32-random-characters!!
PORT=3000
NODE_ENV=development
ALLOWED_ORIGINS=http://localhost:5173
```

## Node.js compatibility

Swap the Bun-specific entry point:

```ts
// src/index.node.ts
import { serve } from '@hono/node-server'
import { createApp } from './app'
serve({ fetch: createApp().fetch, port: Number(process.env.PORT ?? 3000) })
```

Everything else (routes, middleware, auth, DB) stays identical across runtimes.
