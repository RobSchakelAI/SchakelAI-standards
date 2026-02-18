# §25-27 Build & Deploy

> Onderdeel van de B2B SaaS Infrastructure Skills — Deel E: Build & Deploy
>
> Zie [skills/README.md](README.md) voor het overzicht van alle secties.

## 25. Dev & Production Server Setup

### Development server (server/index-dev.ts)

In development draait Vite als middleware IN Express. Geen aparte proxy configuratie nodig.

```typescript
// server/index-dev.ts
import fs from "fs";
import path from "path";
import express from "express";
import { createServer as createViteServer } from "vite";
import { app } from "./app";
import { registerRoutes } from "./routes";

async function startDev() {
  // 1. Registreer API routes
  const server = await registerRoutes(app);

  // 2. Maak Vite dev server in middleware mode
  const vite = await createViteServer({
    server: { middlewareMode: true },
    appType: "custom",
  });

  // 3. Mount Vite middleware (HMR, file serving, transforms)
  app.use(vite.middlewares);

  // 4. Fallback: alle niet-API routes → index.html (SPA)
  app.use("*", async (req, res, next) => {
    if (req.originalUrl.startsWith("/api")) return next();
    try {
      let html = fs.readFileSync(path.resolve("client/index.html"), "utf-8");
      html = await vite.transformIndexHtml(req.originalUrl, html);
      res.status(200).set({ "Content-Type": "text/html" }).end(html);
    } catch (e) {
      next(e);
    }
  });

  const port = process.env.PORT || 5000;
  server.listen(port, () => console.log(`Dev server: http://localhost:${port}`));
}

