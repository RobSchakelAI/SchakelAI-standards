# B2B SaaS Infrastructure Skills Document

> **Doel:** Dit document is een volledige blauwdruk om een simpele applicatie om te bouwen naar een productie-ready multi-tenant B2B SaaS. Het bevat alle code, SQL, configuratie, en architectuurbeslissingen die je nodig hebt — zonder iets te hoeven vragen.
>
> **Bron:** Productie-gevalideerde patronen uit meerdere B2B SaaS implementaties.
>
> **Stack:** Node.js + Express, React 18 + TypeScript, PostgreSQL (Supabase/Neon), Stripe, Zod, TailwindCSS + shadcn/ui
>
> **Versie:** 2.8

### Taal & naamgeving

- **Code** (variabelen, functies, types): **Engels**. `tenantId`, `getItem()`, `SessionUser`.
- **Error messages en UI teksten**: **Taal van je doelgroep**. Dit document toont Nederlandse voorbeelden ("Niet geauthenticeerd"), pas aan voor jouw markt.
- **Comments in code**: Engels (zodat elke developer het leest, ook niet-Nederlandstaligen).
- **Database kolommen**: Engels, snake_case. `tenant_id`, `created_at`, `password_hash`.

---

## Hoe dit document te gebruiken

### Greenfield vs. Brownfield

Dit document werkt voor twee scenario's:
- **Greenfield** (nieuw project): Volg de implementatie-checklist (§27) van boven naar beneden. Kopieer de code.
- **Brownfield** (bestaand project): Je hebt al werkende code. Niet alles hoeft vervangen te worden.

### Principes vs. Implementatie

Elke sectie bevat optioneel een blok:

> **Principes (niet-onderhandelbaar)**

Dit zijn invarianten die **altijd** moeten gelden, ongeacht je implementatie. Ze beschermen de veiligheid en integriteit van het systeem. Alles buiten een Principes-blok is een **referentie-implementatie** — één bewezen manier om het principe te realiseren.

**De vuistregel:**
- Lees eerst het Principes-blok van een sectie
- Check of jouw bestaande code aan **alle** principes voldoet
- **Ja** → Houd je eigen code. Ga naar de volgende sectie.
- **Deels** → Upgrade alleen de onderdelen die niet voldoen.
- **Nee** → Vervang met de referentie-implementatie.

### Voorbeeld

Je hebt al een uitnodigingssysteem. §13 zegt:

> Tokens worden met bcrypt gehasht opgeslagen — NOOIT plain text in de database.

Jouw systeem slaat tokens plain text op maar werkt verder goed. **Upgrade alleen de token-opslag** — vervang niet het hele systeem.

### Leesroute

1. Lees eerst **Visie & Architectuur** (user journey, hoe alles samenhangt)
2. Implementeer in de volgorde van de **Implementatie Checklist** (§27)
3. Gebruik de **Principes-blokken** als guardrails bij bestaande code

### Brownfield Audit Checklist

Heb je al een werkende applicatie? Draai deze audit **voordat** je begint. Per item weet je of je moet upgraden of kunt overslaan.

```bash
# === 1. Multi-tenancy status ===
# Heb je een tenants tabel?
grep -ri "tenants" migrations/ shared/schema.ts
# Heeft elke domein-tabel een tenant_id kolom?
grep -ri "tenant_id" migrations/ shared/schema.ts

# === 2. Auth model ===
# Gebruik je JWT in localStorage? → VERVANG (§7 Principes)
grep -ri "localStorage.*token\|jwt\|jsonwebtoken" client/src/ server/
# Gebruik je session-based auth? → Check §7 voor CSRF
grep -ri "express-session\|connect-pg-simple\|passport" server/

# === 3. Wachtwoord opslag ===
# Gebruik je bcrypt? → Goed. Check rounds (minimaal 12)
grep -ri "bcrypt\|argon2\|scrypt" server/
# Staat er ergens plain text? → VERVANG (§9 Principes)
grep -ri "password.*=.*req.body\|password.*text" server/

# === 4. Invitation tokens ===
# Zijn tokens gehasht opgeslagen? → §13 Principes
grep -ri "token_hash\|tokenHash" server/ shared/
# Staan tokens plain text in de database? → UPGRADE token-opslag

# === 5. Tenant isolatie ===
# Zijn storage calls tenant-scoped? → §5
grep -rn "tenantId?" server/storage.ts server/supabase-storage.ts
# Elke `tenantId?` (optioneel) is een potentieel security risk

# === 6. RLS ===
# Gebruik je Row Level Security?
grep -ri "ENABLE ROW LEVEL\|FORCE ROW LEVEL\|set_tenant_context" migrations/
```

