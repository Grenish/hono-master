# Routing & Middleware Patterns

## Basic routing

```ts
app.get('/posts', (c) => c.json({ posts: [] }))
app.post('/posts', async (c) => {
  const body = await c.req.json()
  return c.json(body, 201)
})
app.put('/posts/:id', (c) => c.json({ id: c.req.param('id') }))
app.delete('/posts/:id', (c) => c.json({ deleted: true }))
app.all('/any', (c) => c.text('any method'))
```

## Sub-apps (app.route)

```ts
// routes/users.ts
import { Hono } from 'hono'

const app = new Hono()
app.get('/', (c) => c.json({ users: [] }))
app.get('/:id', (c) => c.json({ id: c.req.param('id') }))
app.post('/', async (c) => c.json(await c.req.json(), 201))
export default app

// app.ts
app.route('/users', usersApp)
app.route('/posts', postsApp)
```

## Chained method API (required for RPC type inference)

```ts
// routes/users.ts
const app = new Hono()
  .get('/', (c) => c.json({ users: [] as User[] }))
  .post('/', async (c) => c.json(await c.req.json<CreateUser>(), 201))
  .get('/:id', (c) => c.json({ id: c.req.param('id') }))

export default app
export type AppType = typeof app   // export for hc<AppType>()
```

## ⚠️ Don't make controllers

Path param types are only inferred when the handler is inline:
```ts
// ❌ Wrong — loses path param types
const handler = (c: Context) => c.json(c.req.param('id'))
app.get('/posts/:id', handler)

// ✅ Correct
app.get('/posts/:id', (c) => c.json(c.req.param('id')))  // 'id' is typed
```

If you need a reusable handler, use `createFactory`:
```ts
import { createFactory } from 'hono/factory'
const factory = createFactory()
const handlers = factory.createHandlers(authMiddleware, (c) => c.json(c.var.userId))
app.get('/me', ...handlers)
```

## Built-in middleware

```ts
import { logger } from 'hono/logger'
import { cors } from 'hono/cors'
import { secureHeaders } from 'hono/secure-headers'
import { compress } from 'hono/compress'
import { timeout } from 'hono/timeout'
import { requestId } from 'hono/request-id'
import { bodyLimit } from 'hono/body-limit'
import { ipRestriction } from 'hono/ip-restriction'
import { trimTrailingSlash } from 'hono/trailing-slash'
import { timing } from 'hono/timing'

app.use(logger())
app.use(requestId())          // adds X-Request-Id header
app.use(timing())             // adds Server-Timing header
app.use(secureHeaders())      // CSP, HSTS, X-Frame-Options etc
app.use(cors({ origin: process.env.ALLOWED_ORIGINS?.split(',') ?? '*' }))
app.use(compress())
app.use(trimTrailingSlash())
app.use(timeout(10_000))      // 10s global timeout
app.use('/upload/*', bodyLimit({ maxSize: 10 * 1024 * 1024 }))  // 10MB
```

## Custom middleware (createMiddleware)

```ts
import { createMiddleware } from 'hono/factory'

// Typed middleware — injects variables downstream
export const requireAuth = createMiddleware<{
  Variables: { userId: string; role: string }
}>(async (c, next) => {
  // ... verify JWT ...
  c.set('userId', userId)
  c.set('role', role)
  await next()
})

// Apply selectively
app.use('/api/*', requireAuth)
app.get('/api/me', (c) => c.json({ userId: c.get('userId') }))
```

## Error handling

```ts
import { HTTPException } from 'hono/http-exception'

// Global error handler
app.onError((err, c) => {
  if (err instanceof HTTPException) return err.getResponse()
  console.error(`[${c.get('requestId')}]`, err)
  return c.json({ error: 'Internal server error' }, 500)
})

app.notFound((c) => c.json({ error: `Route ${c.req.path} not found` }, 404))

// In route handlers
throw new HTTPException(422, { message: 'Invalid input' })
```

## Query params

```ts
// Single value
const { q, page = '1', limit = '20', sort = 'createdAt' } = c.req.query()

// Array (e.g. ?tag=a&tag=b)
const tags = c.req.queries('tag') ?? []

// Typed via zValidator
app.get('/search',
  zValidator('query', z.object({
    q: z.string().min(1),
    page: z.coerce.number().int().positive().default(1),
    limit: z.coerce.number().int().min(1).max(100).default(20),
  })),
  (c) => {
    const { q, page, limit } = c.req.valid('query')
    return c.json({ q, page, limit })
  }
)
```

## Streaming responses (SSE)

```ts
import { streamText, stream } from 'hono/streaming'
import { streamSSE } from 'hono/streaming'

// Plain streaming
app.get('/stream', (c) =>
  streamText(c, async (s) => {
    for (const chunk of chunks) {
      await s.write(chunk)
      await s.sleep(50)
    }
  })
)

// Server-Sent Events
app.get('/events', (c) =>
  streamSSE(c, async (sse) => {
    let id = 0
    while (true) {
      await sse.writeSSE({ data: JSON.stringify({ id: id++ }), event: 'update' })
      await sse.sleep(1000)
    }
  })
)
```

## RPC-style type-safe client

```ts
// server/routes/posts.ts — chained methods required
export const postsRoutes = new Hono()
  .get('/', (c) => c.json({ posts: [] as Post[] }))
  .post('/', async (c) => c.json(await c.req.json<CreatePost>(), 201))
export type PostsRoutes = typeof postsRoutes

// client (browser or test)
import { hc } from 'hono/client'
import type { PostsRoutes } from '../server/routes/posts'

const client = hc<PostsRoutes>('http://localhost:3000')
const res = await client.posts.$get()
const { posts } = await res.json()   // typed as Post[]
```

## Context Storage (global access to context)

```ts
import { contextStorage } from 'hono/context-storage'

app.use(contextStorage())

// Now accessible anywhere (outside handlers) via getContext()
import { getContext } from 'hono/context-storage'
function getCurrentUserId() {
  return getContext<AppType>().get('userId')
}
```