startDev();
```

**Hoe het werkt:** Vite middleware onderschept `/src/**` requests voor HMR. API calls (`/api/**`) gaan direct naar Express routes. Geen proxy config nodig — alles draait op dezelfde poort.

### Production server (server/index.ts)

In productie serveert Express de statische Vite build.

```typescript
// server/index.ts (of index-prod.ts)
import express from "express";
import path from "path";
import { type Server } from "http";
import { app } from "./app";
import { registerRoutes } from "./routes";
import { storage } from "./storage";
import { validateEnvironment } from "./utils/env";  // Zie §26

validateEnvironment();

async function startProd() {
  const server = await registerRoutes(app);

  // Serve Vite-gebouwde frontend
  const publicDir = path.resolve("dist/public");
  app.use(express.static(publicDir));

  // SPA fallback: alle niet-API routes → index.html
  app.get("*", (req, res) => {
    if (req.originalUrl.startsWith("/api")) return res.status(404).json({ error: "Not found" });
    res.sendFile(path.join(publicDir, "index.html"));
  });

  const port = process.env.PORT || 5000;
  server.listen(port, () => console.log(`Production server on port ${port}`));

  // Graceful shutdown — VERPLICHT voor Railway/container deploys
  setupGracefulShutdown(server);
}

function setupGracefulShutdown(server: Server): void {
  let isShuttingDown = false;

  async function shutdown(signal: string) {
    if (isShuttingDown) return;
    isShuttingDown = true;
    console.log(`${signal} received — starting graceful shutdown`);

    // Force exit na 30 seconden (voorkom hanging)
    const forceTimer = setTimeout(() => {
      console.error("Graceful shutdown timed out — forcing exit");
      process.exit(1);
    }, 30_000);
    forceTimer.unref();

    try {
      server.close();                // 1. Stop accepting connections
      await storage.close();         // 2. Close database pool
      console.log("Graceful shutdown complete");
      process.exit(0);
    } catch (err) {
      console.error("Error during shutdown:", err);
      process.exit(1);
    }
  }

  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT", () => shutdown("SIGINT"));
}

startProd();
```

### Scripts in package.json

```json
{
  "scripts": {
    "dev": "NODE_ENV=development tsx server/index-dev.ts",
    "build": "vite build && esbuild server/index.ts --platform=node --packages=external --bundle --format=esm --outfile=dist/index.js",
    "start": "NODE_ENV=production node dist/index.js"
  }
}
```

**KRITIEK: `--packages=external`** — Dit voorkomt dat native modules (bcrypt, postgres) gebundeld worden. Zonder deze flag crasht de build op C++ bindings.

### Migraties uitvoeren in productie

SQL migraties (schema wijzigingen, RLS policies) moeten **apart** van de app deploy worden uitgevoerd. Drie methoden:

**Methode 1: Supabase SQL Editor (aanbevolen voor start)**
```
1. Open Supabase Dashboard → SQL Editor
2. Plak de migratie SQL (bijv. de RLS policies uit §6)
3. Klik "Run"
4. Verifieer: \dt of SELECT * FROM pg_policies;
```

**Methode 2: psql via connection string**
```bash
# Voer een migration file uit
psql "$DATABASE_URL" -f migrations/005_rls_tenant_isolation.sql

# Of interactief
psql "$DATABASE_URL"
\i migrations/005_rls_tenant_isolation.sql
```

**Methode 3: Drizzle Kit (voor schema-migraties, NIET voor RLS)**
```bash
# Genereer migratie vanuit schema.ts wijzigingen
npx drizzle-kit generate

# Pas toe op productie database
DATABASE_URL="postgresql://..." npx drizzle-kit migrate
```

**Belangrijk:** RLS policies (`CREATE POLICY`, `ENABLE ROW LEVEL SECURITY`) worden NIET door Drizzle beheerd. Die staan in handgeschreven `.sql` bestanden en moeten via methode 1 of 2 worden uitgevoerd.

**Volgorde bij eerste setup:**
```
1. npx drizzle-kit push (of migrate) — schema tabellen aanmaken
2. psql -f migrations/005_rls.sql — RLS functies + policies
3. npx tsx server/seed.ts — eerste superadmin + tenant
4. ENABLE_RLS=true in env vars zetten
5. Deploy de applicatie
```

---

## 26. Build & Deploy Configuration

### Startup validatie (crash early, crash loud)

Voeg dit toe aan je server entry point (`server/index.ts`). Voorkomt cryptische runtime errors door missende env vars.

```typescript
// server/utils/env.ts
function requireEnv(name: string): string {
  const value = process.env[name];
  if (!value) {
    console.error(`FATAL: Environment variable ${name} is not set`);
    process.exit(1);
  }
  return value;
}

// Verplicht bij startup
export function validateEnvironment() {
  requireEnv("DATABASE_URL");
  requireEnv("SESSION_SECRET");

  if (process.env.NODE_ENV === "production") {
    requireEnv("ENCRYPTION_KEY");
    requireEnv("FRONTEND_URL");

    // Waarschuw (maar crash niet) bij optionele vars
    if (!process.env.STRIPE_SECRET_KEY) console.warn("WARN: STRIPE_SECRET_KEY not set — billing disabled");
    if (!process.env.MAILERSEND_API_KEY) console.warn("WARN: MAILERSEND_API_KEY not set — emails disabled");
  }
}

// Roep aan in server/index.ts:
// validateEnvironment();
```

### Health check endpoint

Load balancers (Railway, AWS ALB) hebben een health check nodig. Voeg toe aan je routes (publiek, geen auth):

```typescript
// Registreer VOOR alle andere routes (in registerRoutes())
app.get("/api/health", (req, res) => {
  res.json({ status: "ok", timestamp: new Date().toISOString() });
});

// Optioneel: deep health check (database connectivity)
// sql = postgres client instantie, bijv: import { storage } from "./storage";
app.get("/api/health/ready", async (req, res) => {
  try {
    await storage.healthCheck();  // Implementeer als: async healthCheck() { await this.sql`SELECT 1`; }
    res.json({ status: "ready", database: "connected" });
  } catch {
    res.status(503).json({ status: "unhealthy", database: "disconnected" });
  }
});
```

### Environment Variables (compleet)
```bash
# === CORE (verplicht) ===
DATABASE_URL=postgresql://user:pass@host:5432/db
SESSION_SECRET=minimaal-16-tekens-unieke-string
ENCRYPTION_KEY=andere-unieke-string-minimaal-16-tekens     # ANDERS dan SESSION_SECRET!
FRONTEND_URL=https://app.yourdomain.com
CORS_ORIGIN=https://app.yourdomain.com
NODE_ENV=production
PORT=5000

# === STRIPE (verplicht voor billing) ===
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRICE_STARTER_MONTHLY=price_...
STRIPE_PRICE_STARTER_ANNUAL=price_...
STRIPE_PRICE_PRO_MONTHLY=price_...
STRIPE_PRICE_PRO_ANNUAL=price_...

# === EMAIL (verplicht voor MFA + uitnodigingen) ===
MAILERSEND_API_KEY=...
MAILERSEND_FROM_EMAIL=noreply@yourdomain.com

# === OPTIONEEL ===
ENABLE_RLS=true                        # Activeer na RLS migration
DB_POOL_MAX=10                         # Database connection pool size (default: 10)
ENCRYPTION_SALT=my-random-salt-string  # KDF salt — stel in bij nieuwe installaties (zie §21)
ADMIN_PASSWORD=VeiligWachtwoord123     # Vereist in productie voor seed script (§14)
ADMIN_EMAIL=admin@yourdomain.com       # Superadmin email voor seed script (§14)
VITE_API_URL=https://api.yourdomain.com  # Alleen voor cross-origin deploys (Model B)
SUPABASE_URL=https://xxx.supabase.co   # Als je Supabase file storage gebruikt
SUPABASE_SERVICE_KEY=eyJ...
```

### Deployment architectuur
```
Frontend: Vercel (of Netlify)
  → Bouwt vanuit /client
  → Environment: VITE_API_URL=https://api.yourdomain.com

Backend: Railway (of Render)
  → Bouwt vanuit / (root)
  → npm run build && npm start
  → Alle env vars hierboven

Database: Supabase (of Neon)
  → PostgreSQL met RLS
  → Connection string in DATABASE_URL

Stripe Webhooks:
  → URL: https://api.yourdomain.com/api/webhook/stripe
  → Events: customer.subscription.created, updated, deleted
```

### Same-origin vs. Cross-origin deployment

Dit document ondersteunt twee modellen. Kies één en wees consistent:

| | Same-origin (aanbevolen start) | Cross-origin (voor schaalbaarheid) |
|---|---|---|
| **Hoe** | Backend serveert frontend (`npm run build && npm start`, §25) | Vercel (frontend) + Railway (backend) apart |
| **URL** | `app.yourdomain.com` (alles) | `app.yourdomain.com` + `api.yourdomain.com` |
| **sameSite** | `"lax"` | `"none"` + `secure: true` |
| **CORS** | Niet nodig (zelfde origin) | `CORS_ORIGIN` verplicht |
| **Frontend API calls** | Relatieve URLs: `/api/items` | Absolute URLs: `${VITE_API_URL}/api/items` |
| **Wanneer** | MVP, kleine teams | Gescheiden scaling, CDN voor frontend |

**Bij same-origin:** Verwijder de CORS middleware uit `server/app.ts` en verander `sameSite: "none"` naar `"lax"` in §7.

**Bij cross-origin:** Voeg `VITE_API_URL` toe aan frontend env vars en pas `queryClient.ts` aan:
```typescript
// client/src/lib/queryClient.ts
const API_BASE = import.meta.env.VITE_API_URL || "";  // Leeg = same-origin

export async function apiRequest(method: string, url: string, data?: unknown): Promise<Response> {
  // Prefix alle /api URLs met API_BASE
  const fullUrl = url.startsWith("/api") ? `${API_BASE}${url}` : url;
  // ... rest van de functie
}
```

### Structured logging (server/utils/logger.ts)

Console.log werkt voor MVP. Voor productie-monitoring (Railway logs, Datadog, etc.) gebruik structured JSON logging:

```typescript
// server/utils/logger.ts
type LogLevel = "info" | "warn" | "error" | "debug";

function log(level: LogLevel, message: string, context?: Record<string, unknown>) {
  const entry = {
    level,
    message,
    timestamp: new Date().toISOString(),
    ...(context || {}),
  };

  if (process.env.NODE_ENV === "production") {
    // JSON output — leesbaar door Railway, Datadog, ELK, etc.
    console[level === "error" ? "error" : level === "warn" ? "warn" : "log"](JSON.stringify(entry));
  } else {
    // Menselijke output in development
    const prefix = { info: "ℹ️", warn: "⚠️", error: "❌", debug: "🔍" }[level];
    console[level === "error" ? "error" : "log"](`${prefix} ${message}`, context || "");
  }
}

export const logger = {
  info: (msg: string, ctx?: Record<string, unknown>) => log("info", msg, ctx),
  warn: (msg: string, ctx?: Record<string, unknown>) => log("warn", msg, ctx),
  error: (msg: string, ctx?: Record<string, unknown>) => log("error", msg, ctx),
  debug: (msg: string, ctx?: Record<string, unknown>) => {
    if (process.env.NODE_ENV !== "production") log("debug", msg, ctx);
  },
};

// Gebruik: logger.info("Item created", { itemId, tenantId });
// Output (prod): {"level":"info","message":"Item created","itemId":"...","tenantId":"...","timestamp":"..."}
```

### CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"
      - run: npm ci
      - run: npm run check          # TypeScript type checking
      - run: npm test                # Vitest tests
      - run: npm run build           # Verify production build

  # Optioneel: deploy naar Railway/Vercel na merge naar main
  # deploy:
  #   needs: check
  #   if: github.ref == 'refs/heads/main'
  #   runs-on: ubuntu-latest
  #   steps:
  #     - uses: actions/checkout@v4
  #     - run: railway up   # Of: vercel --prod
```

**Minimale CI pipeline**: `npm ci → npm run check → npm test → npm run build`. Als een van deze stappen faalt, wordt de merge geblokkeerd (via GitHub branch protection rules).

---

## 27. Implementatie Checklist

### Brownfield? Start hier:
- [ ] **Draai de Brownfield Audit Checklist** (§Hoe dit document te gebruiken) — bepaal wat je al hebt
- [ ] **Migreer je database** als je nog geen tenants/user_tenants hebt (§2 "Bestaande database migreren")
- [ ] Sla fases over waar je audit "Ja" antwoordt — upgrade alleen wat niet voldoet aan Principes-blokken

### Fase 0: Project Setup (dag 0)
- [ ] Dependencies installeren inclusief devDependencies/@types (§1)
- [ ] `"type": "module"` in package.json
- [ ] TypeScript configuratie (tsconfig.json) met path aliases (§1)
- [ ] Tailwind + PostCSS + shadcn/ui initialiseren (§1)
- [ ] **Frontend entry bestanden aanmaken:** `client/index.html`, `client/src/main.tsx`, `client/src/index.css` (§1)
- [ ] Dev & production server configureren (§25)
- [ ] Build pipeline testen: `npm run dev` + `npm run build` + `npm start` (§25-26)
- [ ] Database aanmaken en connectie testen
- [ ] Drizzle migration workflow kiezen: `db:push` (dev) of `generate` + `migrate` (prod) (§1)
- [ ] Environment variabelen validatie toevoegen (§26) — crash bij startup als KRITIEKE vars missen

### Fase 1: Fundament (dag 1-2)
- [ ] Database schema aanmaken — alle tabellen (§2)
- [ ] Shared types definiëren met Express session types (§3)
- [ ] **Gedeelde utilities aanmaken:** `validateToken()` (§9), `encrypt()`/`decrypt()` (§21), `ApiError`/`ErrorCodes` (§18)
- [ ] Storage interface + MemStorage (§4)
- [ ] SupabaseStorage implementeren met `withTenant()` (§4, §6) — `ENABLE_RLS` nog op false
- [ ] Multi-tenancy in storage layer (§5)
- [ ] **Rate limiter stubs:** maak `loginLimiter`/`publicLimiter` aan als no-op passthrough (§11 volledige config in Fase 2)
- [ ] Session auth + CSRF + login/session/logout/change-password endpoints (§7) — **let op middleware volgorde!**
- [ ] Registratie endpoint voor self-service onboarding (§7b)
- [ ] Frontend auth hook + QueryClient + protected routes (§8)
- [ ] **Frontend pagina's:** Login, Register, Dashboard, Settings, Landing, Pricing, AcceptInvitation, ForgotPassword, ResetPassword, ChangePassword, NotFound (§8)
- [ ] **Stub component:** `SubscriptionGuard` — return `<>{children}</>` (volledig in Fase 3, §16)
- [ ] Error handling met asyncHandler op elke async route (§18)
- [ ] Email service basis (§20)
- [ ] **Tenant provisioning service** (§14) — nodig voor registratie (§7b) en Stripe checkout (§15)
- [ ] **Seed script draaien** voor eerste superadmin + tenant (§14)

### Fase 2: Security (dag 3-4)
- [ ] RLS SQL functies uitvoeren — `set_tenant_context`, `get_current_tenant_id` (§6 stap 1)
- [ ] RLS policies per tabel uitvoeren (§6 stap 2) — via psql of Supabase SQL Editor (§25)
- [ ] `ENABLE_RLS=true` zetten in environment
- [ ] **Verificatietests draaien** (§6) — tenant isolation + RLS + CSRF tests
- [ ] Password security + password reset flow (§9) — **gebruikt validateToken() uit Fase 1**
- [ ] Rate limiting inclusief MFA limiter (§11)
- [ ] Account lockout configureren (§11)
- [ ] Audit logging (§19)
- [ ] OAuth token encryption (§21) — **gebruikt encrypt/decrypt uit Fase 1**

#### Security Verificatie Checklist (draai na Fase 2)
- [ ] Login → session ID wijzigt (DevTools > Cookies — bevestig `req.session.regenerate()`)
- [ ] 5x fout wachtwoord → account locked (423 status)
- [ ] MFA verify → max 5 pogingen per 15 minuten
- [ ] Tenant switch naar inactieve tenant → 403
- [ ] Absolute session timeout na 30 dagen (check `createdAt` in session)
- [ ] `npm run check` — geen TypeScript errors
- [ ] `npm test` — alle tests groen

### Fase 3: Billing & Onboarding (dag 5-7)
- [ ] Stripe billing setup + price ID mapping (§15)
- [ ] Webhook endpoint (VOOR CSRF middleware!) (§22)
- [ ] SubscriptionGuard frontend (§16)
- [ ] Tier limits (§17)
- [ ] Invitation system (§13) — **gebruikt validateToken() uit Fase 1**
- [ ] Customer portal endpoint (§15)

### Fase 4: Compliance & Polish (week 2)
- [ ] GDPR endpoints — data export, 2-staps verwijdering (§23)
- [ ] Email MFA (§10) — frontend + backend volledig verbonden (§8 + §10)
- [ ] Superadmin impersonation (§12)
- [ ] Webhook signature verificatie + deduplicatie (§22)
- [ ] Cookie consent banner (§23)
- [ ] Welkomstmail na checkout (§20)
- [ ] Health check endpoint toevoegen (`/api/health` — zie hieronder)

### Fase 5: Processing (wanneer nodig)
- [ ] Database-backed queue (§24)
- [ ] Exponential backoff
- [ ] Crash recovery bij startup

### Doorlopend (elke fase)
- [ ] `npm run check` (TypeScript) na elke wijziging
- [ ] Verificatietests draaien na security-gerelateerde wijzigingen (§6)
- [ ] Checklist-items afvinken in dit document

---

> **Versie:** 2.8 — **Laatst bijgewerkt:** 10 februari 2026 — **Changelog:** v2.8 = 16 review issues (SQL CHECK constraints, GDPR email/token fixes, upsert velden, COALESCE NULL-reset, stripeWebhookRouter) | v2.7 = 20 review issues (GDPR uitgeschreven, signature alignment, session dedup, invitation routes, JobQueue this.sql) | v2.6 = 23 interne inconsistenties opgelost (SupabaseStorage ↔ IStorage sync, ontbrekende endpoints, type fixes) | v2.5 = 30 review issues opgelost | v2.4 = domain-agnostisch (meetings→items) | v2.3 = agent-review gaps | v2.2 = 28 gaps
> **Bron:** Productie-gevalideerde patronen uit meerdere B2B SaaS implementaties
