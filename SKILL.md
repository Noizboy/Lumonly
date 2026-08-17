---
name: observability-logs-api
description: >
  Implements a structured system-log pipeline for any Next.js (App Router) or Express app using Prisma + PostgreSQL.
  Adds a fire-and-forget logger, a SystemLog table, and a paginated REST API protected by an env-var API key —
  ready to feed an external observability dashboard, Grafana, or any monitoring tool.
  Use this skill whenever the user wants to: log system events, expose logs via API, build observability,
  monitor what the app is doing, debug production issues, or integrate with external monitoring systems.
  Also triggers on phrases such as "I want to see logs", "add logging", "observability system",
  "logs API", "system monitoring", or "see what the system does in production."
---

# Observability Logs API

A self-contained pattern to add structured logging + a query API to any Next.js (App Router) or Express app running Prisma + PostgreSQL. No external services needed — logs live in your existing database and are served by your existing app.

## Architecture overview

```
App code  ──fire-and-forget──▶  logger.ts  ──async──▶  SystemLog (Postgres)
                                                              │
External tool / dashboard ◀──── GET /api/public/logs ◀───────┘
                                (X-Api-Key header)
```

Key design decisions:
- **Never blocks callers** — all writes are `.catch(() => {})` fire-and-forget
- **No external service** — logs stay in your existing DB
- **Static API key** — set once in `.env`, no DB-backed CRUD
- **Structured metadata** — every log accepts a JSON `metadata` field for context

---

## Step 1 — Prisma schema

Add to `prisma/schema.prisma`:

```prisma
model SystemLog {
  id        String   @id @default(cuid())
  level     LogLevel
  category  String   // "auth" | "smtp" | "ai" | "campaign" | "api" | custom
  message   String
  metadata  Json?    // arbitrary structured context — provider, contactId, ip, etc.
  userId    String?  // null for anonymous/system events
  createdAt DateTime @default(now())

  @@index([createdAt])
  @@index([level])
  @@index([category])
  @@index([userId])
}

enum LogLevel {
  info
  warn
  error
}
```

Then regenerate the client:

```bash
npm run prisma:generate   # Next.js + Prisma projects
# OR
npx prisma generate       # standalone
```

---

## Step 2 — Logger utility (`lib/logger.ts`)

```typescript
import { prisma } from "@/lib/prisma";
import type { Prisma } from "@/app/generated/prisma/client";
// Adjust import path to match your project's Prisma output location

type Level = "info" | "warn" | "error";

function log(
  level: Level,
  category: string,
  message: string,
  metadata?: Record<string, unknown>,
  userId?: string
) {
  prisma.systemLog
    .create({
      data: {
        level,
        category,
        message,
        metadata: metadata as Prisma.InputJsonValue | undefined,
        userId,
      },
    })
    .catch(() => {}); // fire-and-forget — never throws, never blocks
}

export const logger = {
  info:  (cat: string, msg: string, meta?: Record<string, unknown>, uid?: string) => log("info",  cat, msg, meta, uid),
  warn:  (cat: string, msg: string, meta?: Record<string, unknown>, uid?: string) => log("warn",  cat, msg, meta, uid),
  error: (cat: string, msg: string, meta?: Record<string, unknown>, uid?: string) => log("error", cat, msg, meta, uid),
};
```

**Import path note:** `@/app/generated/prisma/client` matches the `output` in your `generator client` block. If your project uses the default location, import from `@prisma/client` instead.

### Usage pattern

```typescript
import { logger } from "@/lib/logger";

// Auth
logger.info("auth", "Login successful", { ip, email }, userId);
logger.warn("auth", "Failed login attempt", { ip, email, reason });
logger.error("auth", "Rate limit exceeded", { ip, windowMs: 60000 });

// Background worker
logger.info("campaign", "Generation started", { campaignId }, userId);
logger.error("ai", "Generation failed", { campaignId, contactId, error: err.message, provider, model }, userId);

// SMTP / email sending
logger.info("smtp", "Email sent", { to, campaignId, messageId }, userId);
logger.error("smtp", "Send failed", { to, error: err.message }, userId);

// API / external requests
logger.warn("api", "Slow response", { endpoint, durationMs: 4200 });
logger.error("api", "External service error", { service: "stripe", status: 503 });
```

---

## Step 3 — API key validation (`lib/api-key.ts`)

```typescript
export function validateApiKey(raw: string): boolean {
  const envKey = process.env.PUBLIC_LOGS_API_KEY?.trim();
  return !!envKey && raw.trim() === envKey;
}
```

Plain string comparison — no hashing, no DB. Simple and correct for a single static key.

---

## Step 4 — Logs endpoint

### Next.js App Router (`app/api/public/logs/route.ts`)

