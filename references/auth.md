# Auth & JWT

## JWT with Hono's built-in helper

No extra install — `hono/jwt` is part of Hono.

```ts
import { sign, verify, decode } from 'hono/jwt'

// Supported algorithms: HS256 (default), HS384, HS512, RS256, RS384, RS512, PS256, PS384, PS512, ES256, ES384, ES512, EdDSA

// Sign
const token = await sign(
  { sub: userId, role, exp: Math.floor(Date.now() / 1000) + 15 * 60 },
  secret  // string for HS*, CryptoKey for RS*/ES*
)

// Verify (throws on invalid/expired)
const payload = await verify(token, secret)

// Decode without verification (for inspection)
const { header, payload } = decode(token)
```

## Token pair factory

```ts
// src/lib/tokens.ts
import { sign } from 'hono/jwt'
import { env } from '../env'

export async function makeTokens(userId: string, role: string) {
  const now = Math.floor(Date.now() / 1000)
  const accessToken = await sign(
    { sub: userId, role, exp: now + 15 * 60 },       // 15 min
    env.JWT_SECRET
  )
  const refreshToken = await sign(
    { sub: userId, type: 'refresh', exp: now + 30 * 24 * 60 * 60 },  // 30 days
    env.JWT_SECRET
  )
  return { accessToken, refreshToken }
}
```

## Auth middleware

```ts
// src/middleware/auth.ts
import { createMiddleware } from 'hono/factory'
import { verify } from 'hono/jwt'
import { HTTPException } from 'hono/http-exception'
import { env } from '../env'

type AuthVars = { userId: string; role: string }

export const requireAuth = createMiddleware<{ Variables: AuthVars }>(
  async (c, next) => {
    // Support both Bearer token and cookie
    const header = c.req.header('Authorization')
    const cookieToken = getCookie(c, 'access_token')
    const token = header?.startsWith('Bearer ') ? header.slice(7) : cookieToken

    if (!token) throw new HTTPException(401, { message: 'Missing token' })

    try {
      const payload = await verify(token, env.JWT_SECRET)
      c.set('userId', payload.sub as string)
      c.set('role', payload.role as string)
    } catch {
      throw new HTTPException(401, { message: 'Invalid or expired token' })
    }
    await next()
  }
)

export const requireRole = (...roles: string[]) =>
  createMiddleware<{ Variables: AuthVars }>(async (c, next) => {
    if (!roles.includes(c.get('role'))) {
      throw new HTTPException(403, { message: 'Forbidden' })
    }
    await next()
  })
```

## Auth routes (signup / login / refresh / logout)

```ts
// src/routes/auth.ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'
import { setCookie, deleteCookie } from 'hono/cookie'
import { verify } from 'hono/jwt'
import { z } from 'zod'
import bcrypt from 'bcryptjs'
import { db } from '../db'
import { users } from '../db/schema'
import { eq } from 'drizzle-orm'
import { makeTokens } from '../lib/tokens'
import { env } from '../env'

const AuthSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

export const authRoutes = new Hono()

  .post('/signup', zValidator('json', AuthSchema), async (c) => {
    const { email, password } = c.req.valid('json')
    const [existing] = await db.select({ id: users.id }).from(users).where(eq(users.email, email))
    if (existing) throw new HTTPException(409, { message: 'Email already registered' })

    const hashed = await bcrypt.hash(password, 12)
    const [user] = await db.insert(users).values({ email, password: hashed }).returning()
    const tokens = await makeTokens(user.id, user.role)

    setCookie(c, 'refresh_token', tokens.refreshToken, {
      httpOnly: true, secure: env.NODE_ENV === 'production',
      sameSite: 'Strict', maxAge: 30 * 24 * 60 * 60, path: '/auth/refresh',
    })

    return c.json({ accessToken: tokens.accessToken, user: { id: user.id, email: user.email } }, 201)
  })

  .post('/login', zValidator('json', AuthSchema), async (c) => {
    const { email, password } = c.req.valid('json')
    const [user] = await db.select().from(users).where(eq(users.email, email))
    if (!user || !(await bcrypt.compare(password, user.password))) {
      throw new HTTPException(401, { message: 'Invalid credentials' })
    }

    const tokens = await makeTokens(user.id, user.role)
    setCookie(c, 'refresh_token', tokens.refreshToken, {
      httpOnly: true, secure: env.NODE_ENV === 'production',
      sameSite: 'Strict', maxAge: 30 * 24 * 60 * 60, path: '/auth/refresh',
    })

    return c.json({ accessToken: tokens.accessToken })
  })

  .post('/refresh', async (c) => {
    const refreshToken = getCookie(c, 'refresh_token') ?? (await c.req.json()).refreshToken
    if (!refreshToken) throw new HTTPException(401, { message: 'No refresh token' })

    try {
      const payload = await verify(refreshToken, env.JWT_SECRET)
      if (payload.type !== 'refresh') throw new Error()
      const [user] = await db.select().from(users).where(eq(users.id, payload.sub as string))
      if (!user) throw new Error()

      const tokens = await makeTokens(user.id, user.role)
      return c.json({ accessToken: tokens.accessToken })
    } catch {
      throw new HTTPException(401, { message: 'Invalid refresh token' })
    }
  })

  .post('/logout', (c) => {
    deleteCookie(c, 'refresh_token', { path: '/auth/refresh' })
    return c.json({ ok: true })
  })
```

## Protecting routes

```ts
// Protect all routes in a sub-app
userRoutes.use('*', requireAuth)

// Protect specific routes with role check
app.delete('/admin/users/:id', requireAuth, requireRole('admin'), handler)

// Nested protected group
const adminRoutes = new Hono()
adminRoutes.use('*', requireAuth, requireRole('admin', 'moderator'))
adminRoutes.get('/dashboard', dashboardHandler)
app.route('/admin', adminRoutes)
```

## Hono's built-in JWT middleware (simple use cases)

For simpler scenarios where you just need to protect routes:
```ts
import { jwt } from 'hono/jwt'

app.use('/api/*', jwt({ secret: env.JWT_SECRET }))
// Access payload via c.get('jwtPayload')
app.get('/api/me', (c) => c.json(c.get('jwtPayload')))
```

## OAuth / Social login

For OAuth (Google, GitHub, Discord etc.), use **Better Auth**:
```bash
bun add better-auth
```

```ts
import { betterAuth } from 'better-auth'
import { drizzleAdapter } from 'better-auth/adapters/drizzle'

export const auth = betterAuth({
  database: drizzleAdapter(db, { provider: 'pg' }),
  socialProviders: {
    github: { clientId: env.GITHUB_ID, clientSecret: env.GITHUB_SECRET },
    google: { clientId: env.GOOGLE_ID, clientSecret: env.GOOGLE_SECRET },
  },
})
```
See https://www.better-auth.com/docs/integrations/hono for Hono integration.

## Password reset flow (outline)

1. `POST /auth/forgot-password` — generate a short-lived signed token, email it
2. `POST /auth/reset-password` — verify token, hash new password, invalidate token
3. Store reset tokens in DB with expiry; delete on use