**Interpretatie:**

| Resultaat | Actie |
|-----------|-------|
| Geen `tenants` tabel | Begin bij §2 (schema) + §5 (multi-tenancy). Zie ook §2 "Bestaande database migreren". |
| JWT in localStorage | Vervang volledig met session-based auth (§7). Geen halve maatregelen. |
| `tenantId?` (optioneel) in storage | Upgrade naar `tenantId: string \| null` (§5). Elke plek. |
| Tokens plain text in DB | Upgrade alleen token-opslag naar bcrypt-hash (§13). Rest van je systeem kan intact blijven. |
| Geen RLS | Implementeer na app-layer multi-tenancy werkt (§6). RLS is laag 2, niet laag 1. |
| bcrypt met < 12 rounds | Verhoog naar 12. Bestaande hashes werken nog — nieuwe worden sterker. |

### Brownfield Migration Guide

Dit is de uitgebreide gids voor het ombouwen van een bestaand project. Het behandelt de meest voorkomende conflicten en geeft per situatie een concreet transitiepad.

#### JWT naar Session-based Auth (transitiestrategie)

Dit is de meest ingrijpende wijziging. Twee aanpakken:

**Aanpak A: Big bang (aanbevolen voor kleine teams, < 50 actieve users)**

```
1. Bouw de volledige session-auth stack (§7) parallel aan je bestaande JWT code
2. Test de session-auth grondig in een staging omgeving
3. Kies een low-traffic moment (bijv. zondag 03:00)
4. Deploy de nieuwe code
5. Alle bestaande JWT tokens worden ongeldig — users moeten opnieuw inloggen
6. Stuur vooraf een e-mail: "We upgraden de beveiliging. Log na [datum] opnieuw in."
```

**Aanpak B: Dual-auth periode (voor grotere applicaties)**

```typescript
// Middleware die BEIDE auth methoden accepteert tijdens de transitieperiode
async function dualAuthMiddleware(req: Request, res: Response, next: NextFunction) {
  try {
    // Probeer session-auth eerst (nieuwe methode)
    if (req.session?.user) return next();

    // Fallback: JWT (oude methode) — alleen tijdens transitieperiode
    const authHeader = req.headers.authorization;
    if (authHeader?.startsWith("Bearer ")) {
      const decoded = jwt.verify(authHeader.slice(7), process.env.JWT_SECRET!) as { userId: string };
      const user = await storage.getUser(decoded.userId);
      if (user) {
        const userTenants = await storage.getUserTenants(user.id);
        // Bouw SessionUser op (zelfde structuur als in §7 login endpoint):
        req.session.user = {
          id: user.id, username: user.username,
          email: user.email ?? undefined, displayName: user.displayName ?? undefined,
          globalRole: user.globalRole as GlobalRole,
          activeTenantId: userTenants[0]?.tenantId,
          tenants: userTenants.map(ut => ({ id: ut.tenantId, name: ut.tenantName, slug: ut.tenantSlug, role: ut.role as TenantRole })),
          mustChangePassword: user.mustChangePassword === true,
        };
        req.session.save();
      }
      return next();
    }

    return res.status(401).json({ error: "Niet geauthenticeerd" });
  } catch (err) {
    return next(err);  // Doorsturen naar error handler (§18)
  }
}

// Na 2-4 weken: verwijder de JWT fallback en de `jsonwebtoken` dependency
```

**Frontend transitie (JWT → session cookies):**