```typescript
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import { prisma } from "@/lib/prisma";
import { validateApiKey } from "@/lib/api-key";

export const runtime = "nodejs";

const querySchema = z.object({
  level:    z.enum(["info", "warn", "error"]).optional(),
  category: z.string().optional(),
  userId:   z.string().optional(),
  from:     z.string().datetime().optional(),
  to:       z.string().datetime().optional(),
  page:     z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(200).default(50),
});

export async function GET(req: NextRequest) {
  const apiKey = req.headers.get("x-api-key");
  if (!apiKey) {
    return NextResponse.json({ error: "Missing X-Api-Key header" }, { status: 401 });
  }
  if (!validateApiKey(apiKey)) {
    return NextResponse.json({ error: "Invalid API key" }, { status: 401 });
  }

  const parsed = querySchema.safeParse(
    Object.fromEntries(req.nextUrl.searchParams)
  );
  if (!parsed.success) {
    return NextResponse.json({ error: parsed.error.flatten() }, { status: 400 });
  }

  const { level, category, userId, from, to, page, pageSize } = parsed.data;

  const where = {
    ...(level    ? { level }    : {}),
    ...(category ? { category } : {}),
    ...(userId   ? { userId }   : {}),
    ...(from || to
      ? { createdAt: { ...(from ? { gte: new Date(from) } : {}), ...(to ? { lte: new Date(to) } : {}) } }
      : {}),
  };

  const [logs, total] = await Promise.all([
    prisma.systemLog.findMany({
      where,
      orderBy: { createdAt: "desc" },
      skip: (page - 1) * pageSize,
      take: pageSize,
    }),
    prisma.systemLog.count({ where }),
  ]);

  return NextResponse.json({
    data: logs,
    pagination: { page, pageSize, total, pages: Math.ceil(total / pageSize) },
  });
}
```

### Express variant

```typescript
import express from "express";
import { z } from "zod";
import { prisma } from "./lib/prisma";
import { validateApiKey } from "./lib/api-key";

const router = express.Router();

const querySchema = z.object({
  level:    z.enum(["info", "warn", "error"]).optional(),
  category: z.string().optional(),
  userId:   z.string().optional(),
  from:     z.string().datetime().optional(),
  to:       z.string().datetime().optional(),
  page:     z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(200).default(50),
});

router.get("/api/public/logs", async (req, res) => {
  const apiKey = req.headers["x-api-key"] as string | undefined;
  if (!apiKey || !validateApiKey(apiKey)) {
    return res.status(401).json({ error: "Invalid or missing X-Api-Key" });
  }

  const parsed = querySchema.safeParse(req.query);
  if (!parsed.success) return res.status(400).json({ error: parsed.error.flatten() });

  const { level, category, userId, from, to, page, pageSize } = parsed.data;
  const where = {
    ...(level    ? { level }    : {}),
    ...(category ? { category } : {}),
    ...(userId   ? { userId }   : {}),
    ...(from || to ? { createdAt: { ...(from ? { gte: new Date(from) } : {}), ...(to ? { lte: new Date(to) } : {}) } } : {}),
  };

  const [data, total] = await Promise.all([
    prisma.systemLog.findMany({ where, orderBy: { createdAt: "desc" }, skip: (page - 1) * pageSize, take: pageSize }),
    prisma.systemLog.count({ where }),
  ]);

  res.json({ data, pagination: { page, pageSize, total, pages: Math.ceil(total / pageSize) } });
});

export default router;
```

---

## Step 5 — Debug endpoint (verify env var is loaded)

**Next.js:** `app/api/public/logs/debug/route.ts`
```typescript
export async function GET() {
  const envKey = process.env.PUBLIC_LOGS_API_KEY;
  return Response.json({
    set: !!envKey,
    length: envKey?.length ?? 0,
    preview: envKey ? `${envKey.slice(0, 4)}...${envKey.slice(-4)}` : null,
  });
}
```

**Express:**
```typescript
app.get("/api/public/logs/debug", (_req, res) => {
  const envKey = process.env.PUBLIC_LOGS_API_KEY;
  res.json({ set: !!envKey, length: envKey?.length ?? 0, preview: envKey ? `${envKey.slice(0, 4)}...${envKey.slice(-4)}` : null });
});
```

**First check after deploy:**
```bash
curl https://your-domain.com/api/public/logs/debug
# {"set":true,...}   ← OK
# {"set":false,...}  ← env var didn't reach the container (see Step 7)
```

---

## Step 6 — Environment variable

Add to `.env` and `.env.example`:

```bash
# .env
PUBLIC_LOGS_API_KEY=your-secret-key-here

# .env.example
PUBLIC_LOGS_API_KEY=   # Generate with: openssl rand -hex 32
```

Generate a secure key:
```bash
openssl rand -hex 32
```

---

## Step 7 — Docker Compose passthrough (CRITICAL)

Environment variables set on the host or platform (EasyPanel, Railway, Render) do NOT reach the container unless they are explicitly declared in `docker-compose.yml`.

