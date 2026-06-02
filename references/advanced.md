# Advanced Features: WebSockets, SSE, File Uploads, Rate Limiting

## WebSockets

Hono has a built-in `upgradeWebSocket` helper. On Bun, use `hono/bun`.

```ts
import { Hono } from 'hono'
import { createBunWebSocket } from 'hono/bun'

const { upgradeWebSocket, websocket } = createBunWebSocket()
const app = new Hono()

app.get('/ws', upgradeWebSocket((c) => {
  return {
    onOpen(event, ws) {
      console.log('Client connected')
      ws.send('Welcome!')
    },
    onMessage(event, ws) {
      console.log('Received:', event.data)
      ws.send(`Echo: ${event.data}`)
    },
    onClose(event, ws) {
      console.log('Client disconnected')
    },
    onError(event, ws) {
      console.error('WS error:', event)
    },
  }
}))

// Entry point must pass the websocket handler to Bun
export default {
  port: 3000,
  fetch: app.fetch,
  websocket,   // ← required for Bun WS to work
}
```

### WS with auth (query param token)
```ts
app.get('/ws', upgradeWebSocket(async (c) => {
  const token = c.req.query('token')
  if (!token) return  // reject connection

  try {
    const payload = await verify(token, env.JWT_SECRET)
    return {
      onOpen(e, ws) { ws.send(JSON.stringify({ type: 'connected', userId: payload.sub })) },
      onMessage(e, ws) { /* handle */ },
    }
  } catch {
    return   // reject invalid token
  }
}))
```

### Broadcasting to all clients
```ts
// Simple in-memory hub (single instance; use Redis pub/sub for multi-process)
const clients = new Set<WSContext>()

app.get('/ws', upgradeWebSocket((c) => ({
  onOpen(e, ws) { clients.add(ws) },
  onClose(e, ws) { clients.delete(ws) },
  onMessage(e, ws) {
    // broadcast to all
    for (const client of clients) {
      client.send(e.data as string)
    }
  },
})))
```

---

## Server-Sent Events (SSE)

```ts
import { streamSSE } from 'hono/streaming'

app.get('/events', requireAuth, (c) => {
  return streamSSE(c, async (sse) => {
    // Set up cleanup on client disconnect
    let active = true
    c.req.raw.signal.addEventListener('abort', () => { active = false })

    while (active) {
      const data = await getLatestData()
      await sse.writeSSE({
        id: String(Date.now()),
        event: 'update',
        data: JSON.stringify(data),
      })
      await sse.sleep(5000)
    }
  })
})
```

---

## File Uploads

### Single file upload
```ts
import { bodyLimit } from 'hono/body-limit'

app.post(
  '/upload',
  bodyLimit({ maxSize: 10 * 1024 * 1024, onError: (c) => c.json({ error: 'File too large' }, 413) }),
  async (c) => {
    const body = await c.req.parseBody()
    const file = body['file'] as File

    if (!file || !(file instanceof File)) {
      return c.json({ error: 'No file provided' }, 400)
    }

    // Validate type
    const allowed = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf']
    if (!allowed.includes(file.type)) {
      return c.json({ error: 'Unsupported file type' }, 415)
    }

    // Read as buffer and write to disk (Bun)
    const buffer = await file.arrayBuffer()
    const filename = `${crypto.randomUUID()}-${file.name.replace(/[^a-z0-9.-]/gi, '_')}`
    await Bun.write(`./uploads/${filename}`, buffer)

    return c.json({ url: `/files/${filename}`, size: file.size, type: file.type })
  }
)
```

### Multiple files
```ts
app.post('/upload-many', async (c) => {
  const body = await c.req.parseBody({ all: true })
  const files = body['files[]']
  const fileList = Array.isArray(files) ? files : [files]

  const results = await Promise.all(
    (fileList as File[]).map(async (file) => {
      const name = `${crypto.randomUUID()}-${file.name}`
      await Bun.write(`./uploads/${name}`, await file.arrayBuffer())
      return { name, size: file.size }
    })
  )
  return c.json({ uploaded: results })
})
```

### Serve static files (Bun)
```ts
import { serveStatic } from 'hono/bun'

app.use('/files/*', serveStatic({ root: './uploads', rewriteRequestPath: (p) => p.replace('/files', '') }))
```

---

## Rate Limiting

```bash
bun add hono-rate-limiter
```

### Basic (in-memory, single process)
```ts
import { rateLimiter } from 'hono-rate-limiter'

app.use(
  '/api/*',
  rateLimiter({
    windowMs: 15 * 60 * 1000,   // 15 minutes
    limit: 100,                  // max 100 requests per window per IP
    standardHeaders: 'draft-6', // returns RateLimit-* headers
    keyGenerator: (c) => c.req.header('x-forwarded-for') ?? c.req.header('x-real-ip') ?? 'unknown',
  })
)
```

### Stricter limits on auth endpoints
```ts
const authLimiter = rateLimiter({
  windowMs: 15 * 60 * 1000,
  limit: 10,
  message: 'Too many auth attempts, try again later',
  keyGenerator: (c) => c.req.header('x-forwarded-for') ?? 'unknown',
})

app.use('/auth/login', authLimiter)
app.use('/auth/signup', authLimiter)
```

### Redis store (production multi-instance)
```bash
bun add @hono-rate-limiter/redis @upstash/redis
```
```ts
import { RedisStore } from '@hono-rate-limiter/redis'
import { Redis } from '@upstash/redis'

const redis = new Redis({ url: env.UPSTASH_URL, token: env.UPSTASH_TOKEN })

app.use('/api/*', rateLimiter({
  windowMs: 15 * 60 * 1000,
  limit: 100,
  store: new RedisStore({ client: redis }),
  keyGenerator: (c) => c.req.header('x-forwarded-for') ?? 'unknown',
}))
```

---

## IP Restriction (built-in)

```ts
import { ipRestriction } from 'hono/ip-restriction'
import { getConnInfo } from 'hono/bun'

app.use('/admin/*', ipRestriction(getConnInfo, {
  denyList: ['192.168.1.100'],
  allowList: ['10.0.0.0/8', '172.16.0.0/12'],
}))
```

---

## Timeout (built-in)

```ts
import { timeout } from 'hono/timeout'
import { HTTPException } from 'hono/http-exception'

// Global 10s timeout
app.use('*', timeout(10_000))

// Custom per-route timeout with handler
app.get('/slow',
  timeout(30_000, (c) => new HTTPException(504, { message: 'Request timed out' })),
  slowHandler
)
```

---

## Body size limit (built-in)

```ts
import { bodyLimit } from 'hono/body-limit'

// Global 1MB limit
app.use('*', bodyLimit({ maxSize: 1024 * 1024 }))

// Override for upload route
app.use('/upload', bodyLimit({ maxSize: 50 * 1024 * 1024 }))
```