```typescript
// STAP 1: Vervang alle fetch calls
// OUD (JWT):
const res = await fetch("/api/items", {
  headers: { "Authorization": `Bearer ${localStorage.getItem("token")}` },
});

// NIEUW (session cookies):
const res = await fetch("/api/items", {
  credentials: "include",  // Cookie wordt automatisch meegestuurd
});

// STAP 2: Vervang login response handling
// OUD: localStorage.setItem("token", data.token);
// NIEUW: setCsrfToken(data.csrfToken);

// STAP 3: Verwijder alle JWT-gerelateerde code
// - localStorage.getItem("token") / setItem / removeItem
// - Authorization header toevoegingen
// - jwt-decode imports
// - Token refresh logic

// STAP 4: Verwijder JWT packages
// npm uninstall jsonwebtoken jwt-decode express-jwt passport-jwt
```

#### INTEGER Primary Keys naar UUID (migratiepad)

**Optie A: Behoud INTEGER PKs (aanbevolen als je geen UUID nodig hebt)**

Het document's schema gebruikt UUIDs, maar INTEGER PKs werken net zo goed voor multi-tenancy. De kern is `tenant_id` op elke tabel, niet het PK type. Pas de `pgTable` definities aan:

```typescript
// In plaats van: id: uuid("id").primaryKey().defaultRandom()
// Gebruik: id: serial("id").primaryKey()
import { serial } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: serial("id").primaryKey(),  // Behoudt je bestaande auto-increment
  // ... rest hetzelfde
});
```

**Optie B: Migreer naar UUIDs (als je bijv. API-exposed IDs wilt verbergen)**

```sql
-- STAP 1: Voeg UUID kolom toe (parallel aan bestaand id)
ALTER TABLE users ADD COLUMN uuid_id UUID DEFAULT gen_random_uuid();
UPDATE users SET uuid_id = gen_random_uuid() WHERE uuid_id IS NULL;
ALTER TABLE users ALTER COLUMN uuid_id SET NOT NULL;
CREATE UNIQUE INDEX idx_users_uuid ON users(uuid_id);

-- STAP 2: Update alle foreign keys (per tabel die naar users verwijst)
ALTER TABLE user_tenants ADD COLUMN user_uuid UUID;
UPDATE user_tenants SET user_uuid = (SELECT uuid_id FROM users WHERE users.id = user_tenants.user_id);
-- Herhaal voor elke FK

-- STAP 3: Swap columns (in een maintenance window)
ALTER TABLE users RENAME COLUMN id TO old_id;
ALTER TABLE users RENAME COLUMN uuid_id TO id;
ALTER TABLE users DROP CONSTRAINT users_pkey;
ALTER TABLE users ADD PRIMARY KEY (id);
-- Herhaal FK swaps

-- STAP 4: Verwijder oude kolommen (na verificatie)
ALTER TABLE users DROP COLUMN old_id;
```

**Vuistregel:** Kies optie A tenzij je een specifieke reden hebt voor UUIDs. Het document's code werkt met beide types — je hoeft alleen de schema definities aan te passen.

#### Role systeem migratie (flat → two-level)

```sql
-- STAP 1: Vertaal bestaande rollen naar het new model
-- Jouw bestaand: role = 'admin' | 'user'
-- Nieuw: globalRole = 'superadmin' | 'standard' + user_tenants.role = 'admin' | 'member'

-- Voeg global_role kolom toe
ALTER TABLE users ADD COLUMN global_role TEXT NOT NULL DEFAULT 'standard';

-- Maak je hoofd-admin superadmin (kies bewust WIE dit wordt)
UPDATE users SET global_role = 'superadmin' WHERE email = 'jouw-admin@bedrijf.com';
-- Of: de eerste admin die is aangemaakt
-- UPDATE users SET global_role = 'superadmin' WHERE role = 'admin' ORDER BY created_at LIMIT 1;

-- STAP 2: Bij het aanmaken van user_tenants (§2 stap 4), vertaal de oude rollen:
INSERT INTO user_tenants (user_id, tenant_id, role)
SELECT
  id,
  (SELECT id FROM tenants WHERE slug = 'mijn-bedrijf'),
  CASE
    WHEN role = 'admin' THEN 'admin'   -- Bestaande admins worden tenant-admins
    ELSE 'member'                       -- Alle anderen worden members
  END
FROM users;

-- STAP 3: Verwijder de oude role kolom (na verificatie)
ALTER TABLE users DROP COLUMN role;
```