```yaml
services:
  app:
    environment:
      DATABASE_URL: ${DATABASE_URL}
      PUBLIC_LOGS_API_KEY: ${PUBLIC_LOGS_API_KEY:-}   # ← add this line

  worker:                                              # if you have a separate worker process
    environment:
      DATABASE_URL: ${DATABASE_URL}
      PUBLIC_LOGS_API_KEY: ${PUBLIC_LOGS_API_KEY:-}
```

After changing `docker-compose.yml`, do a **full rebuild** (not just restart):
- EasyPanel: rebuild, not redeploy
- Docker: `docker compose up -d --build`

---

## Step 8 — Database migration

### Dev (local)
```bash
npx prisma db push
```

### Production (uses `prisma migrate deploy`)

Create `prisma/migrations/YYYYMMDD_add_system_logs/migration.sql`:

```sql
-- CreateEnum
DO $$ BEGIN
  CREATE TYPE "LogLevel" AS ENUM ('info', 'warn', 'error');
EXCEPTION WHEN duplicate_object THEN NULL;
END $$;

-- CreateTable
CREATE TABLE IF NOT EXISTS "SystemLog" (
  "id"        TEXT NOT NULL,
  "level"     "LogLevel" NOT NULL,
  "category"  TEXT NOT NULL,
  "message"   TEXT NOT NULL,
  "metadata"  JSONB,
  "userId"    TEXT,
  "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT "SystemLog_pkey" PRIMARY KEY ("id")
);

CREATE INDEX IF NOT EXISTS "SystemLog_createdAt_idx" ON "SystemLog"("createdAt");
CREATE INDEX IF NOT EXISTS "SystemLog_level_idx"     ON "SystemLog"("level");
CREATE INDEX IF NOT EXISTS "SystemLog_category_idx"  ON "SystemLog"("category");
CREATE INDEX IF NOT EXISTS "SystemLog_userId_idx"    ON "SystemLog"("userId");
```

Idempotent — safe to run multiple times.

---

## Instrumentation reference

| Category   | Events to instrument |
|------------|---------------------|
| `auth`     | login success/failure, rate limit hits, token refresh |
| `campaign` | start, complete, cancelled, no contacts found |
| `ai`       | config resolution failure, per-contact error, service abort |
| `smtp`     | email sent, email failed, SMTP test |
| `api`      | slow responses (>2s), 5xx errors, external service failures |
| `worker`   | job start, job complete, job retry, job dead-lettered |
| `import`   | file uploaded, rows processed, validation errors |
| `system`   | startup, shutdown, config errors |

### Metadata conventions

Keep metadata flat and serializable. Consistent keys = easier Grafana/dashboard queries:

```typescript
// Auth
{ ip: "1.2.3.4", email: "user@x.com", reason: "wrong_password" }

// AI / generation
{ campaignId: "xxx", contactId: "yyy", provider: "anthropic", model: "claude-haiku-4-5", error: "..." }

// SMTP
{ to: "contact@x.com", campaignId: "xxx", messageId: "<abc@mail.x.com>", durationMs: 420 }

// API
{ endpoint: "/api/campaigns", method: "POST", status: 500, durationMs: 8200 }
```

---

## Querying the API

```bash
# All errors
curl -H "X-Api-Key: $KEY" "https://your-domain.com/api/public/logs?level=error"

# AI errors in a time window
curl -H "X-Api-Key: $KEY" \
  "https://your-domain.com/api/public/logs?level=error&category=ai&from=2026-08-17T10:00:00Z"

# All logs for a user
curl -H "X-Api-Key: $KEY" \
  "https://your-domain.com/api/public/logs?userId=clxxx&pageSize=200"
```

PowerShell:
```powershell
$h = @{ "X-Api-Key" = $env:PUBLIC_LOGS_API_KEY }
Invoke-WebRequest "https://your-domain.com/api/public/logs?level=error" -Headers $h | ConvertFrom-Json
```

---

## Deployment checklist

- [ ] `SystemLog` model + `LogLevel` enum added to `schema.prisma`
- [ ] `prisma:generate` run after schema change
- [ ] `lib/logger.ts` created with correct Prisma import path
- [ ] `lib/api-key.ts` created
- [ ] Logs endpoint at `/api/public/logs`
- [ ] Debug endpoint at `/api/public/logs/debug`
- [ ] `PUBLIC_LOGS_API_KEY` in `.env` and `.env.example`
- [ ] `PUBLIC_LOGS_API_KEY` declared in `docker-compose.yml`
- [ ] Migration SQL file created for production
- [ ] Full container rebuild done (not just restart)
- [ ] Debug endpoint returns `{"set":true}` ✓
- [ ] Test query returns `200` with `data` array ✓
- [ ] At least one `logger.*` call added per major subsystem
