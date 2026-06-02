# Database Integration

## PostgreSQL with Drizzle ORM

### Install

```bash
bun add drizzle-orm postgres
bun add -d drizzle-kit
```

### Drizzle client (src/db/index.ts)

```ts
import { drizzle } from 'drizzle-orm/postgres-js'
import postgres from 'postgres'
import { env } from '../env'
import * as schema from './schema'

const client = postgres(env.DATABASE_URL)
export const db = drizzle(client, { schema })
```

### Schema example (src/db/schema.ts)

```ts
import { pgTable, text, timestamp, uuid, boolean } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id: uuid('id').defaultRandom().primaryKey(),
  email: text('email').notNull().unique(),
  password: text('password').notNull(),
  role: text('role', { enum: ['user', 'admin'] }).default('user').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
})

export const posts = pgTable('posts', {
  id: uuid('id').defaultRandom().primaryKey(),
  title: text('title').notNull(),
  content: text('content').notNull(),
  authorId: uuid('author_id').references(() => users.id, { onDelete: 'cascade' }).notNull(),
  published: boolean('published').default(false).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
})
```

### drizzle.config.ts

```ts
import { defineConfig } from 'drizzle-kit'
import { env } from './src/env'

export default defineConfig({
  schema: './src/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: { url: env.DATABASE_URL },
})
```

### Common queries

```ts
import { db } from '../db'
import { users, posts } from '../db/schema'
import { eq, and, like, desc, sql } from 'drizzle-orm'

// Select
const allUsers = await db.select().from(users)
const user = await db.select().from(users).where(eq(users.id, id)).limit(1)

// Insert + return
const [newUser] = await db.insert(users).values({ email, password }).returning()

// Update
await db.update(users).set({ role: 'admin' }).where(eq(users.id, id))

// Delete
await db.delete(users).where(eq(users.id, id))

// Join
const postsWithAuthors = await db
  .select({ post: posts, author: users })
  .from(posts)
  .leftJoin(users, eq(posts.authorId, users.id))
  .where(eq(posts.published, true))
  .orderBy(desc(posts.createdAt))

// Pagination
const page = 1, limit = 20
const results = await db.select().from(posts)
  .limit(limit)
  .offset((page - 1) * limit)

// Count
const [{ count }] = await db.select({ count: sql<number>`count(*)` }).from(users)
```

### Zod schema from Drizzle (drizzle-orm >= 1.0.0-beta.15)

```ts
// drizzle-zod is deprecated; use createSchemaFactory from drizzle-orm/zod
import { createSchemaFactory } from 'drizzle-orm/zod'
import { z } from '@hono/zod-openapi'  // use extended zod if you use zod-openapi

const { createInsertSchema, createSelectSchema } = createSchemaFactory({ zodInstance: z })

export const InsertUserSchema = createInsertSchema(users, {
  email: (s) => s.email(),
  password: (s) => s.min(8),
}).omit({ id: true, createdAt: true })

export const SelectUserSchema = createSelectSchema(users).omit({ password: true })
```

### Migrations

```bash
bun run db:generate   # generates SQL migration files
bun run db:migrate    # applies migrations
bun run db:studio     # opens Drizzle Studio in browser
```

---

## MongoDB

### Install

```bash
bun add mongoose
# or native driver
bun add mongodb
```

### Mongoose connection (src/db/mongo.ts)

```ts
import mongoose from 'mongoose'
import { env } from '../env'

export async function connectMongo() {
  await mongoose.connect(env.MONGODB_URI)
  console.log('MongoDB connected')
}

// Call in entry point
// src/index.ts
await connectMongo()
```

### Schema + model example

```ts
import { Schema, model, Types } from 'mongoose'

const PostSchema = new Schema({
  title: { type: String, required: true },
  content: { type: String, required: true },
  authorId: { type: Types.ObjectId, ref: 'User', required: true },
  published: { type: Boolean, default: false },
}, { timestamps: true })

export const Post = model('Post', PostSchema)
```

### Usage in route

```ts
import { Post } from '../db/models/post'

app.get('/posts', async (c) => {
  const posts = await Post.find({ published: true }).sort({ createdAt: -1 }).limit(20)
  return c.json(posts)
})

app.post('/posts', async (c) => {
  const body = await c.req.json()
  const post = await Post.create({ ...body, authorId: c.get('userId') })
  return c.json(post, 201)
})
```