**Beslisboom:** "Wie wordt superadmin?"
- Superadmin = **platform-beheerder** die ALLE tenants kan zien. Meestal de oprichter/CTO.
- Tenant admin = **organisatie-beheerder** die alleen eigen tenant beheert.
- Als je twijfelt: maak 1 superadmin aan en de rest tenant-admin.

#### Stripe: charges naar subscriptions

Als je al Stripe hebt voor directe charges en nu subscriptions wilt toevoegen:

```typescript
// STAP 1: Behoud je bestaande Stripe webhook handler (rename)
// OUD: app.post("/api/webhook/stripe", handleChargeWebhook);
// NIEUW: app.post("/api/webhook/stripe-charges", handleChargeWebhook);  // Legacy

// STAP 2: Voeg de subscription webhook toe (§15)
app.post("/api/webhook/stripe", handleSubscriptionWebhook);  // Nieuwe handler

// STAP 3: Koppel bestaande Stripe customers aan tenants
// Als je bestaande customers hebt, voeg tenantId toe aan hun metadata:
async function migrateExistingStripeCustomers() {
  const customers = await stripe.customers.list({ limit: 100 });
  for (const customer of customers.data) {
    // Zoek de bijbehorende tenant (via email of een ander veld)
    const user = await storage.getUserByEmail(customer.email!);
    if (!user) continue;
    const tenants = await storage.getUserTenants(user.id);
    if (tenants.length === 0) continue;

    // Koppel customer aan tenant
    await stripe.customers.update(customer.id, {
      metadata: { tenantId: tenants[0].tenantId },
    });

    // Maak een subscription record aan (als ze al betalen)
    await storage.upsertStripeSubscription({
      tenantId: tenants[0].tenantId,
      stripeCustomerId: customer.id,
      status: "active",
      tier: "starter",  // Of bepaal tier op basis van bestaand bedrag
      billingInterval: "monthly",
      cancelAtPeriodEnd: false,
    });
  }
}

// STAP 4: Bestaande charges blijven werken — subscriptions zijn een TOEVOEGING
// Je kunt beide modellen naast elkaar gebruiken via aparte webhook handlers
```

#### ORM adapter (Prisma, Knex, TypeORM → dit document's patronen)

Het document gebruikt `postgres` tagged templates met `withTenant()`. Als je een andere ORM hebt:

**Prisma:**

```typescript
// Prisma's equivalent van withTenant() voor RLS:
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

async function withTenantPrisma<T>(tenantId: string, callback: (tx: any) => Promise<T>): Promise<T> {
  return prisma.$transaction(async (tx) => {
    // Zet tenant context voor RLS
    await tx.$executeRaw`SELECT set_config('app.current_tenant_id', ${tenantId}, true)`;
    return callback(tx);
  });
}

// Gebruik:
async function getItems(tenantId: string) {
  return withTenantPrisma(tenantId, async (tx) => {
    return tx.item.findMany({ orderBy: { createdAt: "desc" } });
    // RLS filtert automatisch op tenant
  });
}
```

**Knex:**

```typescript
async function withTenantKnex<T>(tenantId: string, callback: (trx: Knex.Transaction) => Promise<T>): Promise<T> {
  return knex.transaction(async (trx) => {
    await trx.raw("SELECT set_config('app.current_tenant_id', ?, true)", [tenantId]);
    return callback(trx);
  });
}
```

**Principe:** Ongeacht je ORM moet elke tenant-scoped operatie:
1. Een transactie starten
2. `set_config('app.current_tenant_id', tenantId, true)` uitvoeren
3. De query uitvoeren (RLS filtert automatisch)
4. Transactie committen

De `IStorage` interface (§4) is ORM-agnostisch — je kunt de methoden implementeren met Prisma, Knex, of raw SQL. Het patroon (interface → implementatie) blijft hetzelfde.

#### Frontend router (react-router compatibiliteit)

Het document gebruikt **wouter** maar de patronen werken met elke router. Vertaaltabel:

