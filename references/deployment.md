# Deployment

## Docker (universal)

```dockerfile
# Dockerfile
FROM oven/bun:1-alpine AS base
WORKDIR /app

# Install dependencies
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile --production

# Copy source
COPY src ./src
COPY tsconfig.json ./

EXPOSE 3000

CMD ["bun", "run", "src/index.ts"]
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/myapp
      JWT_SECRET: supersecretkey32charsminimum!!
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

Run:
```bash
docker compose up --build
```

---

## Railway

Railway auto-detects Bun. Just connect your repo.

1. Push to GitHub
2. New project → Deploy from repo
3. Add Postgres plugin (auto-sets `DATABASE_URL`)
4. Set env vars in Railway dashboard
5. Railway uses `bun run start` by default — ensure your `package.json` has it

Optional `railway.json`:
```json
{
  "build": { "builder": "NIXPACKS" },
  "deploy": { "startCommand": "bun run src/index.ts" }
}
```

---

## Fly.io

```bash
# Install flyctl, then:
fly launch   # detects Bun, generates fly.toml
fly deploy
```

`fly.toml` (adjust port):
```toml
[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true

[[vm]]
  memory = "256mb"
  cpu_kind = "shared"
  cpus = 1
```

Set secrets:
```bash
fly secrets set DATABASE_URL="..." JWT_SECRET="..."
```

For Postgres:
```bash
fly postgres create --name myapp-db
fly postgres attach myapp-db
# DATABASE_URL is set automatically
```

---

## Render

1. New Web Service → connect GitHub repo
2. Build command: `bun install`
3. Start command: `bun run src/index.ts`
4. Add Postgres database (Render managed)
5. Set env vars in dashboard

---

## Running migrations on deploy

Add a migration step before starting the server:

```ts
// src/index.ts
import { migrate } from 'drizzle-orm/postgres-js/migrator'
import { db, client } from './db'
import { env } from './env'

if (env.NODE_ENV === 'production') {
  await migrate(db, { migrationsFolder: './drizzle' })
}
```

Or as a separate script:
```bash
# package.json
"db:deploy": "drizzle-kit migrate"
# Run before starting the server in CI/CD
```

---

## Health check endpoint (recommended)

```ts
app.get('/health', async (c) => {
  try {
    await db.execute(sql`SELECT 1`)
    return c.json({ status: 'ok', db: 'ok' })
  } catch {
    return c.json({ status: 'error', db: 'unreachable' }, 503)
  }
})
```
