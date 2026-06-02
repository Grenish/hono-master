# Testing Hono Apps

## Setup (bun:test — no extra install needed)

```ts
// src/app.ts — export the app factory so tests can import it
export function createApp() {
  const app = new Hono()
  // ... routes ...
  return app
}

// src/index.ts — entry point only
import { createApp } from './app'
export default { port: 3000, fetch: createApp().fetch }
```

## app.request() — the core test primitive

`app.request(path, init?)` creates a real `Request` and returns a `Response`.
No HTTP server needed. Fast. Works with `bun:test`.

```ts
import { describe, it, expect, beforeEach } from 'bun:test'
import { createApp } from '../src/app'

const app = createApp()

describe('GET /posts', () => {
  it('returns 200 with posts array', async () => {
    const res = await app.request('/posts')
    expect(res.status).toBe(200)
    const body = await res.json()
    expect(body.posts).toBeArray()
  })
})

describe('POST /posts', () => {
  it('creates a post and returns 201', async () => {
    const res = await app.request('/posts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title: 'Hello', content: 'World' }),
    })
    expect(res.status).toBe(201)
    const body = await res.json()
    expect(body.title).toBe('Hello')
  })

  it('returns 422 on invalid input', async () => {
    const res = await app.request('/posts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title: '' }),   // missing content
    })
    expect(res.status).toBe(422)
  })
})
```

## Testing authenticated routes

```ts
import { sign } from 'hono/jwt'

async function makeAuthHeader(userId = 'user-1', role = 'user') {
  const token = await sign(
    { sub: userId, role, exp: Math.floor(Date.now() / 1000) + 3600 },
    'test-secret-at-least-32-chars-long!!'
  )
  return { Authorization: `Bearer ${token}` }
}

it('GET /me returns current user', async () => {
  const res = await app.request('/api/me', { headers: await makeAuthHeader() })
  expect(res.status).toBe(200)
  expect((await res.json()).userId).toBe('user-1')
})

it('returns 401 without token', async () => {
  const res = await app.request('/api/me')
  expect(res.status).toBe(401)
})
```

## Mocking the database layer

Don't hit a real DB in unit tests. Use a simple mock module.

```ts
// src/db/index.ts
export const db = drizzle(client)   // real client

// src/__mocks__/db.ts (Bun mock)
export const db = {
  select: () => ({ from: () => ({ where: () => ({ limit: () => Promise.resolve([mockUser]) }) }) }),
  insert: () => ({ values: () => ({ returning: () => Promise.resolve([mockUser]) }) }),
}

// In tests:
import { mock } from 'bun:test'
mock.module('../src/db', () => import('../src/__mocks__/db'))
```

For integration tests (testing real DB), use a test database or in-memory SQLite:
```bash
bun add -d better-sqlite3 drizzle-orm
```
```ts
import Database from 'bun:sqlite'
import { drizzle } from 'drizzle-orm/bun-sqlite'

const testDb = drizzle(new Database(':memory:'))
// Run migrations: await migrate(testDb, { migrationsFolder: './drizzle' })
```

## Testing multipart / form-data uploads

```ts
it('uploads a file', async () => {
  const form = new FormData()
  form.append('file', new File(['hello world'], 'test.txt', { type: 'text/plain' }))

  const res = await app.request('/upload', { method: 'POST', body: form })
  expect(res.status).toBe(200)
  const { url } = await res.json()
  expect(url).toStartWith('/files/')
})
```

## Testing WebSockets (Bun)

```ts
import { describe, it, expect } from 'bun:test'

it('websocket echoes messages', async () => {
  const server = Bun.serve({ port: 0, fetch: app.fetch, websocket })
  const ws = new WebSocket(`ws://localhost:${server.port}/ws`)

  await new Promise<void>((resolve) => { ws.onopen = () => resolve() })
  ws.send('hello')

  const msg = await new Promise<string>((resolve) => { ws.onmessage = (e) => resolve(e.data as string) })
  expect(msg).toBe('Echo: hello')

  ws.close()
  server.stop()
})
```

## Running tests

```bash
bun test                          # run all tests
bun test --watch                  # watch mode
bun test src/routes/auth.test.ts  # single file
bun test --coverage               # coverage report
```

## Test file conventions

```
src/
├── routes/
│   ├── auth.ts
│   └── auth.test.ts    # co-locate tests with source
├── middleware/
│   ├── auth.ts
│   └── auth.test.ts
└── app.test.ts         # integration/smoke tests
```

## Useful bun:test matchers

```ts
expect(res.status).toBe(200)
expect(body).toEqual({ id: '123', name: 'test' })
expect(body.items).toBeArray()
expect(body.items).toHaveLength(3)
expect(body.token).toBeString()
expect(fn).toHaveBeenCalledTimes(1)
expect(fn).toHaveBeenCalledWith('arg1', expect.any(String))
```

## HEAD request testing

Hono auto-converts HEAD to GET and strips the body. Test both together:
```ts
it('HEAD /posts returns same status and headers as GET', async () => {
  const get = await app.request('/posts')
  const head = await app.request('/posts', { method: 'HEAD' })
  expect(head.status).toBe(get.status)
  expect(head.body).toBeNull()
  expect(head.headers.get('content-type')).toBe(get.headers.get('content-type'))
})
```