| wouter | react-router v6 | Uitleg |
|--------|-----------------|--------|
| `import { Switch, Route } from "wouter"` | `import { Routes, Route } from "react-router-dom"` | Container component |
| `<Switch>` | `<Routes>` | Route matching |
| `<Route path="/x" component={X} />` | `<Route path="/x" element={<X />} />` | Route definitie |
| `<Redirect to="/x" />` | `<Navigate to="/x" replace />` | Redirect component |
| `const [location, setLocation] = useLocation()` | `const navigate = useNavigate()` | Programmatische navigatie |
| `setLocation("/dashboard")` | `navigate("/dashboard")` | Navigeer naar route |
| `const params = useParams()` | `const params = useParams()` | Route params (identiek) |

```tsx
// react-router App.tsx equivalent:
import { BrowserRouter, Routes, Route, Navigate } from "react-router-dom";

function App() {
  const { user, isLoading, isAuthenticated } = useAuth();
  if (isLoading) return <LoadingScreen />;

  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/register" element={<Register />} />
        <Route path="/pricing" element={<Pricing />} />
        <Route path="/" element={isAuthenticated ? <Navigate to="/dashboard" /> : <Landing />} />
        <Route path="/dashboard" element={
          !isAuthenticated ? <Navigate to="/login" /> : <Layout><Dashboard /></Layout>
        } />
        <Route path="*" element={<NotFound />} />
      </Routes>
    </BrowserRouter>
  );
}
```

#### Multi-tenancy migratiescript voor N tabellen

Als je veel tabellen hebt, genereer de SQL automatisch:

```sql
-- Genereer ALTER TABLE + backfill SQL voor elke tabel
-- Pas de tabel-lijst aan voor jouw schema

DO $$
DECLARE
  tbl TEXT;
  tables TEXT[] := ARRAY['projects', 'tasks', 'comments', 'files', 'invoices',
                         'documents', 'notifications', 'tags'];  -- PAS AAN
  default_tenant UUID := (SELECT id FROM tenants WHERE slug = 'mijn-bedrijf');
BEGIN
  FOREACH tbl IN ARRAY tables LOOP
    -- 1. Voeg tenant_id kolom toe (als die nog niet bestaat)
    EXECUTE format('ALTER TABLE %I ADD COLUMN IF NOT EXISTS tenant_id UUID REFERENCES tenants(id)', tbl);

    -- 2. Backfill bestaande data naar default tenant
    EXECUTE format('UPDATE %I SET tenant_id = $1 WHERE tenant_id IS NULL', tbl) USING default_tenant;

    -- 3. Maak NOT NULL (alleen als alle rijen een waarde hebben)
    EXECUTE format('ALTER TABLE %I ALTER COLUMN tenant_id SET NOT NULL', tbl);

    -- 4. Maak een index
    EXECUTE format('CREATE INDEX IF NOT EXISTS idx_%I_tenant ON %I(tenant_id)', tbl, tbl);

    RAISE NOTICE 'Migrated table: %', tbl;
  END LOOP;
END $$;

-- Verifieer: geen orphaned data
-- SELECT tablename FROM pg_tables WHERE schemaname = 'public'
-- EXCEPT SELECT table_name FROM information_schema.columns WHERE column_name = 'tenant_id';
```

**RLS policies genereren voor N tabellen:**

```sql
-- Genereer RLS policies voor ALLE tabellen met tenant_id
DO $$
DECLARE
  tbl TEXT;
BEGIN
  FOR tbl IN
    SELECT DISTINCT table_name FROM information_schema.columns
    WHERE column_name = 'tenant_id' AND table_schema = 'public'
  LOOP
    -- Activeer RLS
    EXECUTE format('ALTER TABLE %I ENABLE ROW LEVEL SECURITY', tbl);
    EXECUTE format('ALTER TABLE %I FORCE ROW LEVEL SECURITY', tbl);

    -- Maak policy (skip als al bestaat)
    BEGIN
      EXECUTE format(
        'CREATE POLICY tenant_isolation ON %I FOR ALL USING (tenant_matches(tenant_id)) WITH CHECK (tenant_matches(tenant_id))',
        tbl
      );
    EXCEPTION WHEN duplicate_object THEN
      RAISE NOTICE 'Policy already exists for %', tbl;
    END;

    RAISE NOTICE 'RLS enabled for: %', tbl;
  END LOOP;
END $$;
```

#### Indirecte tenant relaties (wanneer GEEN tenant_id toevoegen)

