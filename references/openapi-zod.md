# OpenAPI + Zod Validation

## Install

```bash
bun add @hono/zod-openapi @scalar/hono-api-reference zod
```

## Setup — OpenAPIHono app

```ts
// src/app.ts
import { OpenAPIHono } from '@hono/zod-openapi'
import { apiReference } from '@scalar/hono-api-reference'

export function createApp() {
  const app = new OpenAPIHono()

  // OpenAPI spec endpoint
  app.doc('/openapi.json', {
    openapi: '3.0.0',
    info: { title: 'My API', version: '1.0.0' },
  })

  // Interactive docs UI (Scalar)
  app.get('/docs', apiReference({ spec: { url: '/openapi.json' } }))

  return app
}
```

## Defining a route with Zod + OpenAPI

```ts
import { createRoute, z } from '@hono/zod-openapi'

// 1. Define schemas
const ParamSchema = z.object({
  id: z.string().uuid().openapi({ example: '123e4567-e89b-12d3-a456-426614174000' }),
})

const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  role: z.enum(['user', 'admin']),
}).openapi('User')

const CreateUserSchema = z.object({
  email: z.string().email().openapi({ example: 'user@example.com' }),
  password: z.string().min(8),
}).openapi('CreateUser')

const ErrorSchema = z.object({ error: z.string() }).openapi('Error')

// 2. Define route
const getUserRoute = createRoute({
  method: 'get',
  path: '/users/{id}',
  tags: ['Users'],
  request: { params: ParamSchema },
  responses: {
    200: { content: { 'application/json': { schema: UserSchema } }, description: 'User found' },
    404: { content: { 'application/json': { schema: ErrorSchema } }, description: 'Not found' },
  },
})

// 3. Implement
userRoutes.openapi(getUserRoute, async (c) => {
  const { id } = c.req.valid('param')   // fully typed
  const user = await db.query.users.findFirst({ where: eq(users.id, id) })
  if (!user) return c.json({ error: 'Not found' }, 404)
  return c.json(user, 200)
})
```

## createRoute — full example with body

```ts
const createUserRoute = createRoute({
  method: 'post',
  path: '/users',
  tags: ['Users'],
  request: {
    body: {
      content: { 'application/json': { schema: CreateUserSchema } },
      required: true,
    },
  },
  responses: {
    201: { content: { 'application/json': { schema: UserSchema } }, description: 'Created' },
    409: { content: { 'application/json': { schema: ErrorSchema } }, description: 'Already exists' },
  },
})

userRoutes.openapi(createUserRoute, async (c) => {
  const body = c.req.valid('json')   // typed as CreateUser
  // ...
})
```

## Global validation error handler (defaultHook)

```ts
const app = new OpenAPIHono({
  defaultHook: (result, c) => {
    if (!result.success) {
      return c.json({ error: 'Validation error', issues: result.error.issues }, 422)
    }
  },
})
```

## Simple validation without OpenAPI (zValidator)

For lightweight routes that don't need docs:

```ts
import { zValidator } from '@hono/zod-validator'
import { z } from 'zod'

app.post('/simple', zValidator('json', z.object({ name: z.string() })), (c) => {
  const { name } = c.req.valid('json')
  return c.json({ hello: name })
})

// Also works for query, param, form, header
app.get('/search', zValidator('query', z.object({ q: z.string().min(1) })), (c) => {
  const { q } = c.req.valid('query')
  return c.json({ results: [] })
})
```

## Drizzle schema → Zod → OpenAPI

```ts
import { createSchemaFactory } from 'drizzle-orm/zod'
import { z } from '@hono/zod-openapi'

const { createInsertSchema, createSelectSchema } = createSchemaFactory({ zodInstance: z })

// These schemas are compatible with createRoute directly
export const InsertPostSchema = createInsertSchema(posts).omit({ id: true, createdAt: true })
export const SelectPostSchema = createSelectSchema(posts)
```

## Mounting an OpenAPIHono sub-router

```ts
// routes/users.ts
export const userRoutes = new OpenAPIHono()
userRoutes.openapi(getUserRoute, handler)

// app.ts
app.route('/users', userRoutes)
// The openapi.json will include all routes from sub-routers automatically
```