Niet elke tabel heeft een directe `tenant_id` nodig. Beslisboom:

| Tabel relatie | Voorbeeld | Strategie |
|---------------|-----------|-----------|
| Direct tenant-owned | `projects` → `tenant_id` | Direct `tenant_id` + RLS policy |
| Kind van tenant-owned | `tasks` → `project_id` → `projects.tenant_id` | **Keuze:** direct `tenant_id` (eenvoudiger) OF indirect via JOIN |
| Kleinkind | `comments` → `task_id` → `tasks` → ... | Direct `tenant_id` toevoegen (vermijd diepe JOINs in RLS) |
| Globaal/gedeeld | `users`, `tenants`, `stripe_subscriptions` | `allow_all` RLS policy, app-layer controle |

**Vuistregel:** Bij meer dan 1 niveau diep → voeg direct `tenant_id` toe. RLS met diepe JOINs is traag en moeilijk te debuggen.

#### File storage migratie

Als je bestaande bestanden hebt zonder tenant prefix:

```bash
# 1. Genereer een lijst van alle bestanden die gemigreerd moeten worden
psql "$DATABASE_URL" -c "SELECT id, file_path FROM files WHERE tenant_id IS NOT NULL" > files_to_migrate.csv

# 2. Kopieer bestanden naar tenant-prefixed paden
# (script afhankelijk van je storage: S3, local disk, Supabase storage)
while IFS=',' read -r id old_path; do
  tenant_id=$(psql -t -c "SELECT tenant_id FROM files WHERE id = '$id'" "$DATABASE_URL" | tr -d ' ')
  new_path="uploads/${tenant_id}/${old_path##*/}"
  # Kopieer bestand (pas aan voor jouw storage)
  cp "$old_path" "$new_path"  # Of: aws s3 cp, supabase storage move, etc.
  # Update database
  psql "$DATABASE_URL" -c "UPDATE files SET file_path = '$new_path' WHERE id = '$id'"
done < files_to_migrate.csv
```

**Principe:** Na migratie moeten ALLE file paden het formaat `uploads/{tenantId}/...` volgen. Test met een script dat verifieert dat elk bestand een tenant prefix heeft.

#### Brownfield implementatievolgorde

Voor brownfield projecten is de volgorde ANDERS dan voor greenfield (§27):

```
Fase 0: Voorbereiding
  └── Database backup maken (VERPLICHT voor brownfield)
  └── Brownfield Audit Checklist draaien
  └── Bepaal: INTEGER of UUID PKs (behoud of migreer)
  └── Bepaal: ORM behouden of wisselen

Fase 1: Database multi-tenancy (GEEN code wijzigingen nog)
  └── tenants tabel + default tenant aanmaken (§2)
  └── user_tenants koppeltabel + role mapping
  └── tenant_id toevoegen aan alle domein-tabellen (script hierboven)
  └── Verifieer: geen orphaned data

Fase 2: Auth migratie
  └── Session store installeren (express-session + connect-pg-simple)
  └── Dual-auth middleware (JWT fallback) OF big bang
  └── CSRF middleware toevoegen
  └── Frontend: credentials: "include" + CSRF token handling
  └── Na 2-4 weken: JWT fallback verwijderen

Fase 3: Storage interface
  └── IStorage interface definiëren (§4)
  └── Bestaande queries migreren naar storage methoden
  └── withTenant() implementeren (§6) — ENABLE_RLS nog op false

Fase 4: RLS activeren
  └── SQL functies + policies uitvoeren (§6)
  └── ENABLE_RLS=true
  └── Verificatietests draaien

Fase 5: Billing + overig
  └── Stripe subscriptions toevoegen (§15)
  └── SubscriptionGuard (§16)
  └── GDPR, MFA, etc.
```

**Terugdraaien bij problemen:**
- RLS: `ALTER TABLE items DISABLE ROW LEVEL SECURITY;` + `ENABLE_RLS=false`
- Auth: schakel terug naar JWT-only (verwijder session middleware)
- tenant_id kolommen: `ALTER TABLE items DROP COLUMN tenant_id;` (destructief — alleen als backup beschikbaar)

---

## Visie & Architectuur

### Het doel

Een SaaS-applicatie die we veilig en eenvoudig wereldwijd voor organisaties kunnen inzetten. Intuïtief voor gebruikers, effortless voor beheerders. Gedeployed op een modulaire stack, op de meest robuuste manier. Zodat we modern zijn, veilig zijn, en onafhankelijk van één of twee providers.

Dit document beschrijft niet 27 losse technieken. Het beschrijft **één samenhangend systeem** dat een simpele applicatie transformeert naar een schaalbaar, veilig, multi-tenant SaaS-platform. Elk onderdeel versterkt de andere.

### Hoe alles samenhangt

```
┌───────────────────────────────────────────────────────────┐
│                    GEBRUIKER                               │
│  Bezoeker → Pricing → Checkout → Onboarding → Dashboard   │
└────────┬──────────────────────────────────────────────────┘
         │
┌────────▼──────────────────────────────────────────────────┐
│                  FRONTEND LAAG                             │
│  React + wouter + TanStack Query + shadcn/ui              │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────┐   │
│  │ AuthProvider │ │ QueryClient  │ │ SubscriptionGuard│   │
│  │ (sessie)    │ │ (CSRF sync)  │ │ (toegangsctrl)   │   │
│  └─────────────┘ └──────────────┘ └──────────────────┘   │
└────────┬──────────────────────────────────────────────────┘
         │ credentials: "include" + X-CSRF-Token
┌────────▼──────────────────────────────────────────────────┐
│                  API LAAG                                   │
│  Express + Session + CSRF + Rate Limiting                  │
│  ┌───────────┐ ┌──────────┐ ┌─────────┐ ┌────────────┐   │
│  │ Auth      │ │ Webhooks │ │ Billing │ │ Domein-API  │   │
│  │ (login,   │ │ (Stripe, │ │ (check- │ │ (jouw CRUD  │   │
│  │  MFA,     │ │  extern) │ │  out,   │ │  endpoints) │   │
│  │  CSRF)    │ │          │ │  portal)│ │             │   │
│  └───────────┘ └──────────┘ └─────────┘ └────────────┘   │
└────────┬──────────────────────────────────────────────────┘
         │ tenantId: string | null
┌────────▼──────────────────────────────────────────────────┐
│                  DATA LAAG                                  │
│  PostgreSQL + RLS + Versleuteling                          │
│  ┌──────────────┐ ┌─────────────────┐ ┌──────────────┐   │
│  │ Storage      │ │ Row Level       │ │ Audit Log    │   │
│  │ Interface    │ │ Security        │ │ (nooit       │   │
│  │ (withTenant) │ │ (fail-closed)   │ │  blokkeren)  │   │
│  └──────────────┘ └─────────────────┘ └──────────────┘   │
└───────────────────────────────────────────────────────────┘
```

### De user journey

Dit is het verhaal dat alle secties samenbrengt — van eerste bezoek tot dagelijks gebruik:

1. **Bezoeker landt** op de marketingpagina (publieke routes, geen auth nodig)
2. **Kiest een plan** op de pricing pagina → Stripe checkout sessie (§15)
3. **Stripe webhook** ontvangt betaling → tenant wordt automatisch aangemaakt (§14) → welkomstmail (§20)
4. **Eerste login** met wachtwoord → sessie aangemaakt (§7) → CSRF token gegenereerd
5. **MFA check** als de tenant dit vereist → 6-cijferige code per email (§10)
6. **Dashboard** verschijnt → standaard categorieën en instellingen al aanwezig (§14)
7. **Admin nodigt teamleden uit** → bcrypt-hashed token verstuurd per email (§13)
8. **Teamlid accepteert uitnodiging** → account aangemaakt, gekoppeld aan tenant (§13)
9. **Dagelijks gebruik** → alle data gefilterd op tenant (§5) + RLS als vangnet (§6)
10. **Tier limieten** bewaken gebruik per abonnement (§17)
11. **Subscription verloopt** → grace period → waarschuwing → blokkade via SubscriptionGuard (§16)
12. **Support nodig?** → Superadmin impersoneert gebruiker met volledige audit trail (§12)
13. **Gebruiker vertrekt** → GDPR data-export of 2-staps account-verwijdering (§23)

### Modulaire stack

Elk bouwblok is bewust vervangbaar. De **patronen** zijn universeel, de tools zijn keuzes:

| Bouwblok | Huidige keuze | Alternatieven | Waarom vervangbaar |
|----------|--------------|---------------|-------------------|
| Frontend | React + Vite | Vue, Svelte, Next.js | Alleen de auth hook en API calls raken de backend |
| Backend | Express | Fastify, Hono, NestJS | Storage interface abstraheert de database |
| Database | PostgreSQL (Supabase) | Neon, CockroachDB, zelf-gehost | RLS is standaard PostgreSQL, niet Supabase-specifiek |
| Hosting frontend | Vercel | Netlify, Cloudflare Pages | Statische build, geen vendor lock-in |
| Hosting backend | Railway | Render, Fly.io, AWS ECS | Docker container, standaard Node.js |
| Betalingen | Stripe | Mollie, Paddle, Adyen | Webhook-patroon is universeel |
| Email | MailerSend | Resend, Postmark, SendGrid | Service achter een interface |
| LLM | Claude (Anthropic) | GPT, Gemini, Llama | API call achter een service layer |
| CI/CD | GitHub Actions | GitLab CI, Bitbucket Pipelines | Standaard npm scripts |
| IDE/Agent | Claude Code | Cursor, Copilot, Windsurf | CLAUDE.md werkt met elke AI-assistent |

**Het kernprincipe:** Geen enkele dependency mag zo diep verweven zijn dat vervanging een herschrijving vereist. Services zitten achter interfaces. Database-queries zitten achter een storage layer. Externe API-calls zitten achter service classes.

---

## Inhoudsopgave

**Deel A: Fundament**
1. [Dependencies & Project Setup](01-project-setup.md)
2. [Database Schema (alle tabellen)](02-database-schema.md)
3. [Shared TypeScript Types](03-shared-types.md)
4. [Storage Interface Pattern](04-storage-interface.md)
5. [Multi-Tenancy (App Layer)](05-multi-tenancy.md)
6. [Row Level Security (Database Layer)](05-multi-tenancy.md#6-row-level-security-database-layer)

**Deel B: Authenticatie & Security**
7. [Session Auth + CSRF (complete flow)](06-auth-security.md)
7b. [Registration & Signup Flow](06-auth-security.md#7b-registration--signup-flow)
8. [Frontend Auth (useAuth + QueryClient + Page Templates)](06-auth-security.md#8-frontend-auth)
9. [Password Security & Password Reset](06-auth-security.md#9-password-security--password-reset)
10. [Email-Based MFA](06-auth-security.md#10-email-based-mfa)
11. [Rate Limiting](06-auth-security.md#11-rate-limiting)
12. [Superadmin Impersonation](06-auth-security.md#12-superadmin-impersonation)

**Deel C: Billing & Onboarding**
13. [Invitation System](07-billing-onboarding.md)
14. [Tenant Provisioning & Bootstrap](07-billing-onboarding.md#14-tenant-provisioning--bootstrap)
15. [Stripe Billing (checkout, webhook, portal)](07-billing-onboarding.md#15-stripe-billing)
16. [Subscription Access Control](07-billing-onboarding.md#16-subscription-access-control)
17. [Tier Limits](07-billing-onboarding.md#17-tier-limits)

**Deel D: Platform Kwaliteit**
18. [Centralized Error Handling](08-platform-quality.md)
19. [Audit Logging](08-platform-quality.md#19-audit-logging)
20. [Email Service](08-platform-quality.md#20-email-service)
21. [OAuth Token Encryption](08-platform-quality.md#21-oauth-token-encryption)
22. [Webhook Handling](08-platform-quality.md#22-webhook-handling)
23. [GDPR Compliance](08-platform-quality.md#23-gdpr-compliance)
24. [Database-Backed Queue Processing](08-platform-quality.md#24-database-backed-queue-processing)

**Deel E: Build & Deploy**
25. [Dev & Production Server Setup](09-build-deploy.md)
26. [Build & Deploy Configuration](09-build-deploy.md#26-build--deploy-configuration)
27. [Implementatie Checklist](09-build-deploy.md#27-implementatie-checklist)

