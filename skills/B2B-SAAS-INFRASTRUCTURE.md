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
1. [Dependencies & Project Setup](#1-dependencies--project-setup)
2. [Database Schema (alle tabellen)](#2-database-schema)
3. [Shared TypeScript Types](#3-shared-typescript-types)
4. [Storage Interface Pattern](#4-storage-interface-pattern)
5. [Multi-Tenancy (App Layer)](#5-multi-tenancy-app-layer)
6. [Row Level Security (Database Layer)](#6-row-level-security-database-layer)

**Deel B: Authenticatie & Security**
7. [Session Auth + CSRF (complete flow)](#7-session-auth--csrf)
7b. [Registration & Signup Flow](#7b-registration--signup-flow)
8. [Frontend Auth (useAuth + QueryClient + Page Templates)](#8-frontend-auth)
9. [Password Security & Password Reset](#9-password-security--password-reset)
10. [Email-Based MFA](#10-email-based-mfa)
11. [Rate Limiting](#11-rate-limiting)
12. [Superadmin Impersonation](#12-superadmin-impersonation)

**Deel C: Billing & Onboarding**
13. [Invitation System](#13-invitation-system)
14. [Tenant Provisioning & Bootstrap](#14-tenant-provisioning--bootstrap)
15. [Stripe Billing (checkout, webhook, portal)](#15-stripe-billing)
16. [Subscription Access Control](#16-subscription-access-control)
17. [Tier Limits](#17-tier-limits)

**Deel D: Platform Kwaliteit**
18. [Centralized Error Handling](#18-centralized-error-handling)
19. [Audit Logging](#19-audit-logging)
20. [Email Service](#20-email-service)
21. [OAuth Token Encryption](#21-oauth-token-encryption)
22. [Webhook Handling](#22-webhook-handling)
23. [GDPR Compliance](#23-gdpr-compliance)
24. [Database-Backed Queue Processing](#24-database-backed-queue-processing)

**Deel E: Build & Deploy**
25. [Dev & Production Server Setup](#25-dev--production-server-setup)
26. [Build & Deploy Configuration](#26-build--deploy-configuration)
27. [Implementatie Checklist](#27-implementatie-checklist)

---

# DEEL A: FUNDAMENT

---

## 1. Dependencies & Project Setup

### Wanneer toepassen
Altijd. Dit is stap 0 — installeer deze packages voordat je aan iets anders begint.

### Package.json dependencies

```json
{
  "type": "module",
  "dependencies": {
    // === CORE FRAMEWORK ===
    "express": "^4.21.2",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "typescript": "5.6.3",

    // === DATABASE & ORM ===
    "drizzle-orm": "^0.39.1",
    "postgres": "^3.4.7",

    // === SESSION & AUTH ===
    "express-session": "^1.18.1",
    "connect-pg-simple": "^10.0.0",
    "bcrypt": "^6.0.0",
    "express-rate-limit": "^8.2.1",

    // === VALIDATION ===
    "zod": "^3.24.2",
    "drizzle-zod": "^0.7.0",

    // === DATA FETCHING ===
    "@tanstack/react-query": "^5.60.5",

    // === ROUTING (frontend) ===
    "wouter": "^3.3.5",

    // === STRIPE ===
    "stripe": "^20.3.0",

    // === EMAIL ===
    "mailersend": "^2.6.0",

    // === SECURITY ===
    "helmet": "^8.1.0",

    // === SUPABASE (file storage + DB) ===
    "@supabase/supabase-js": "^2.84.0",

    // === DATES ===
    "date-fns": "^3.6.0",
    "date-fns-tz": "^3.2.0",

    // === UI (shadcn/ui + dependencies) ===
    "tailwindcss": "^3.4.17",
    "lucide-react": "^0.453.0",
    "clsx": "^2.1.1",
    "tailwind-merge": "^2.6.0",
    "class-variance-authority": "^0.7.1",
    "@radix-ui/react-dialog": "latest",
    "@radix-ui/react-dropdown-menu": "latest",
    "@radix-ui/react-accordion": "latest"
  },
  "devDependencies": {
    "drizzle-kit": "^0.31.4",
    "vite": "^5.4.20",
    "@vitejs/plugin-react": "^4.7.0",
    "esbuild": "^0.25.0",
    "tsx": "^4.20.5",
    "vitest": "^4.0.18",
    "tailwindcss-animate": "^1.0.7",
    "autoprefixer": "^10.4.20",
    "postcss": "^8.4.49",

    // === TYPE DEFINITIONS (verplicht voor TypeScript) ===
    "@types/express": "^5.0.2",
    "@types/express-session": "^1.18.1",
    "@types/connect-pg-simple": "^7.0.3",
    "@types/bcrypt": "^5.0.2",
    "@types/node": "^22.15.0"
  }
}
```

### Project structuur
```
/client                          # React frontend (Vite)
  /src
    /components                  # Reusable UI (shadcn/ui)
      /ui                        # Base shadcn components
    /pages                       # Route pages
    /hooks                       # React hooks (useAuth, etc.)
    /lib                         # Utilities (queryClient, timezone)
    /contexts                    # React contexts
    App.tsx                      # Route definitions
    main.tsx                     # Entry point
  index.html

/server                          # Express backend
  /routes                        # API route handlers
  /services                      # Business logic
  /middleware                    # Express middleware
  /utils                         # Shared utilities
  auth.ts                        # Auth setup + endpoints
  storage.ts                     # IStorage interface + MemStorage
  supabase-storage.ts            # Production storage (PostgreSQL)
  app.ts                         # Express app setup + middleware
  index.ts                       # Production entry point

/shared                          # Shared types (frontend + backend)
  schema.ts                      # Zod schemas, Drizzle tables, TS types

/migrations                      # SQL migration files
```

### Vite configuratie
```typescript
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "client/src"),
      "@shared": path.resolve(__dirname, "shared"),
    },
  },
  root: path.resolve(__dirname, "client"),
  build: {
    outDir: path.resolve(__dirname, "dist/public"),
    emptyOutDir: true,
  },
});
```

### Drizzle configuratie
```typescript
// drizzle.config.ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  out: "./migrations",
  schema: "./shared/schema.ts",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

### Drizzle migration workflow

Drizzle wordt **alleen gebruikt voor schema-definities en migraties**. Alle queries gebruiken de `postgres` library direct (zie §4).

```bash
# === Development: direct push (geen migration files) ===
npm run db:push
# Introspects database, vergelijkt met shared/schema.ts, past wijzigingen toe.

# === Production: genereer + apply migration files ===
npx drizzle-kit generate    # Genereert SQL in /migrations
npx drizzle-kit migrate     # Past pending migrations toe op de database

# === Schema wijzigen (workflow): ===
# 1. Pas shared/schema.ts aan (Drizzle pgTable definitie)
# 2. Draai db:push (dev) of generate + migrate (prod)
# 3. Update IStorage interface + implementaties (MemStorage + SupabaseStorage)
```

**RLS apart:** Drizzle beheert geen RLS policies. Die staan in aparte `.sql` bestanden in `/migrations` en worden handmatig uitgevoerd na de initiële schema-migratie (zie §6).

**Transitie van `db:push` naar `generate+migrate`:**
Wanneer je van development naar productie gaat, schakel je over van `db:push` naar migratie-bestanden:
1. Zorg dat je schema in `shared/schema.ts` up-to-date is met de huidige database
2. Draai `npx drizzle-kit generate` — dit genereert de eerste migratie-snapshot
3. Markeer deze migratie als "al toegepast": `npx drizzle-kit migrate --skip` (of verwijder het gegenereerde SQL bestand als de database al up-to-date is)
4. Vanaf nu: elke schema-wijziging → `npx drizzle-kit generate` → review SQL → `npx drizzle-kit migrate`

### Scripts
```json
{
  "scripts": {
    "dev": "NODE_ENV=development tsx server/index-dev.ts",
    "build": "vite build && esbuild server/index.ts --bundle --platform=node --outdir=dist",
    "start": "NODE_ENV=production node dist/index.js",
    "check": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "db:push": "drizzle-kit push"
  }
}
```

### TypeScript configuratie (tsconfig.json)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": ".",
    "paths": {
      "@/*": ["client/src/*"],
      "@shared/*": ["shared/*"]
    }
  },
  "include": [
    "client/src/**/*",
    "server/**/*",
    "shared/**/*"
  ],
  "exclude": ["node_modules", "dist"]
}
```

**KRITIEK:** De `paths` moeten exact overeenkomen met de `resolve.alias` in `vite.config.ts`. Zonder dit geeft TypeScript errors op elke `@/` import.

### Tailwind configuratie (tailwind.config.js)

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  darkMode: "class",
  content: ["./client/index.html", "./client/src/**/*.{ts,tsx}"],
  theme: {
    extend: {
      // shadcn/ui CSS variabelen
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: { DEFAULT: "hsl(var(--primary))", foreground: "hsl(var(--primary-foreground))" },
        secondary: { DEFAULT: "hsl(var(--secondary))", foreground: "hsl(var(--secondary-foreground))" },
        destructive: { DEFAULT: "hsl(var(--destructive))", foreground: "hsl(var(--destructive-foreground))" },
        muted: { DEFAULT: "hsl(var(--muted))", foreground: "hsl(var(--muted-foreground))" },
        accent: { DEFAULT: "hsl(var(--accent))", foreground: "hsl(var(--accent-foreground))" },
        card: { DEFAULT: "hsl(var(--card))", foreground: "hsl(var(--card-foreground))" },
      },
      borderRadius: { lg: "var(--radius)", md: "calc(var(--radius) - 2px)", sm: "calc(var(--radius) - 4px)" },
    },
  },
  plugins: [require("tailwindcss-animate")],
};
```

### PostCSS configuratie (postcss.config.js)

```javascript
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

### shadcn/ui setup

shadcn/ui is **geen npm package** maar een CLI die componenten als broncode in je project plaatst. Setup:

```bash
# Initialiseer shadcn/ui (eenmalig)
npx shadcn-ui@latest init
# Kies: TypeScript, Default style, CSS variables, client/src/components/ui

# Componenten toevoegen
npx shadcn-ui@latest add button card dialog dropdown-menu input label toast
```

De `cn()` utility (class merging) wordt automatisch aangemaakt in `client/src/lib/utils.ts`:

```typescript
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

**CSS variabelen** staan in `client/src/index.css` (aangemaakt door shadcn-init). Dit bestand importeer je in `client/src/main.tsx`.

**Extra dependencies:** `clsx`, `tailwind-merge`, `tailwindcss-animate`, `class-variance-authority`, `autoprefixer` — worden automatisch geïnstalleerd door shadcn-init. Als je shadcn-init niet gebruikt, installeer ze handmatig:

```bash
npm install clsx tailwind-merge class-variance-authority
npm install -D tailwindcss-animate autoprefixer
```

### Frontend entry bestanden (VERPLICHT)

Deze drie bestanden zijn nodig voordat Vite je frontend kan bouwen. Ze worden niet automatisch aangemaakt.

#### client/index.html (Vite entry point)

```html
<!DOCTYPE html>
<html lang="nl">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>App</title>
    <meta name="description" content="B2B SaaS Platform" />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

**KRITIEK:** Het `src` attribuut verwijst naar `/src/main.tsx` (relatief aan de `client/` directory, niet de project root). Vite gebruikt `client/` als `root` (zie vite.config.ts).

#### client/src/main.tsx (React entry point)

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { QueryClientProvider } from "@tanstack/react-query";
import { queryClient } from "@/lib/queryClient";
import { AuthProvider } from "@/hooks/use-auth";
import App from "./App";
import "./index.css";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <AuthProvider>
        <App />
      </AuthProvider>
    </QueryClientProvider>
  </StrictMode>
);
```

**Provider volgorde:** `QueryClientProvider` → `AuthProvider` → `App`. AuthProvider gebruikt useQuery intern, dus moet binnen QueryClientProvider staan.

#### client/src/index.css (Tailwind + shadcn CSS variabelen)

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;
    --primary: 222.2 47.4% 11.2%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96.1%;
    --secondary-foreground: 222.2 47.4% 11.2%;
    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96.1%;
    --accent-foreground: 222.2 47.4% 11.2%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 222.2 84% 4.9%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;
    --primary: 210 40% 98%;
    --primary-foreground: 222.2 47.4% 11.2%;
    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;
    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 212.7 26.8% 83.9%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

**Dit bestand wordt aangemaakt door `npx shadcn-ui@latest init`** maar als je handmatig setup doet, moet je het zelf aanmaken. De CSS variabelen matchen met de Tailwind config in `tailwind.config.js`.

---

## 2. Database Schema

### Wanneer toepassen
Dit is het datamodel voor elke multi-tenant SaaS. Maak ALLE tabellen aan voordat je begint met code schrijven.

### Kern-tabellen (verplicht voor multi-tenancy)

```sql
-- === TENANTS (organisaties) ===
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,                    -- URL-friendly identifier
  billing_exempt BOOLEAN DEFAULT FALSE,          -- Enterprise: bypass Stripe
  is_active BOOLEAN DEFAULT TRUE,                -- FALSE = subscription canceled
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- === USERS ===
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username TEXT UNIQUE NOT NULL,
  email TEXT UNIQUE,                             -- Voor uitnodigingen + MFA
  display_name TEXT,
  password_hash TEXT NOT NULL,                   -- Bcrypt, NOOIT plain text
  global_role TEXT NOT NULL DEFAULT 'standard'
    CHECK (global_role IN ('superadmin', 'standard')),
  is_active BOOLEAN DEFAULT TRUE,                -- Soft-delete
  must_change_password BOOLEAN DEFAULT FALSE,    -- Forceer wachtwoord wijziging
  failed_login_attempts INTEGER DEFAULT 0,       -- Account lockout (§11)
  locked_until TIMESTAMPTZ,                      -- Account lockout (§11)
  last_login_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- === USER-TENANT KOPPELING (many-to-many) ===
CREATE TABLE user_tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  role TEXT NOT NULL DEFAULT 'member'
    CHECK (role IN ('admin', 'member')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, tenant_id)
);

-- === SETTINGS (key-value per tenant) ===
CREATE TABLE settings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID REFERENCES tenants(id),         -- NULL = global/system settings
  key TEXT NOT NULL,
  value TEXT NOT NULL,                           -- JSON string
  is_encrypted BOOLEAN DEFAULT FALSE,
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(tenant_id, key)
);

-- === ACTIVITY LOG (audit trail) ===
CREATE TABLE activity_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id TEXT NOT NULL,                       -- TEXT (niet UUID) — kan "system" bevatten voor cross-tenant events
                                                 -- Geen FK constraint: "system" is geen geldige tenant UUID
  user_id TEXT,
  action TEXT NOT NULL,
  field TEXT,
  before_value JSONB,
  after_value JSONB,
  context JSONB,
  timestamp TIMESTAMPTZ DEFAULT NOW()
);
```

### Authenticatie-tabellen

```sql
-- === TOKENS (uitnodigingen + password resets) ===
CREATE TABLE tokens (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  token_hash TEXT NOT NULL,                      -- Bcrypt hash, NOOIT raw token
  token_prefix VARCHAR(8),                       -- Eerste 8 chars van raw token (O(1) lookup)
  type TEXT NOT NULL CHECK (type IN ('invitation', 'password_reset', 'account_deletion')),
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'accepted', 'used', 'expired', 'revoked')),
  email TEXT NOT NULL,
  display_name TEXT,
  tenant_id UUID REFERENCES tenants(id),         -- Voor uitnodigingen
  tenant_role TEXT,                              -- Role bij acceptatie
  user_id UUID REFERENCES users(id),            -- Voor password resets
  invited_by_id UUID REFERENCES users(id),
  expires_at TIMESTAMPTZ NOT NULL,
  used_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- === PASSWORD HISTORY (voorkom hergebruik) ===
CREATE TABLE password_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  password_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_password_history_user ON password_history(user_id, created_at DESC);

-- === OAUTH TOKENS (versleuteld) ===
CREATE TABLE oauth_tokens (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  provider TEXT NOT NULL CHECK (provider IN ('microsoft', 'google')),
  access_token TEXT NOT NULL,                    -- Versleuteld (AES-256-GCM)
  refresh_token TEXT,                            -- Versleuteld
  token_expires_at TIMESTAMPTZ,
  connected_user_email TEXT,
  scopes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- === OAUTH PENDING STATES (CSRF voor OAuth flow) ===
CREATE TABLE oauth_pending_states (
  state TEXT PRIMARY KEY,                        -- Random nonce
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  provider TEXT NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,               -- Voorkom replay attacks
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Billing-tabellen

```sql
-- === STRIPE SUBSCRIPTIONS ===
CREATE TABLE stripe_subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  stripe_customer_id TEXT NOT NULL,
  stripe_subscription_id TEXT UNIQUE,
  stripe_price_id TEXT,
  status TEXT NOT NULL DEFAULT 'incomplete'
    CHECK (status IN ('trialing', 'active', 'past_due', 'canceled',
                      'unpaid', 'incomplete', 'incomplete_expired', 'paused')),
  tier TEXT NOT NULL DEFAULT 'starter'
    CHECK (tier IN ('starter', 'pro', 'enterprise')),
  billing_interval TEXT NOT NULL DEFAULT 'monthly'
    CHECK (billing_interval IN ('monthly', 'annual')),
  trial_ends_at TIMESTAMPTZ,
  trial_started_at TIMESTAMPTZ,
  current_period_start TIMESTAMPTZ,
  current_period_end TIMESTAMPTZ,
  cancel_at_period_end BOOLEAN DEFAULT FALSE,
  canceled_at TIMESTAMPTZ,
  welcome_email_sent_at TIMESTAMPTZ,             -- Idempotency: voorkom dubbele emails
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Domein-tabel (voorbeeld: items — vervang door jouw entiteit)

```sql
-- === JOUW DOMEIN-ENTITEIT (vervang "items" door projects, invoices, tickets, etc.) ===
CREATE TABLE items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),  -- ALTIJD tenant_id!
  title TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  metadata JSONB,                                   -- Flexibel veld voor domein-data
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_items_tenant ON items(tenant_id, created_at DESC);

-- === CHAT / AI THREADS (optioneel) ===
CREATE TABLE chat_threads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  user_id UUID NOT NULL REFERENCES users(id),
  title TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE chat_messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  thread_id UUID NOT NULL REFERENCES chat_threads(id) ON DELETE CASCADE,
  sender TEXT NOT NULL CHECK (sender IN ('user', 'assistant')),
  content TEXT NOT NULL,
  citations JSONB,
  tokens_used INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- === JOB QUEUE (voor achtergrondverwerking, §24) ===
CREATE TABLE job_queue (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_id UUID NOT NULL,                        -- ID van het te verwerken item
  tenant_id UUID NOT NULL,
  job_type TEXT NOT NULL DEFAULT 'default',        -- Type job (bijv. 'generate_report', 'send_email')
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
  retry_count INTEGER DEFAULT 0,
  correlation_id TEXT,
  error_message TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_job_queue_status ON job_queue(status, created_at);

-- === WEBHOOK DEDUPLICATIE (§22) ===
CREATE TABLE processed_webhooks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  webhook_id TEXT UNIQUE NOT NULL,              -- Stripe event ID, Recall webhook ID, etc.
  source TEXT NOT NULL DEFAULT 'stripe',        -- 'stripe', 'recall', etc.
  processed_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_processed_webhooks_id ON processed_webhooks(webhook_id);
-- Cleanup job: DELETE FROM processed_webhooks WHERE processed_at < NOW() - INTERVAL '30 days';
```

### JSONB gotcha

**KRITIEK:** Gebruik ALTIJD `sql.json()` voor JSONB kolommen in PostgreSQL, NOOIT `JSON.stringify()`:

```typescript
// FOUT — dubbel-escaped string in database
await sql`INSERT INTO items (metadata) VALUES (${JSON.stringify(metadata)})`;

// GOED — correct JSONB formaat
await sql`INSERT INTO items (metadata) VALUES (${sql.json(metadata)})`;
```

Als je dit fout doet, krijg je errors als `metadata.map is not a function` omdat de waarde een string is ipv een object/array.

### Bestaande database migreren (brownfield)

Heb je al een werkende database met users en domein-data? Volg deze stappen om multi-tenancy toe te voegen **zonder data te verliezen**.

```sql
-- === STAP 1: Maak de tenants tabel aan ===
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  billing_exempt BOOLEAN DEFAULT FALSE,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- === STAP 2: Maak een tenant aan voor je bestaande organisatie ===
INSERT INTO tenants (name, slug) VALUES ('Mijn Bedrijf', 'mijn-bedrijf');
-- Noteer het tenant ID (gebruik in volgende stappen)
-- SELECT id FROM tenants WHERE slug = 'mijn-bedrijf';

-- === STAP 3: Voeg ontbrekende kolommen toe aan users ===
ALTER TABLE users ADD COLUMN IF NOT EXISTS global_role TEXT NOT NULL DEFAULT 'standard';
ALTER TABLE users ADD COLUMN IF NOT EXISTS is_active BOOLEAN DEFAULT TRUE;
ALTER TABLE users ADD COLUMN IF NOT EXISTS must_change_password BOOLEAN DEFAULT FALSE;
-- Als je email nog niet als kolom hebt:
-- ALTER TABLE users ADD COLUMN IF NOT EXISTS email TEXT UNIQUE;

-- === STAP 4: Maak user_tenants koppeltabel ===
CREATE TABLE IF NOT EXISTS user_tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  role TEXT NOT NULL DEFAULT 'member',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, tenant_id)
);

-- Koppel ALLE bestaande users aan je tenant
INSERT INTO user_tenants (user_id, tenant_id, role)
SELECT id, (SELECT id FROM tenants WHERE slug = 'mijn-bedrijf'), 'member'
FROM users;

-- Maak je eigen account admin
UPDATE user_tenants SET role = 'admin'
WHERE user_id = (SELECT id FROM users WHERE email = 'jouw@email.com');

-- === STAP 5: Voeg tenant_id toe aan je domein-tabellen ===
-- Voorbeeld voor een "items" tabel (pas aan voor jouw tabellen)
ALTER TABLE items ADD COLUMN IF NOT EXISTS tenant_id UUID REFERENCES tenants(id);

-- Backfill: alle bestaande data naar je tenant
UPDATE items SET tenant_id = (SELECT id FROM tenants WHERE slug = 'mijn-bedrijf')
WHERE tenant_id IS NULL;

-- Nu pas NOT NULL afdwingen
ALTER TABLE items ALTER COLUMN tenant_id SET NOT NULL;

-- Index voor performance
CREATE INDEX IF NOT EXISTS idx_items_tenant ON items(tenant_id, created_at DESC);

-- === STAP 6: Herhaal stap 5 voor ELKE tabel met domein-data ===
-- Bijv: projects, invoices, documents, etc.
-- Patroon: ADD COLUMN → backfill → SET NOT NULL → CREATE INDEX
```

**Volgorde is belangrijk:**
1. Eerst `tenants` + `user_tenants` aanmaken
2. Dan domein-tabellen uitbreiden met `tenant_id`
3. Dan pas de applicatiecode updaten (§4-5: storage interface + routes)
4. Als laatste RLS activeren (§6)

**Verifieer na migratie:**
```sql
-- Geen orphaned data (alles heeft een tenant)
SELECT COUNT(*) FROM items WHERE tenant_id IS NULL;  -- Moet 0 zijn

-- Alle users gekoppeld aan een tenant
SELECT u.id, u.email FROM users u
LEFT JOIN user_tenants ut ON u.id = ut.user_id
WHERE ut.id IS NULL;  -- Moet leeg zijn
```

### Array safety

**KRITIEK:** Altijd `Array.isArray()` checks toevoegen wanneer je data van de API ontvangt:

```typescript
// FOUT — crasht als API een object {} retourneert ipv []
const items = data.items;
items.filter(m => m.status === "completed");

// GOED — defensief programmeren
const items = Array.isArray(data?.items) ? data.items : [];
items.filter(m => m.status === "completed");
```

---

## 3. Shared TypeScript Types

### Wanneer toepassen
Altijd. Dit bestand wordt gedeeld tussen frontend en backend. Het is de single source of truth voor alle types.

### shared/schema.ts — De kern

```typescript
import { pgTable, text, varchar, uuid, boolean, integer, timestamp, jsonb } from "drizzle-orm/pg-core";
import { createInsertSchema, createSelectSchema } from "drizzle-zod";
import { z } from "zod";

// === ENUMS ===
export const globalRoleEnum = z.enum(["superadmin", "standard"]);
export const tenantRoleEnum = z.enum(["admin", "member"]);
export const subscriptionTierEnum = z.enum(["starter", "pro", "enterprise"]);
export const billingIntervalEnum = z.enum(["monthly", "annual"]);
export type GlobalRole = z.infer<typeof globalRoleEnum>;
export type TenantRole = z.infer<typeof tenantRoleEnum>;
export type SubscriptionTier = z.infer<typeof subscriptionTierEnum>;
export type BillingInterval = z.infer<typeof billingIntervalEnum>;

// === DRIZZLE TABLE DEFINITIONS ===
export const tenants = pgTable("tenants", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: text("name").notNull(),
  slug: text("slug").unique().notNull(),
  billingExempt: boolean("billing_exempt").notNull().default(false),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  username: text("username").unique().notNull(),
  email: text("email").unique(),
  displayName: text("display_name"),
  passwordHash: text("password_hash").notNull(),
  globalRole: text("global_role").notNull().default("standard"),
  isActive: boolean("is_active").notNull().default(true),
  mustChangePassword: boolean("must_change_password").notNull().default(false),
  failedLoginAttempts: integer("failed_login_attempts").notNull().default(0),
  lockedUntil: timestamp("locked_until"),
  lastLoginAt: timestamp("last_login_at"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});

export const userTenants = pgTable("user_tenants", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id, { onDelete: "cascade" }),
  role: text("role").notNull().default("member"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});

// === OVERIGE DRIZZLE PGTABLE DEFINITIES ===

export const tokens = pgTable("tokens", {
  id: uuid("id").primaryKey().defaultRandom(),
  tokenHash: text("token_hash").notNull(),
  tokenPrefix: varchar("token_prefix", { length: 8 }),  // Matches SQL VARCHAR(8)
  type: text("type").notNull(),
  status: text("status").notNull().default("pending"),
  email: text("email").notNull(),
  displayName: text("display_name"),
  tenantId: uuid("tenant_id").references(() => tenants.id),
  tenantRole: text("tenant_role"),
  userId: uuid("user_id").references(() => users.id),
  invitedById: uuid("invited_by_id").references(() => users.id),
  expiresAt: timestamp("expires_at").notNull(),
  usedAt: timestamp("used_at"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});

export const passwordHistory = pgTable("password_history", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  passwordHash: text("password_hash").notNull(),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});

export const oauthTokens = pgTable("oauth_tokens", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id, { onDelete: "cascade" }),
  provider: text("provider").notNull(),
  accessToken: text("access_token").notNull(),
  refreshToken: text("refresh_token"),
  tokenExpiresAt: timestamp("token_expires_at"),
  connectedUserEmail: text("connected_user_email"),
  scopes: text("scopes"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});

export const oauthPendingStates = pgTable("oauth_pending_states", {
  state: text("state").primaryKey(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id),
  provider: text("provider").notNull(),
  expiresAt: timestamp("expires_at").notNull(),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});

export const settings = pgTable("settings", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: uuid("tenant_id").references(() => tenants.id),
  key: text("key").notNull(),
  value: text("value").notNull(),
  isEncrypted: boolean("is_encrypted").default(false),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});

export const activityLog = pgTable("activity_log", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: text("tenant_id").notNull(),
  userId: text("user_id"),
  action: text("action").notNull(),
  field: text("field"),
  beforeValue: jsonb("before_value"),
  afterValue: jsonb("after_value"),
  context: jsonb("context"),
  timestamp: timestamp("timestamp").notNull().defaultNow(),
});

export const stripeSubscriptions = pgTable("stripe_subscriptions", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id),
  stripeCustomerId: text("stripe_customer_id").notNull(),
  stripeSubscriptionId: text("stripe_subscription_id").unique(),
  stripePriceId: text("stripe_price_id"),
  status: text("status").notNull().default("incomplete"),
  tier: text("tier").notNull().default("starter"),
  billingInterval: text("billing_interval").notNull().default("monthly"),
  trialEndsAt: timestamp("trial_ends_at"),
  trialStartedAt: timestamp("trial_started_at"),
  currentPeriodStart: timestamp("current_period_start"),
  currentPeriodEnd: timestamp("current_period_end"),
  cancelAtPeriodEnd: boolean("cancel_at_period_end").default(false),
  canceledAt: timestamp("canceled_at"),
  welcomeEmailSentAt: timestamp("welcome_email_sent_at"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});

// === DOMEIN-ENTITEIT (vervang door jouw entiteit) ===
export const items = pgTable("items", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id),
  title: text("title").notNull(),
  status: text("status").notNull().default("pending"),
  metadata: jsonb("metadata"),  // Flexibel JSONB veld voor domein-data
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});

export const chatThreads = pgTable("chat_threads", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id),
  userId: uuid("user_id").notNull().references(() => users.id),
  title: text("title"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});

export const chatMessages = pgTable("chat_messages", {
  id: uuid("id").primaryKey().defaultRandom(),
  threadId: uuid("thread_id").notNull().references(() => chatThreads.id, { onDelete: "cascade" }),
  sender: text("sender").notNull(),
  content: text("content").notNull(),
  citations: jsonb("citations"),
  tokensUsed: integer("tokens_used"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});

export const jobQueue = pgTable("job_queue", {
  id: uuid("id").primaryKey().defaultRandom(),
  entityId: uuid("entity_id").notNull(),
  tenantId: uuid("tenant_id").notNull(),
  jobType: text("job_type").notNull().default("default"),
  status: text("status").notNull().default("pending"),
  retryCount: integer("retry_count").default(0),
  correlationId: text("correlation_id"),
  errorMessage: text("error_message"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  startedAt: timestamp("started_at"),
  completedAt: timestamp("completed_at"),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});

export const processedWebhooks = pgTable("processed_webhooks", {
  id: uuid("id").primaryKey().defaultRandom(),
  webhookId: text("webhook_id").unique().notNull(),
  source: text("source").notNull().default("stripe"),
  processedAt: timestamp("processed_at").notNull().defaultNow(),
});

// De bijbehorende TypeScript types staan verderop in dit bestand (zoek op "OVERIGE TYPE DEFINITIES")
```

### Performance indexes

Voeg deze indexes toe nadat je schema draait. Zonder indexes zijn dashboard queries traag bij >1000 records.

```sql
-- Domein-entiteit: dashboard, lijst, facturering (pas tabelnaam aan)
CREATE INDEX IF NOT EXISTS idx_items_tenant_status ON items(tenant_id, status);
CREATE INDEX IF NOT EXISTS idx_items_tenant_created ON items(tenant_id, created_at DESC);

-- Activity log: audit trail
CREATE INDEX IF NOT EXISTS idx_activity_log_tenant_timestamp ON activity_log(tenant_id, timestamp DESC);

-- Tokens: uitnodigingen, password resets
CREATE INDEX IF NOT EXISTS idx_tokens_tenant_type_status ON tokens(tenant_id, type, status);
CREATE INDEX IF NOT EXISTS idx_tokens_email_type ON tokens(email, type);
CREATE INDEX IF NOT EXISTS idx_tokens_prefix_type_status ON tokens(token_prefix, type, status);  -- O(1) prefix lookup

-- Settings: key lookups per tenant
CREATE INDEX IF NOT EXISTS idx_settings_tenant_key ON settings(tenant_id, key);

-- User-tenants: reverse lookups
CREATE INDEX IF NOT EXISTS idx_user_tenants_tenant ON user_tenants(tenant_id);
```

**Vuistregel:** Voeg een index toe voor elke `WHERE` combinatie die je storage layer gebruikt. Gebruik `EXPLAIN ANALYZE` om te verifiëren.

```typescript
// === INFERRED TYPES ===
export type Tenant = typeof tenants.$inferSelect;
export type InsertTenant = typeof tenants.$inferInsert;
export type User = typeof users.$inferSelect;
export type UserTenant = typeof userTenants.$inferSelect;

// === ZOD VALIDATION SCHEMAS ===
export const loginSchema = z.object({
  email: z.string().email("Ongeldig emailadres").max(255),
  password: z.string().min(1, "Wachtwoord is verplicht").max(128),
});

export const changePasswordSchema = z.object({
  currentPassword: z.string().min(1).max(128),
  newPassword: z.string()
    .min(12, "Minimaal 12 tekens").max(128)
    .regex(/[A-Z]/, "Minimaal 1 hoofdletter")
    .regex(/[a-z]/, "Minimaal 1 kleine letter")
    .regex(/[0-9]/, "Minimaal 1 cijfer"),
});

export const registerSchema = z.object({
  email: z.string().email("Ongeldig emailadres").max(255),
  password: z.string()
    .min(12, "Minimaal 12 tekens").max(128)
    .regex(/[A-Z]/, "Minimaal 1 hoofdletter")
    .regex(/[a-z]/, "Minimaal 1 kleine letter")
    .regex(/[0-9]/, "Minimaal 1 cijfer"),
  companyName: z.string().min(2, "Bedrijfsnaam is verplicht").max(100),
  displayName: z.string().max(100).optional(),
});

export const acceptInvitationSchema = z.object({
  token: z.string().min(1).max(500),
  password: z.string()
    .min(12, "Minimaal 12 tekens").max(128)
    .regex(/[A-Z]/, "Minimaal 1 hoofdletter")
    .regex(/[a-z]/, "Minimaal 1 kleine letter")
    .regex(/[0-9]/, "Minimaal 1 cijfer"),
});

// === SESSION USER (gedeeld tussen frontend en backend) ===
export interface SessionUser {
  id: string;
  username: string;
  email?: string;
  displayName?: string;
  globalRole: GlobalRole;
  activeTenantId?: string;
  tenants: Array<{
    id: string;
    name: string;
    slug: string;
    role: TenantRole;
  }>;
  mustChangePassword?: boolean;
  isImpersonating?: boolean;
  impersonationStartedAt?: number;  // Date.now() — voor auto-expire (§12)
  originalUser?: { id: string; username: string; globalRole: GlobalRole };
}

// === EXPRESS SESSION TYPE DECLARATIONS ===
// KRITIEK: Zonder dit crasht TypeScript op elke req.session.user toegang
declare module "express-session" {
  interface SessionData {
    user?: SessionUser;
    csrfToken?: string;
    createdAt?: number;            // Date.now() — voor absolute session timeout (§7)
    // MFA fields
    mfaPending?: boolean;
    mfaCodeHash?: string;
    mfaCodeExpiresAt?: number;
    mfaUserId?: string;
    mfaTenantId?: string;         // Tenant die MFA vereiste (nodig bij verify voor session restore)
  }
}
// Plaats dit bovenin server/auth.ts (voor alle andere imports)

// === BILLING STATUS ===
export interface BillingStatus {
  hasSubscription: boolean;
  tier: SubscriptionTier;
  status?: string;
  trialEndsAt?: string;
  currentPeriodEnd?: string;
  cancelAtPeriodEnd?: boolean;
  isTrialing: boolean;
  isActive: boolean;
  accessLevel: "full" | "warning" | "blocked";
  accessReason: string;
  accessMessage?: string;
  daysUntilBlocked?: number;
  daysUntilCancellation?: number;
  billingExempt?: boolean;
}

// === SUBSCRIPTION TIERS ===
export const SUBSCRIPTION_TIERS = {
  starter: {
    name: "Starter",
    priceMonthly: 49,       // EUR
    priceAnnual: 470,        // EUR (20% korting)
    features: ["50 items/maand", "1 gebruiker", "Basis functies"],
    limits: { itemsPerMonth: 50, users: 1 },
  },
  pro: {
    name: "Pro",
    priceMonthly: 149,
    priceAnnual: 1430,
    features: ["250 items/maand", "10 gebruikers", "Alle integraties"],
    limits: { itemsPerMonth: 250, users: 10 },
  },
  enterprise: {
    name: "Enterprise",
    priceMonthly: null,      // Custom pricing
    priceAnnual: null,
    features: ["Onbeperkt", "Onbeperkt gebruikers", "Dedicated support"],
    limits: { itemsPerMonth: Infinity, users: Infinity },
    // LET OP: JSON.stringify(Infinity) → null. Als je SUBSCRIPTION_TIERS via een API response
    // naar de frontend stuurt, gebruik dan Number.MAX_SAFE_INTEGER of -1 als sentinel.
    // Binnen de app (import) werkt Infinity correct.
  },
} as const;

// === OVERIGE TYPE DEFINITIES ===
// Alle types die de IStorage interface (§4) en route handlers gebruiken

export type Token = {
  id: string;
  tokenHash: string;
  tokenPrefix?: string;           // Eerste 8 chars van raw token (O(1) lookup)
  type: "invitation" | "password_reset" | "account_deletion";
  status: "pending" | "accepted" | "used" | "expired" | "revoked";
  email: string;
  displayName?: string;
  tenantId?: string;
  tenantRole?: string;
  userId?: string;
  invitedById?: string;
  expiresAt: Date;
  usedAt?: Date;
  createdAt: Date;
};

export type InsertToken = Omit<Token, "id" | "createdAt" | "usedAt"> & {
  displayName?: string;
};

export type StripeSubscription = {
  id: string;
  tenantId: string;
  stripeCustomerId: string;
  stripeSubscriptionId?: string;
  stripePriceId?: string;
  status: string;
  tier: SubscriptionTier;
  billingInterval: BillingInterval;
  trialEndsAt?: Date;
  trialStartedAt?: Date;
  currentPeriodStart?: Date | null;
  currentPeriodEnd?: Date | null;
  cancelAtPeriodEnd: boolean;
  canceledAt?: Date;
  welcomeEmailSentAt?: Date;
  createdAt: Date;
  updatedAt: Date;
};

export type InsertStripeSubscription = Omit<StripeSubscription, "id" | "createdAt" | "updatedAt"> & {
  cancelAtPeriodEnd?: boolean;
};

// === DOMEIN-ENTITEIT (vervang door jouw entiteit: Project, Invoice, Ticket, etc.) ===
export type Item = {
  id: string;
  tenantId: string;
  title: string;
  status: string;
  metadata?: Record<string, unknown>;  // JSONB — flexibel veld voor domein-data
  createdAt: Date;
  updatedAt: Date;
  // Voeg hier jouw domein-specifieke velden toe
};

export type InsertItem = Omit<Item, "id" | "createdAt" | "updatedAt">;

export type ActivityLog = {
  id: string;
  tenantId: string;
  userId?: string;
  action: string;
  field?: string;
  beforeValue?: unknown;
  afterValue?: unknown;
  context?: Record<string, unknown>;
  timestamp: Date;
};

export type InsertActivityLog = Omit<ActivityLog, "id" | "timestamp">;

export type ChatThread = {
  id: string;
  tenantId: string;
  userId: string;
  title?: string;
  createdAt: Date;
  updatedAt: Date;
};

export type InsertChatThread = Omit<ChatThread, "id" | "createdAt" | "updatedAt">;

export type ChatMessage = {
  id: string;
  threadId: string;
  sender: "user" | "assistant";
  content: string;
  citations?: unknown;
  tokensUsed?: number;
  createdAt: Date;
};

export type InsertChatMessage = Omit<ChatMessage, "id" | "createdAt">;

export type InvitationStatus = Token & {
  // Uitgebreid met display info voor de uitnodigingslijst
  tenantName?: string;
  invitedByName?: string;
};

// === TIER LIMIT RESULT ===
export type TierLimitResult = {
  allowed: boolean;
  reason?: string;
  current?: number;
  limit?: number;
};
```

---

## 4. Storage Interface Pattern

### Wanneer toepassen
Altijd. Dit patroon geeft je twee implementaties: MemStorage (voor tests/development) en SupabaseStorage (voor productie). Hierdoor kun je tests schrijven zonder database.

### Drizzle ORM vs raw SQL — BELANGRIJK
Drizzle wordt **alleen gebruikt voor schema-definities en migraties**, NIET als query builder:
- `shared/schema.ts` — Drizzle `pgTable()` definities → voor TypeScript types + migratie-generatie
- `drizzle-kit push` — Schema naar database pushen
- **Alle queries** gebruiken de `postgres` library's `sql` tagged templates direct

```typescript
// GOED — postgres library tagged template (wat we gebruiken)
const rows = await sql`SELECT * FROM items WHERE tenant_id = ${tenantId}`;

// NIET GEBRUIKT — Drizzle query builder
const rows = await db.select().from(items).where(eq(items.tenantId, tenantId));
```

Waarom: De `postgres` library geeft directe controle over SQL, wat essentieel is voor RLS (`set_config`), `FOR UPDATE SKIP LOCKED`, en complexe joins. Drizzle's abstractie voegt hier geen waarde toe.

### server/storage.ts — Interface + MemStorage

```typescript
// De interface definieert ALLE database operaties
// KRITIEK: tenantId is string | null — NOOIT optioneel
//   string = tenant-scoped (RLS)
//   null   = superadmin/system bypass (geen RLS)
export interface IStorage {
  // --- Tenants ---
  getTenant(id: string): Promise<Tenant | undefined>;
  getTenantBySlug(slug: string): Promise<Tenant | undefined>;
  createTenant(tenant: InsertTenant): Promise<Tenant>;
  updateTenant(id: string, updates: Partial<Tenant>): Promise<Tenant | undefined>;

  // --- Users ---
  getUser(id: string): Promise<User | undefined>;
  getUserByEmail(email: string): Promise<User | undefined>;
  createUser(username: string, passwordHash: string, globalRole?: GlobalRole): Promise<User>;
  createUserWithDetails(details: {
    username: string; email: string; passwordHash: string;
    displayName?: string; globalRole: GlobalRole; isActive: boolean;
    mustChangePassword: boolean;
  }): Promise<User>;
  getUserByUsername(username: string): Promise<User | undefined>;
  updateUserDetails(id: string, updates: Partial<User>): Promise<User | undefined>;
  deleteUser(id: string): Promise<boolean>;

  // --- User-Tenant relaties ---
  getUserTenants(userId: string): Promise<UserTenantWithDetails[]>;
  getUsersForTenant(tenantId: string): Promise<Array<{ id: string; role: string }>>;
  addUserToTenant(userId: string, tenantId: string, role?: TenantRole): Promise<UserTenant>;
  removeUserFromTenant(userId: string, tenantId: string): Promise<{ success: boolean }>;

  // --- Domein-entiteiten (vervang "Item" door jouw entiteit) ---
  // BELANGRIJK: tenantId: string | null
  getItems(tenantId: string | null): Promise<Item[]>;
  getItem(id: string, tenantId: string | null): Promise<Item | undefined>;
  createItem(item: InsertItem): Promise<Item>;
  updateItem(id: string, updates: Partial<Item>, tenantId: string | null): Promise<Item | undefined>;
  deleteItem(id: string, tenantId: string | null): Promise<boolean>;
  getEntityCountForPeriod(tenantId: string): Promise<number>;  // Voor tier limits (§17)

  // --- Settings (key-value per tenant) ---
  getSetting(key: string, tenantId: string | null): Promise<string | null>;
  setSetting(key: string, value: string, isEncrypted: boolean, tenantId: string | null): Promise<void>;
  getAllSettings(tenantId: string | null): Promise<Record<string, string>>;

  // --- Tokens (uitnodigingen + password resets) ---
  createToken(token: InsertToken): Promise<Token>;
  getPendingTokensByType(type: Token["type"]): Promise<Token[]>;
  updateTokenStatus(id: string, status: string, usedAt?: Date): Promise<void>;
  revokePendingTokens(email: string, tenantId: string, type: string): Promise<void>;
  getTenantInvitations(tenantId: string): Promise<InvitationStatus[]>;

  // --- Password history ---
  addPasswordToHistory(userId: string, passwordHash: string): Promise<void>;
  getPasswordHistory(userId: string, limit?: number): Promise<string[]>;

  // --- Activity log ---
  createActivityLog(log: InsertActivityLog): Promise<ActivityLog>;
  getActivityLogs(tenantId: string | null, options?: { userId?: string }): Promise<ActivityLog[]>;

  // --- Stripe subscriptions ---
  getStripeSubscription(tenantId: string): Promise<StripeSubscription | undefined>;
  upsertStripeSubscription(sub: InsertStripeSubscription): Promise<StripeSubscription>;
  updateStripeSubscription(tenantId: string, updates: Partial<StripeSubscription>): Promise<StripeSubscription | undefined>;

  // --- Chat threads (optioneel) ---
  getChatThreads(userId: string, tenantId: string): Promise<ChatThread[]>;
  getChatThread(id: string, tenantId: string): Promise<ChatThread | undefined>;
  createChatThread(thread: InsertChatThread): Promise<ChatThread>;
  deleteChatThread(id: string, tenantId: string): Promise<boolean>;
  getChatMessages(threadId: string, tenantId: string): Promise<ChatMessage[]>;
  createChatMessage(message: InsertChatMessage, tenantId: string): Promise<ChatMessage>;

  // --- Tenant settings (bulk) ---
  updateSystemSettings(settings: Record<string, string>, tenantId: string): Promise<void>;

  // --- Token prefix lookup ---
  getPendingTokensByPrefix(prefix: string, type: Token["type"]): Promise<Token[]>;

  // --- Webhook deduplicatie (§22) ---
  isWebhookProcessed(webhookId: string): Promise<boolean>;
  markWebhookProcessed(webhookId: string, source?: string): Promise<void>;

  // --- Lifecycle ---
  healthCheck(): Promise<{ ok: boolean }>;  // Simpele SELECT 1 check
  close(): Promise<void>;  // Sluit database pool (voor graceful shutdown §25)
}

// Type voor tenant-info bij user queries
export interface UserTenantWithDetails {
  tenantId: string;
  tenantName: string;
  tenantSlug: string;
  role: TenantRole;
  createdAt: Date;
}

// MemStorage: in-memory implementatie voor tests
// ALLE methoden van IStorage zijn geïmplementeerd — je kunt dit direct gebruiken voor unit tests
export class MemStorage implements IStorage {
  private users = new Map<string, User>();
  private tenants = new Map<string, Tenant>();
  private userTenants = new Map<string, UserTenant>();
  private items = new Map<string, Item>();
  private tokens = new Map<string, Token>();
  private passwordHistory = new Map<string, Array<{ hash: string; createdAt: Date }>>();
  private settings = new Map<string, { value: string; isEncrypted: boolean }>();
  private activityLogs: ActivityLog[] = [];
  private stripeSubscriptions = new Map<string, StripeSubscription>();
  private chatThreads = new Map<string, ChatThread>();
  private chatMessages = new Map<string, ChatMessage>();
  private processedWebhookIds = new Set<string>();
  private idCounter = 0;

  private nextId(): string { return `mem_${++this.idCounter}_${Date.now().toString(36)}`; }

  // --- Tenants ---
  async getTenant(id: string): Promise<Tenant | undefined> { return this.tenants.get(id); }
  async getTenantBySlug(slug: string): Promise<Tenant | undefined> {
    return Array.from(this.tenants.values()).find(t => t.slug === slug);
  }
  async createTenant(data: InsertTenant): Promise<Tenant> {
    const tenant: Tenant = { id: this.nextId(), createdAt: new Date(), billingExempt: false, isActive: true, ...data };
    this.tenants.set(tenant.id, tenant);
    return tenant;
  }
  async updateTenant(id: string, updates: Partial<Tenant>): Promise<Tenant | undefined> {
    const t = this.tenants.get(id);
    if (!t) return undefined;
    const updated = { ...t, ...updates };
    this.tenants.set(id, updated);
    return updated;
  }

  // --- Users ---
  async getUser(id: string): Promise<User | undefined> { return this.users.get(id); }
  async getUserByEmail(email: string): Promise<User | undefined> {
    const lower = email.toLowerCase();
    return Array.from(this.users.values()).find(u => u.email?.toLowerCase() === lower);
  }
  async getUserByUsername(username: string): Promise<User | undefined> {
    return Array.from(this.users.values()).find(u => u.username === username);
  }
  async createUser(username: string, passwordHash: string, globalRole?: GlobalRole): Promise<User> {
    const user: User = {
      id: this.nextId(), username, passwordHash, email: null, displayName: null,
      globalRole: globalRole || "standard", isActive: true, mustChangePassword: false,
      failedLoginAttempts: 0, lockedUntil: null, lastLoginAt: null, createdAt: new Date(),
    };
    this.users.set(user.id, user);
    return user;
  }
  async createUserWithDetails(details: {
    username: string; email: string; passwordHash: string;
    displayName?: string; globalRole: GlobalRole; isActive: boolean; mustChangePassword: boolean;
  }): Promise<User> {
    const user: User = {
      id: this.nextId(), username: details.username, email: details.email,
      passwordHash: details.passwordHash, displayName: details.displayName || null,
      globalRole: details.globalRole, isActive: details.isActive,
      mustChangePassword: details.mustChangePassword,
      failedLoginAttempts: 0, lockedUntil: null, lastLoginAt: null, createdAt: new Date(),
    };
    this.users.set(user.id, user);
    return user;
  }
  async updateUserDetails(id: string, updates: Partial<User>): Promise<User | undefined> {
    const u = this.users.get(id);
    if (!u) return undefined;
    const updated = { ...u, ...updates };
    this.users.set(id, updated);
    return updated;
  }
  async deleteUser(id: string): Promise<boolean> { return this.users.delete(id); }

  // --- User-Tenant relaties ---
  async getUserTenants(userId: string): Promise<UserTenantWithDetails[]> {
    return Array.from(this.userTenants.values())
      .filter(ut => ut.userId === userId)
      .map(ut => {
        const tenant = this.tenants.get(ut.tenantId);
        return {
          tenantId: ut.tenantId, tenantName: tenant?.name || "", tenantSlug: tenant?.slug || "",
          role: ut.role as TenantRole, createdAt: ut.createdAt ?? new Date(),
        };
      });
  }
  async getUsersForTenant(tenantId: string): Promise<Array<{ id: string; role: string }>> {
    return Array.from(this.userTenants.values())
      .filter(ut => ut.tenantId === tenantId)
      .map(ut => ({ id: ut.userId, role: ut.role }));
  }
  async addUserToTenant(userId: string, tenantId: string, role?: TenantRole): Promise<UserTenant> {
    const ut: UserTenant = {
      id: this.nextId(), userId, tenantId, role: role || "member",
      createdAt: new Date(),
    };
    this.userTenants.set(ut.id, ut);
    return ut;
  }
  async removeUserFromTenant(userId: string, tenantId: string): Promise<{ success: boolean }> {
    for (const [key, ut] of this.userTenants) {
      if (ut.userId === userId && ut.tenantId === tenantId) { this.userTenants.delete(key); return { success: true }; }
    }
    return { success: false };
  }

  // --- Items (domein-entiteit — vervang door jouw entiteit) ---
  async getItems(tenantId: string | null): Promise<Item[]> {
    return Array.from(this.items.values())
      .filter(m => tenantId === null || m.tenantId === tenantId);
  }
  async getItem(id: string, tenantId: string | null): Promise<Item | undefined> {
    const item = this.items.get(id);
    if (!item) return undefined;
    if (tenantId !== null && item.tenantId !== tenantId) return undefined;
    return item;
  }
  async createItem(data: InsertItem): Promise<Item> {
    const item: Item = { id: this.nextId(), createdAt: new Date(), updatedAt: new Date(), ...data };
    this.items.set(item.id, item);
    return item;
  }
  async updateItem(id: string, updates: Partial<Item>, tenantId: string | null): Promise<Item | undefined> {
    const m = await this.getItem(id, tenantId);
    if (!m) return undefined;
    const updated = { ...m, ...updates, updatedAt: new Date() };
    this.items.set(id, updated);
    return updated;
  }
  async deleteItem(id: string, tenantId: string | null): Promise<boolean> {
    const m = await this.getItem(id, tenantId);
    if (!m) return false;
    return this.items.delete(id);
  }
  async getEntityCountForPeriod(tenantId: string): Promise<number> {
    const startOfMonth = new Date(new Date().getFullYear(), new Date().getMonth(), 1);
    let count = 0;
    for (const item of this.items.values()) {
      if (item.tenantId === tenantId && item.createdAt >= startOfMonth) count++;
    }
    return count;
  }

  // --- Settings ---
  private settingKey(key: string, tenantId: string | null): string { return `${tenantId || "SYSTEM"}:${key}`; }
  async getSetting(key: string, tenantId: string | null): Promise<string | null> {
    const s = this.settings.get(this.settingKey(key, tenantId));
    return s?.value ?? null;
  }
  async setSetting(key: string, value: string, isEncrypted: boolean, tenantId: string | null): Promise<void> {
    this.settings.set(this.settingKey(key, tenantId), { value, isEncrypted });
  }
  async getAllSettings(tenantId: string | null): Promise<Record<string, string>> {
    const prefix = `${tenantId || "SYSTEM"}:`;
    const result: Record<string, string> = {};
    for (const [k, v] of this.settings) {
      if (k.startsWith(prefix)) result[k.slice(prefix.length)] = v.value;
    }
    return result;
  }
  async updateSystemSettings(settingsData: Record<string, string>, tenantId: string): Promise<void> {
    for (const [key, value] of Object.entries(settingsData)) {
      await this.setSetting(key, value, false, tenantId);
    }
  }

  // --- Tokens ---
  async createToken(data: InsertToken): Promise<Token> {
    const token: Token = { id: this.nextId(), createdAt: new Date(), ...data };
    this.tokens.set(token.id, token);
    return token;
  }
  async getPendingTokensByType(type: Token["type"]): Promise<Token[]> {
    return Array.from(this.tokens.values()).filter(t => t.type === type && t.status === "pending");
  }
  async getPendingTokensByPrefix(prefix: string, type: Token["type"]): Promise<Token[]> {
    return Array.from(this.tokens.values())
      .filter(t => t.status === "pending" && t.type === type && t.tokenPrefix === prefix);
  }
  async updateTokenStatus(id: string, status: string, usedAt?: Date): Promise<void> {
    const t = this.tokens.get(id);
    if (t) this.tokens.set(id, { ...t, status: status as Token["status"], usedAt });
  }
  async revokePendingTokens(email: string, tenantId: string, type: string): Promise<void> {
    for (const [id, t] of this.tokens) {
      const emailMatch = t.email === email && t.type === type && t.status === "pending";
      const tenantMatch = !tenantId || t.tenantId === tenantId;  // Lege string = alle tenants
      if (emailMatch && tenantMatch) {
        this.tokens.set(id, { ...t, status: "revoked" });
      }
    }
  }
  async getTenantInvitations(tenantId: string): Promise<InvitationStatus[]> {
    return Array.from(this.tokens.values())
      .filter(t => t.tenantId === tenantId && t.type === "invitation") as InvitationStatus[];
  }

  // --- Password history ---
  async addPasswordToHistory(userId: string, passwordHash: string): Promise<void> {
    const history = this.passwordHistory.get(userId) || [];
    history.unshift({ hash: passwordHash, createdAt: new Date() });
    this.passwordHistory.set(userId, history);
  }
  async getPasswordHistory(userId: string, limit = 5): Promise<string[]> {
    return (this.passwordHistory.get(userId) || []).slice(0, limit).map(h => h.hash);
  }

  // --- Activity log ---
  async createActivityLog(data: InsertActivityLog): Promise<ActivityLog> {
    const log: ActivityLog = { id: this.nextId(), timestamp: new Date(), ...data };
    this.activityLogs.push(log);
    return log;
  }
  async getActivityLogs(tenantId: string | null, options?: { userId?: string }): Promise<ActivityLog[]> {
    return this.activityLogs.filter(l =>
      (tenantId === null || l.tenantId === tenantId) &&
      (!options?.userId || l.userId === options.userId)
    );
  }

  // --- Stripe subscriptions ---
  async getStripeSubscription(tenantId: string): Promise<StripeSubscription | undefined> {
    return Array.from(this.stripeSubscriptions.values()).find(s => s.tenantId === tenantId);
  }
  async upsertStripeSubscription(data: InsertStripeSubscription): Promise<StripeSubscription> {
    const existing = await this.getStripeSubscription(data.tenantId);
    const sub: StripeSubscription = {
      id: existing?.id || this.nextId(),
      createdAt: existing?.createdAt || new Date(), updatedAt: new Date(),
      cancelAtPeriodEnd: false, ...data,
    };
    this.stripeSubscriptions.set(sub.id, sub);
    return sub;
  }
  async updateStripeSubscription(tenantId: string, updates: Partial<StripeSubscription>): Promise<StripeSubscription | undefined> {
    const sub = await this.getStripeSubscription(tenantId);
    if (!sub) return undefined;
    const updated = { ...sub, ...updates, updatedAt: new Date() };
    this.stripeSubscriptions.set(sub.id, updated);
    return updated;
  }

  // --- Chat threads ---
  async getChatThreads(userId: string, tenantId: string): Promise<ChatThread[]> {
    return Array.from(this.chatThreads.values())
      .filter(t => t.userId === userId && t.tenantId === tenantId);
  }
  async getChatThread(id: string, tenantId: string): Promise<ChatThread | undefined> {
    const t = this.chatThreads.get(id);
    return t && t.tenantId === tenantId ? t : undefined;
  }
  async createChatThread(data: InsertChatThread): Promise<ChatThread> {
    const thread: ChatThread = { id: this.nextId(), createdAt: new Date(), updatedAt: new Date(), ...data };
    this.chatThreads.set(thread.id, thread);
    return thread;
  }
  async deleteChatThread(id: string, tenantId: string): Promise<boolean> {
    const t = this.chatThreads.get(id);
    if (!t || t.tenantId !== tenantId) return false;
    return this.chatThreads.delete(id);
  }
  async getChatMessages(threadId: string, tenantId: string): Promise<ChatMessage[]> {
    const thread = await this.getChatThread(threadId, tenantId);
    if (!thread) return [];
    return Array.from(this.chatMessages.values()).filter(m => m.threadId === threadId);
  }
  async createChatMessage(data: InsertChatMessage, tenantId: string): Promise<ChatMessage> {
    const msg: ChatMessage = { id: this.nextId(), createdAt: new Date(), ...data };
    this.chatMessages.set(msg.id, msg);
    return msg;
  }

  // --- Webhook deduplicatie ---
  async isWebhookProcessed(webhookId: string): Promise<boolean> { return this.processedWebhookIds.has(webhookId); }
  async markWebhookProcessed(webhookId: string, _source?: string): Promise<void> { this.processedWebhookIds.add(webhookId); }

  // --- Lifecycle ---
  async healthCheck(): Promise<{ ok: boolean }> { return { ok: true }; }
  async close(): Promise<void> { /* no-op for MemStorage */ }
}

// Singleton: switch tussen MemStorage en SupabaseStorage op basis van DATABASE_URL
import { SupabaseStorage } from "./supabase-storage";  // §6: productie storage met RLS

export const storage: IStorage = process.env.DATABASE_URL
  ? new SupabaseStorage()
  : new MemStorage();
```

### Paginatie patroon (optioneel — vervangt basic `getItems`)

Voor endpoints die veel data retourneren (items lijst, activity logs), vervang de basic `getItems` methode door een gepagineerde versie. **Dit is een upgrade** — begin met de simpele variant uit IStorage en voeg paginatie toe wanneer je >100 records verwacht:

```typescript
// === Query parameter type ===
export interface PaginationParams {
  limit?: number;   // Aantal items (default: 50, max: 200)
  offset?: number;  // Skip items (default: 0)
}

export interface PaginatedResult<T> {
  items: T[];
  total: number;
  limit: number;
  offset: number;
  hasMore: boolean;
}

// === Storage methode ===
async getItems(tenantId: string | null, pagination?: PaginationParams): Promise<PaginatedResult<Item>> {
  const limit = Math.min(pagination?.limit || 50, 200);  // Max 200 per request
  const offset = pagination?.offset || 0;

  const query = async (sql: any) => {
    const [countResult] = await sql`SELECT COUNT(*)::int AS total FROM items`;
    const rows = await sql`SELECT * FROM items ORDER BY created_at DESC LIMIT ${limit} OFFSET ${offset}`;
    return {
      items: rows, total: countResult.total, limit, offset,
      hasMore: offset + limit < countResult.total,
    };
  };
  // Dezelfde null-check als alle andere methoden: null = superadmin bypass
  return tenantId ? this.withTenant(tenantId, query) : this.withoutTenant(query);
}

// === Route handler ===
router.get("/api/items", requireAuth, requireTenant, asyncHandler(async (req, res) => {
  const tenantId = getActiveTenantId(req);
  const limit = Math.min(parseInt(req.query.limit as string) || 50, 200);
  const offset = parseInt(req.query.offset as string) || 0;
  const result = await storage.getItems(tenantId, { limit, offset });
  res.json(result);
}));

// === Frontend hook ===
function usePaginatedItems(page: number, pageSize = 50) {
  return useQuery({
    queryKey: ["/api/items", `?limit=${pageSize}&offset=${page * pageSize}`],
  });
}
```

---

## 5. Multi-Tenancy (App Layer)

> **Principes (niet-onderhandelbaar)**
> - `tenantId` is **altijd verplicht** in elke storage call. Type is `string | null`, NOOIT optioneel.
> - `null` = expliciete superadmin/system bypass. Nooit per ongeluk.
> - Elke API route haalt tenantId uit de **session** — NOOIT uit request parameters of query strings.
> - File storage paden bevatten ALTIJD een tenant prefix (`uploads/{tenantId}/...`).
> - Eén user kan lid zijn van meerdere tenants (many-to-many via `user_tenants`).

### Kernprincipe
`tenantId` is **altijd verplicht** in elke storage call. Het type is `string | null`:
- `string` = normale tenant-scoped operatie (met RLS)
- `null` = **expliciete** superadmin/system bypass (zonder RLS)

### Session bevat tenant context
```typescript
// Bij login: laad alle tenants van de user
const userTenants = await storage.getUserTenants(user.id);
const sessionUser: SessionUser = {
  id: user.id,
  username: user.username,
  globalRole: user.globalRole as GlobalRole,
  activeTenantId: userTenants.length > 0 ? userTenants[0].tenantId : undefined,
  tenants: userTenants.map(ut => ({
    id: ut.tenantId, name: ut.tenantName, slug: ut.tenantSlug, role: ut.role,
  })),
  // ...
};
```

### Route pattern: altijd tenantId uit session
```typescript
// Gebruik de middleware helpers uit §7 (requireAuth, requireTenant, asyncHandler)
router.get("/api/items", requireAuth, requireTenant, asyncHandler(async (req, res) => {
  const tenantId = getActiveTenantId(req);
  const items = await storage.getItems(tenantId);
  res.json(items);
}));
```

### Superadmin bypass pattern
```typescript
// Superadmin kan cross-tenant data zien
const tenantId = user.globalRole === "superadmin" ? null : user.activeTenantId;
const item = await storage.getItem(id, tenantId);
```

### Tenant switching endpoint
De volledige implementatie staat in `setupAuth()` (§7). Hier het patroon:
```typescript
app.post("/api/auth/switch-tenant", requireAuth, async (req, res) => {
  const user = getAuthUser(req);
  const { tenantId } = req.body;
  // Valideer dat user lid is van deze tenant
  const membership = user.tenants.find(t => t.id === tenantId);
  if (!membership) return res.status(403).json({ error: "Geen lid van deze tenant" });
  // Controleer of tenant actief is (niet gedeactiveerd door Stripe annulering)
  const tenant = await storage.getTenant(tenantId);
  if (!tenant) return res.status(404).json({ error: "Tenant niet gevonden" });
  if (tenant.isActive === false && user.globalRole !== "superadmin") {
    return res.status(403).json({ error: "Deze organisatie is niet meer actief" });
  }
  // Update session
  req.session.user = { ...user, activeTenantId: tenantId };
  req.session.save(() => res.json({ success: true }));
});
```

### File storage: tenant prefix
```typescript
// ALTIJD tenant prefix in file paden
const path = `uploads/${tenantId}/${filename}`;
// NOOIT: `uploads/${filename}` — data lekt tussen tenants
```

### withTenant beslisregel

In `SupabaseStorage` zijn er twee query-paden. De beslisregel is simpel:

| Methode-categorie | Query-pad | Reden |
|-------------------|-----------|-------|
| Items, chats, settings, en andere tenant-data | `withTenant(tenantId, async (sql) => {...})` | Data is tenant-scoped, RLS filtert |
| Tenants, users, tokens, password history, global secrets | Direct via `this.sql` (geen RLS) | Globale/systeem data, niet tenant-gebonden |

```typescript
// TENANT-SCOPED: item data is van één tenant
async getItems(tenantId: string | null): Promise<Item[]> {
  if (tenantId) {
    return this.withTenant(tenantId, async (sql) => {
      return await sql`SELECT * FROM items ORDER BY created_at DESC`;
      // RLS filtert automatisch op tenant — geen WHERE tenant_id nodig (maar mag wel voor clarity)
    });
  }
  // null = superadmin bypass, geen RLS
  return this.withoutTenant(async (sql) => {
    return await sql`SELECT * FROM items ORDER BY created_at DESC`;
  });
}

// GLOBAAL: users bestaan buiten tenant context
async getUser(id: string): Promise<User | undefined> {
  const rows = await this.sql`SELECT * FROM users WHERE id = ${id}`;
  return rows[0];
  // Geen withTenant — users tabel heeft allow_all RLS policy
}
```

**Vuistregel:** Als de tabel een `tenant_id` kolom heeft die in de RLS policy zit → `withTenant()`. Anders → directe query (`this.sql`).

**`withoutTenant()` mag ALLEEN in deze gevallen:**
- Superadmin-endpoints waar `tenantId === null` (cross-tenant data)
- System-level operaties (scheduled jobs, webhook handlers die nog geen tenant kennen)
- Tabellen zonder tenant_id (users, tenants, tokens, stripe_subscriptions)

Als je twijfelt → gebruik `withTenant()`. Het is altijd veiliger.

### Veelgemaakte fouten
1. **tenantId optioneel maken** → Maak het `string | null` (verplicht, null = bewuste bypass).
2. **tenantId vergeten in nieuwe endpoints** → Gebruik middleware: `requireTenant`.
3. **File storage zonder tenant prefix** → Bestanden zijn zichtbaar voor andere tenants.

---

## 6. Row Level Security (Database Layer)

> **Principes (niet-onderhandelbaar)**
> - RLS is de **tweede verdedigingslinie**. De app-layer (§5) is de eerste.
> - Fail-closed: geen tenant context gezet = geen data geretourneerd (NOOIT open-by-default).
> - `FORCE ROW LEVEL SECURITY` op elke tabel met `tenant_id` — voorkom table-owner bypass.
> - `prepare: false` in de postgres client configuratie — prepared statements cachen tenant context.
> - Indirecte tabellen (bijv. `chat_messages` → `chat_threads`) moeten via JOIN filteren op tenant.

### Kernprincipe
RLS is de **tweede verdedigingslinie**. Als de app-layer een bug heeft, voorkomt RLS dat data lekt. Fail-closed: geen context gezet = geen data.

### Stap 1: PostgreSQL functies
```sql
-- Tenant context instellen (transaction-local, auto-cleanup)
CREATE OR REPLACE FUNCTION set_tenant_context(tenant_id TEXT)
RETURNS VOID AS $$
BEGIN
  PERFORM set_config('app.current_tenant_id', tenant_id, true);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Tenant ophalen (NULL als niet gezet)
CREATE OR REPLACE FUNCTION get_current_tenant_id()
RETURNS TEXT AS $$
BEGIN
  RETURN NULLIF(current_setting('app.current_tenant_id', true), '');
EXCEPTION
  WHEN undefined_object THEN RETURN NULL;
END;
$$ LANGUAGE plpgsql STABLE;

-- Check of rij bij huidige tenant hoort (FAIL-CLOSED)
-- Parameter is TEXT, kolommen zijn UUID — PostgreSQL cast UUID naar TEXT impliciet.
-- Dit werkt omdat set_config() altijd TEXT opslaat en de vergelijking string-based is.
CREATE OR REPLACE FUNCTION tenant_matches(row_tenant_id TEXT)
RETURNS BOOLEAN AS $$
BEGIN
  IF get_current_tenant_id() IS NULL THEN
    RETURN FALSE;  -- Geen context = geen toegang
  END IF;
  RETURN row_tenant_id = get_current_tenant_id();
END;
$$ LANGUAGE plpgsql STABLE;
```

### Stap 2: RLS per tabel
```sql
-- Directe tenant_id kolom (pas toe op elke tabel met tenant_id)
ALTER TABLE items ENABLE ROW LEVEL SECURITY;
ALTER TABLE items FORCE ROW LEVEL SECURITY;  -- Voorkom bypass door table owner!
CREATE POLICY "tenant_isolation" ON items
  FOR ALL USING (tenant_matches(tenant_id)) WITH CHECK (tenant_matches(tenant_id));

-- Indirecte tenant via FK (bijv. chat_messages → chat_threads → tenant_id)
ALTER TABLE chat_messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_messages FORCE ROW LEVEL SECURITY;
CREATE POLICY "tenant_isolation" ON chat_messages
  FOR ALL
  USING (EXISTS (
    SELECT 1 FROM chat_threads t WHERE t.id = chat_messages.thread_id AND tenant_matches(t.tenant_id)
  ))
  WITH CHECK (EXISTS (
    SELECT 1 FROM chat_threads t WHERE t.id = chat_messages.thread_id AND tenant_matches(t.tenant_id)
  ));

-- Globale tabellen (users, tenants) — geen filtering, app-layer controleert
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY "allow_all" ON users FOR ALL USING (true) WITH CHECK (true);

-- Tabellen met optionele tenant (settings, tokens)
CREATE POLICY "tenant_or_null" ON settings
  FOR ALL
  USING (tenant_id IS NULL OR tenant_matches(tenant_id))
  WITH CHECK (tenant_id IS NULL OR tenant_matches(tenant_id));
```

### Stap 3: Application-layer `withTenant()`
```typescript
// server/supabase-storage.ts
import postgres from "postgres";

class SupabaseStorage implements IStorage {
  private sql: ReturnType<typeof postgres>;

  constructor() {
    this.sql = postgres(process.env.DATABASE_URL!, {
      ssl: { rejectUnauthorized: false },
      max: parseInt(process.env.DB_POOL_MAX || "10"),
      prepare: false,  // KRITIEK: verplicht voor RLS met connection pooling!
    });
  }

  async close(): Promise<void> {
    await this.sql.end();
  }

  // Tenant-scoped query
  // Als ENABLE_RLS=true: zet tenant context in transactie (RLS filtert)
  // Als ENABLE_RLS niet gezet: voeg WHERE tenant_id = ... toe aan queries (app-layer filtering)
  // Je kunt RLS pas activeren NADAT de migration (§6 stap 1+2) is uitgevoerd.
  private rlsEnabled = process.env.ENABLE_RLS === "true";

  async withTenant<T>(tenantId: string, query: (sql: any) => Promise<T>): Promise<T> {
    if (this.rlsEnabled) {
      // RLS actief: zet tenant context in transactie
      return await this.sql.begin(async (tx) => {
        await tx`SELECT set_config('app.current_tenant_id', ${tenantId}, true)`;
        return await query(tx);
      });
    }
    // RLS nog niet actief: gewone query (app-layer filtering via WHERE clauses)
    // In dit geval MOET je WHERE tenant_id = ${tenantId} in elke query hebben!
    return await query(this.sql);
  }

  // Cross-tenant query (superadmin/system)
  async withoutTenant<T>(query: (sql: any) => Promise<T>): Promise<T> {
    return await query(this.sql);
  }

  // Gebruik:
  async getItem(id: string, tenantId: string | null): Promise<Item | undefined> {
    if (tenantId) {
      // Tenant-scoped: RLS filtert automatisch
      return this.withTenant(tenantId, async (sql) => {
        const rows = await sql`SELECT * FROM items WHERE id = ${id}`;
        return rows[0];
      });
    } else {
      // Superadmin: geen RLS
      return this.withoutTenant(async (sql) => {
        const rows = await sql`SELECT * FROM items WHERE id = ${id}`;
        return rows[0];
      });
    }
  }
}
```

### Meer SupabaseStorage methoden

Hieronder staan alle resterende IStorage methoden voor de productie-implementatie. Ze dekken elk query-patroon dat je nodig hebt: upserts, joins, encryptie, JSONB, globale vs tenant-scoped queries, en bulk-operaties.

```typescript
import { encrypt, decrypt } from "../utils/encryption"; // Zie §21 voor implementatie

// === 1. UPSERT: Stripe subscription opslaan (globaal, geen withTenant) ===
async upsertStripeSubscription(sub: InsertStripeSubscription): Promise<StripeSubscription> {
  const existing = await this.sql`
    SELECT id FROM stripe_subscriptions WHERE tenant_id = ${sub.tenantId} LIMIT 1`;
  if (existing.length > 0) {
    const rows = await this.sql`
      UPDATE stripe_subscriptions SET
        stripe_customer_id = ${sub.stripeCustomerId},
        stripe_subscription_id = ${sub.stripeSubscriptionId},
        stripe_price_id = ${sub.stripePriceId ?? null},
        status = ${sub.status}, tier = ${sub.tier},
        billing_interval = ${sub.billingInterval ?? null},
        current_period_start = ${sub.currentPeriodStart},
        current_period_end = ${sub.currentPeriodEnd},
        cancel_at_period_end = ${sub.cancelAtPeriodEnd ?? false},
        canceled_at = ${sub.canceledAt ?? null},
        trial_started_at = ${sub.trialStartedAt ?? null},
        trial_ends_at = ${sub.trialEndsAt ?? null},
        welcome_email_sent_at = COALESCE(${sub.welcomeEmailSentAt ?? null}, welcome_email_sent_at),
        updated_at = NOW()
      WHERE tenant_id = ${sub.tenantId} RETURNING *`;
    return rows[0] as StripeSubscription;
  }
  const rows = await this.sql`
    INSERT INTO stripe_subscriptions ${this.sql(sub)} RETURNING *`;
  return rows[0] as StripeSubscription;
}

// === 2. JOIN: Users met rollen voor een tenant (globaal) ===
async getUsersForTenant(tenantId: string): Promise<Array<{ id: string; role: string }>> {
  return await this.sql`
    SELECT u.id, ut.role
    FROM users u
    JOIN user_tenants ut ON u.id = ut.user_id
    WHERE ut.tenant_id = ${tenantId} AND u.is_active = true
    ORDER BY ut.role ASC, u.display_name ASC`;
}

// === 3. SETTING met encryptie (tenant-scoped via withTenant) ===
async setSetting(key: string, value: string, isEncrypted: boolean, tenantId: string | null): Promise<void> {
  const storedValue = isEncrypted ? encrypt(value) : value;
  if (tenantId) {
    await this.withTenant(tenantId, async (sql) => {
      await sql`
        INSERT INTO settings (tenant_id, key, value, is_encrypted, updated_at)
        VALUES (${tenantId}, ${key}, ${storedValue}, ${isEncrypted}, NOW())
        ON CONFLICT (tenant_id, key)
        DO UPDATE SET value = ${storedValue}, is_encrypted = ${isEncrypted}, updated_at = NOW()`;
    });
  } else {
    await this.sql`
      INSERT INTO settings (tenant_id, key, value, is_encrypted, updated_at)
      VALUES (NULL, ${key}, ${storedValue}, ${isEncrypted}, NOW())
      ON CONFLICT (tenant_id, key)
      DO UPDATE SET value = ${storedValue}, is_encrypted = ${isEncrypted}, updated_at = NOW()`;
  }
}

// === 4. SETTING ophalen met decryptie ===
async getSetting(key: string, tenantId: string | null): Promise<string | null> {
  let rows;
  if (tenantId) {
    rows = await this.withTenant(tenantId, async (sql) => {
      return await sql`SELECT value, is_encrypted FROM settings WHERE key = ${key} AND tenant_id = ${tenantId}`;
    });
  } else {
    rows = await this.sql`SELECT value, is_encrypted FROM settings WHERE key = ${key} AND tenant_id IS NULL`;
  }
  if (rows.length === 0) return null;
  return rows[0].is_encrypted ? decrypt(rows[0].value) : rows[0].value;
}

async getAllSettings(tenantId: string | null): Promise<Record<string, string>> {
  let rows;
  if (tenantId) {
    rows = await this.withTenant(tenantId, async (sql) => {
      return await sql`SELECT key, value, is_encrypted FROM settings WHERE tenant_id = ${tenantId}`;
    });
  } else {
    rows = await this.sql`SELECT key, value, is_encrypted FROM settings WHERE tenant_id IS NULL`;
  }
  const result: Record<string, string> = {};
  for (const row of rows) {
    result[row.key] = row.is_encrypted ? decrypt(row.value) : row.value;
  }
  return result;
}

// === 5. TENANT-SCOPED INSERT (voorbeeld: item aanmaken) ===
async createItem(item: InsertItem): Promise<Item> {
  return this.withTenant(item.tenantId, async (sql) => {
    const rows = await sql`
      INSERT INTO items (tenant_id, title, status, metadata)
      VALUES (${item.tenantId}, ${item.title}, ${item.status || 'pending'},
              ${item.metadata ? sql.json(item.metadata) : null})
      RETURNING *`;
    return rows[0] as Item;
  });
}

// === 6. UPDATE met JSONB en conditionele velden ===
// BELANGRIJK: De postgres library ondersteunt geen dynamische SET via tagged templates.
// Gebruik een expliciete query per combinatie, of een helper die raw SQL bouwt.
async updateItem(id: string, updates: Partial<Item>, tenantId: string | null): Promise<Item | undefined> {
  const query = async (sql: any) => {
    // Optie A: Alle velden meegeven (eenvoudigst, werkt altijd)
    // COALESCE-patroon: ongewijzigde velden behouden hun huidige waarde
    // LET OP: COALESCE kan velden NIET naar NULL resetten (null = "niet meegegeven").
    // Als je een veld expliciet naar NULL moet zetten (bijv. locked_until na unlock),
    // gebruik dan Optie B hieronder of een aparte UPDATE query.
    const rows = await sql`
      UPDATE items SET
        title = COALESCE(${updates.title ?? null}, title),
        status = COALESCE(${updates.status ?? null}, status),
        metadata = COALESCE(${updates.metadata ? sql.json(updates.metadata) : null}, metadata),
        updated_at = NOW()
      WHERE id = ${id}
      RETURNING *`;
    return rows[0] as Item | undefined;

    // Optie B: Per-veld queries (als je veel optionele velden hebt)
    // Minder elegant maar altijd correct:
    //
    // if (updates.title !== undefined) {
    //   await sql`UPDATE items SET title = ${updates.title}, updated_at = NOW() WHERE id = ${id}`;
    // }
    // if (updates.metadata !== undefined) {
    //   await sql`UPDATE items SET metadata = ${sql.json(updates.metadata)}, updated_at = NOW() WHERE id = ${id}`;
    // }
    // const rows = await sql`SELECT * FROM items WHERE id = ${id}`;
    // return rows[0];
  };
  return tenantId ? this.withTenant(tenantId, query) : this.withoutTenant(query);
}

// === 7. ITEMS LIJST (tenant-scoped, dezelfde null-check als getItem) ===
async getItems(tenantId: string | null): Promise<Item[]> {
  const query = async (sql: any) => await sql`SELECT * FROM items ORDER BY created_at DESC`;
  return tenantId ? this.withTenant(tenantId, query) : this.withoutTenant(query);
}

async deleteItem(id: string, tenantId: string | null): Promise<boolean> {
  const query = async (sql: any) => {
    const rows = await sql`DELETE FROM items WHERE id = ${id} RETURNING id`;
    return rows.length > 0;
  };
  return tenantId ? this.withTenant(tenantId, query) : this.withoutTenant(query);
}

async getEntityCountForPeriod(tenantId: string): Promise<number> {
  return this.withTenant(tenantId, async (sql) => {
    const [row] = await sql`
      SELECT COUNT(*)::int AS count FROM items
      WHERE created_at >= date_trunc('month', NOW())`;
    return row.count;
  });
}

// === 8. GLOBALE METHODEN (gebruiken this.sql direct, geen withTenant) ===
// Users, tenants, tokens, password history — niet tenant-scoped

async getUser(id: string): Promise<User | undefined> {
  const rows = await this.sql`SELECT * FROM users WHERE id = ${id}`;
  return rows[0] as User | undefined;
}

async getUserByEmail(email: string): Promise<User | undefined> {
  const rows = await this.sql`SELECT * FROM users WHERE LOWER(email) = ${email.toLowerCase()}`;
  return rows[0] as User | undefined;
}

async getUserByUsername(username: string): Promise<User | undefined> {
  const rows = await this.sql`SELECT * FROM users WHERE username = ${username}`;
  return rows[0] as User | undefined;
}

async createUser(username: string, passwordHash: string, globalRole?: GlobalRole): Promise<User> {
  const rows = await this.sql`
    INSERT INTO users (username, password_hash, global_role)
    VALUES (${username}, ${passwordHash}, ${globalRole || "standard"})
    RETURNING *`;
  return rows[0] as User;
}

async createUserWithDetails(details: {
  username: string; email: string; passwordHash: string;
  displayName?: string; globalRole: GlobalRole; isActive: boolean; mustChangePassword: boolean;
}): Promise<User> {
  const rows = await this.sql`
    INSERT INTO users (username, email, password_hash, display_name, global_role, is_active, must_change_password)
    VALUES (${details.username}, ${details.email}, ${details.passwordHash},
            ${details.displayName || null}, ${details.globalRole}, ${details.isActive},
            ${details.mustChangePassword})
    RETURNING *`;
  return rows[0] as User;
}

async updateUserDetails(id: string, updates: Partial<User>): Promise<User | undefined> {
  // Gebruik de postgres library's object-to-columns helper voor dynamische updates.
  // Dit lost het COALESCE-NULL-reset probleem op: alle meegegeven velden worden direct gezet.
  // Velden die NIET in updates zitten, worden niet aangeraakt.
  const dbUpdates: Record<string, unknown> = {};
  if (updates.displayName !== undefined) dbUpdates.display_name = updates.displayName;
  if (updates.passwordHash !== undefined) dbUpdates.password_hash = updates.passwordHash;
  if (updates.isActive !== undefined) dbUpdates.is_active = updates.isActive;
  if (updates.mustChangePassword !== undefined) dbUpdates.must_change_password = updates.mustChangePassword;
  if (updates.failedLoginAttempts !== undefined) dbUpdates.failed_login_attempts = updates.failedLoginAttempts;
  if (updates.lockedUntil !== undefined) dbUpdates.locked_until = updates.lockedUntil;  // null = unlock
  if (updates.lastLoginAt !== undefined) dbUpdates.last_login_at = updates.lastLoginAt;
  if (Object.keys(dbUpdates).length === 0) {
    const rows = await this.sql`SELECT * FROM users WHERE id = ${id}`;
    return rows[0] as User | undefined;
  }
  const rows = await this.sql`
    UPDATE users SET ${this.sql(dbUpdates, ...Object.keys(dbUpdates))}
    WHERE id = ${id} RETURNING *`;
  return rows[0] as User | undefined;
}

async deleteUser(id: string): Promise<boolean> {
  const rows = await this.sql`DELETE FROM users WHERE id = ${id} RETURNING id`;
  return rows.length > 0;
}

// === 9. TENANT MANAGEMENT ===
async getTenant(id: string): Promise<Tenant | undefined> {
  const rows = await this.sql`SELECT * FROM tenants WHERE id = ${id}`;
  return rows[0] as Tenant | undefined;
}

async getTenantBySlug(slug: string): Promise<Tenant | undefined> {
  const rows = await this.sql`SELECT * FROM tenants WHERE slug = ${slug}`;
  return rows[0] as Tenant | undefined;
}

async createTenant(tenant: InsertTenant): Promise<Tenant> {
  const rows = await this.sql`INSERT INTO tenants ${this.sql(tenant)} RETURNING *`;
  return rows[0] as Tenant;
}

async updateTenant(id: string, updates: Partial<Tenant>): Promise<Tenant | undefined> {
  const rows = await this.sql`
    UPDATE tenants SET
      name = COALESCE(${updates.name ?? null}, name),
      is_active = COALESCE(${updates.isActive ?? null}, is_active),
      billing_exempt = COALESCE(${updates.billingExempt ?? null}, billing_exempt)
    WHERE id = ${id} RETURNING *`;
  return rows[0] as Tenant | undefined;
}

// === 10. USER-TENANT RELATIES ===
async getUserTenants(userId: string): Promise<UserTenantWithDetails[]> {
  return await this.sql`
    SELECT ut.tenant_id, ut.role, ut.created_at, t.name AS tenant_name, t.slug AS tenant_slug
    FROM user_tenants ut
    JOIN tenants t ON ut.tenant_id = t.id
    WHERE ut.user_id = ${userId} AND t.is_active = true
    ORDER BY t.name ASC`;
}

async addUserToTenant(userId: string, tenantId: string, role?: TenantRole): Promise<UserTenant> {
  const rows = await this.sql`
    INSERT INTO user_tenants (user_id, tenant_id, role)
    VALUES (${userId}, ${tenantId}, ${role || "member"})
    ON CONFLICT (user_id, tenant_id) DO UPDATE SET role = ${role || "member"}
    RETURNING *`;
  return rows[0] as UserTenant;
}

async removeUserFromTenant(userId: string, tenantId: string): Promise<{ success: boolean }> {
  const rows = await this.sql`DELETE FROM user_tenants WHERE user_id = ${userId} AND tenant_id = ${tenantId} RETURNING id`;
  return { success: rows.length > 0 };
}

// === 11. TOKENS (globaal — niet tenant-scoped) ===
async createToken(token: InsertToken): Promise<Token> {
  const rows = await this.sql`INSERT INTO tokens ${this.sql(token)} RETURNING *`;
  return rows[0] as Token;
}

async getPendingTokensByType(type: Token["type"]): Promise<Token[]> {
  return await this.sql`
    SELECT * FROM tokens
    WHERE type = ${type} AND status = 'pending' AND expires_at > NOW()
    ORDER BY created_at DESC`;
}

async getPendingTokensByPrefix(prefix: string, type: Token["type"]): Promise<Token[]> {
  return await this.sql`
    SELECT * FROM tokens
    WHERE token_prefix = ${prefix} AND type = ${type} AND status = 'pending' AND expires_at > NOW()`;
}

async updateTokenStatus(id: string, status: string, usedAt?: Date): Promise<void> {
  await this.sql`UPDATE tokens SET status = ${status}, used_at = ${usedAt || new Date()} WHERE id = ${id}`;
}

async revokePendingTokens(email: string, tenantId: string, type: string): Promise<void> {
  if (tenantId) {
    await this.sql`
      UPDATE tokens SET status = 'revoked'
      WHERE email = ${email} AND tenant_id = ${tenantId} AND type = ${type} AND status = 'pending'`;
  } else {
    // Geen tenantId (bijv. password reset) → revoke voor alle tenants
    await this.sql`
      UPDATE tokens SET status = 'revoked'
      WHERE email = ${email} AND type = ${type} AND status = 'pending'`;
  }
}

async getTenantInvitations(tenantId: string): Promise<InvitationStatus[]> {
  return await this.sql`
    SELECT * FROM tokens
    WHERE type = 'invitation' AND status = 'pending' AND tenant_id = ${tenantId}
    AND expires_at > NOW() ORDER BY created_at DESC` as InvitationStatus[];
}

// === 12. PASSWORD HISTORY ===
async addPasswordToHistory(userId: string, passwordHash: string): Promise<void> {
  await this.sql`INSERT INTO password_history (user_id, password_hash) VALUES (${userId}, ${passwordHash})`;
}

async getPasswordHistory(userId: string, limit = 5): Promise<string[]> {
  const rows = await this.sql`
    SELECT password_hash FROM password_history
    WHERE user_id = ${userId}
    ORDER BY created_at DESC LIMIT ${limit}`;
  return rows.map((r: any) => r.password_hash);
}

// === 13. STRIPE (globaal) ===
async updateStripeSubscription(tenantId: string, updates: Partial<StripeSubscription>): Promise<StripeSubscription | undefined> {
  const rows = await this.sql`
    UPDATE stripe_subscriptions SET
      status = COALESCE(${updates.status ?? null}, status),
      tier = COALESCE(${updates.tier ?? null}, tier),
      cancel_at_period_end = COALESCE(${updates.cancelAtPeriodEnd ?? null}, cancel_at_period_end),
      canceled_at = COALESCE(${updates.canceledAt ?? null}, canceled_at),
      current_period_start = COALESCE(${updates.currentPeriodStart ?? null}, current_period_start),
      current_period_end = COALESCE(${updates.currentPeriodEnd ?? null}, current_period_end),
      updated_at = NOW()
    WHERE tenant_id = ${tenantId} RETURNING *`;
  return rows[0] as StripeSubscription | undefined;
}

async getStripeSubscription(tenantId: string): Promise<StripeSubscription | undefined> {
  const rows = await this.sql`SELECT * FROM stripe_subscriptions WHERE tenant_id = ${tenantId}`;
  return rows[0] as StripeSubscription | undefined;
}

// === 14. ACTIVITY LOG (tenant-scoped) ===
async createActivityLog(log: InsertActivityLog): Promise<ActivityLog> {
  // activity_log.tenant_id is TEXT (niet UUID) — kan "system" bevatten voor cross-tenant events
  const rows = await this.sql`INSERT INTO activity_log ${this.sql(log)} RETURNING *`;
  return rows[0] as ActivityLog;
}

async getActivityLogs(tenantId: string | null, options?: { userId?: string }): Promise<ActivityLog[]> {
  if (!tenantId) {
    // Superadmin: alle logs (geen RLS)
    const rows = options?.userId
      ? await this.sql`SELECT * FROM activity_log WHERE user_id = ${options.userId} ORDER BY timestamp DESC LIMIT 50`
      : await this.sql`SELECT * FROM activity_log ORDER BY timestamp DESC LIMIT 50`;
    return rows as ActivityLog[];
  }
  return this.withTenant(tenantId, async (sql) => {
    return options?.userId
      ? await sql`SELECT * FROM activity_log WHERE user_id = ${options.userId} ORDER BY timestamp DESC LIMIT 50`
      : await sql`SELECT * FROM activity_log ORDER BY timestamp DESC LIMIT 50`;
  });
}

// === 15. CHAT (tenant-scoped) ===
async getChatThreads(userId: string, tenantId: string): Promise<ChatThread[]> {
  return this.withTenant(tenantId, async (sql) => {
    return await sql`SELECT * FROM chat_threads WHERE user_id = ${userId} ORDER BY updated_at DESC`;
  });
}

async getChatThread(id: string, tenantId: string): Promise<ChatThread | undefined> {
  return this.withTenant(tenantId, async (sql) => {
    const rows = await sql`SELECT * FROM chat_threads WHERE id = ${id}`;
    return rows[0] as ChatThread | undefined;
  });
}

async createChatThread(thread: InsertChatThread): Promise<ChatThread> {
  return this.withTenant(thread.tenantId, async (sql) => {
    const rows = await sql`INSERT INTO chat_threads ${sql(thread)} RETURNING *`;
    return rows[0] as ChatThread;
  });
}

async deleteChatThread(id: string, tenantId: string): Promise<boolean> {
  return this.withTenant(tenantId, async (sql) => {
    const rows = await sql`DELETE FROM chat_threads WHERE id = ${id} RETURNING id`;
    return rows.length > 0;
  });
}

async getChatMessages(threadId: string, tenantId: string): Promise<ChatMessage[]> {
  return this.withTenant(tenantId, async (sql) => {
    return await sql`SELECT * FROM chat_messages WHERE thread_id = ${threadId} ORDER BY created_at ASC`;
  });
}

async createChatMessage(message: InsertChatMessage, tenantId: string): Promise<ChatMessage> {
  return this.withTenant(tenantId, async (sql) => {
    const rows = await sql`INSERT INTO chat_messages ${sql(message)} RETURNING *`;
    return rows[0] as ChatMessage;
  });
}

// === 16. WEBHOOK DEDUP + SYSTEM (globaal) ===
async isWebhookProcessed(webhookId: string): Promise<boolean> {
  const rows = await this.sql`SELECT 1 FROM processed_webhooks WHERE webhook_id = ${webhookId}`;
  return rows.length > 0;
}

async markWebhookProcessed(webhookId: string, source = "stripe"): Promise<void> {
  await this.sql`
    INSERT INTO processed_webhooks (webhook_id, source) VALUES (${webhookId}, ${source})
    ON CONFLICT (webhook_id) DO NOTHING`;
}

async updateSystemSettings(settings: Record<string, string>, tenantId: string): Promise<void> {
  // Bulk update: loop over alle key-value paren
  for (const [key, value] of Object.entries(settings)) {
    await this.withTenant(tenantId, async (sql) => {
      await sql`
        INSERT INTO settings (tenant_id, key, value, updated_at)
        VALUES (${tenantId}, ${key}, ${value}, NOW())
        ON CONFLICT (tenant_id, key) DO UPDATE SET value = ${value}, updated_at = NOW()`;
    });
  }
}

async healthCheck(): Promise<{ ok: boolean }> {
  await this.sql`SELECT 1`;
  return { ok: true };
}
```

**Patronen samengevat:**

| Patroon | Wanneer | Voorbeeld |
|---------|---------|-----------|
| `withTenant()` | Tabel met `tenant_id` in RLS policy | items, settings, chat_threads |
| Direct `this.sql` | Globale/systeem data | users, tenants, tokens, stripe_subscriptions |
| `ON CONFLICT ... DO UPDATE` | Upsert (idempotent) | settings, stripe_subscriptions |
| `sql.json()` | JSONB kolommen | metadata, context, citations |
| `encrypt()`/`decrypt()` | Gevoelige waarden | OAuth tokens, API keys |

**`this.sql(obj)` helper:** De `postgres` library ondersteunt object-to-columns mapping. `this.sql\`INSERT INTO tabel ${this.sql(obj)} RETURNING *\`` vertaalt een JavaScript object automatisch naar `INSERT INTO tabel (col1, col2) VALUES ($1, $2)`. De library doet automatisch camelCase → snake_case conversie. Dit werkt alleen met de `postgres` library (niet met `pg` of Knex).

### Veelgemaakte fouten
1. **`prepare: true` (default)** — Prepared statements cachen tenant context. ALTIJD `prepare: false`.
2. **`ENABLE` zonder `FORCE`** — Table owner kan RLS bypassen. ALTIJD `FORCE ROW LEVEL SECURITY`.
3. **Indirecte tabellen vergeten** — Chat messages, pipeline steps, etc. moeten via JOIN filteren.
4. **RLS te vroeg activeren** — Zet `ENABLE_RLS=true` pas NADAT je de SQL functies (stap 1) en policies (stap 2) hebt uitgevoerd. Zonder die functies geeft `set_config` een error.

### Activatievolgorde voor RLS

```
1. Implementeer app-layer multi-tenancy (§5) — tenantId in elke query
2. Draai SQL functies (§6 stap 1) — set_tenant_context, get_current_tenant_id
3. Draai RLS policies (§6 stap 2) — per tabel
4. Zet ENABLE_RLS=true in je environment
5. Test! (zie verificatietests hieronder)
```

### Verificatietests (KRITIEK — draai na elke wijziging)

Deze tests verifiëren dat tenant isolatie en security correct werken. Implementeer met vitest (of handmatig via psql).

```typescript
// server/__tests__/security.test.ts
import { describe, it, expect, beforeAll } from "vitest";
import { MemStorage } from "../storage";

describe("Tenant Isolation", () => {
  const storage = new MemStorage();
  let tenantA: string, tenantB: string;

  beforeAll(async () => {
    const tA = await storage.createTenant({ name: "Tenant A", slug: "tenant-a" });
    const tB = await storage.createTenant({ name: "Tenant B", slug: "tenant-b" });
    tenantA = tA.id;
    tenantB = tB.id;

    // Data aanmaken voor tenant A en B
    await storage.createItem({ title: "Item van A", tenantId: tenantA, status: "active" });
    await storage.createItem({ title: "Item van B", tenantId: tenantB, status: "active" });
  });

  it("tenant A kan ALLEEN eigen items zien", async () => {
    const items = await storage.getItems(tenantA);
    expect(items).toHaveLength(1);
    expect(items[0].title).toBe("Item van A");
  });

  it("tenant B kan NIET de items van tenant A zien", async () => {
    const items = await storage.getItems(tenantB);
    expect(items).toHaveLength(1);
    expect(items[0].title).toBe("Item van B");
  });

  it("getItem met verkeerd tenantId retourneert undefined", async () => {
    const items = await storage.getItems(tenantA);
    const itemA = items[0];
    // Probeer item van A op te halen met tenant B context
    const result = await storage.getItem(itemA.id, tenantB);
    expect(result).toBeUndefined();
  });

  it("null tenantId (superadmin) kan alles zien", async () => {
    const all = await storage.getItems(null);
    expect(all.length).toBeGreaterThanOrEqual(2);
  });
});
```

**RLS verificatie via SQL (na ENABLE_RLS=true):**

```sql
-- Test 1: Met tenant context → alleen eigen data
BEGIN;
SELECT set_config('app.current_tenant_id', 'tenant-a-uuid', true);
SELECT COUNT(*) FROM items;  -- Verwacht: alleen tenant A items
ROLLBACK;

-- Test 2: Zonder tenant context → GEEN data (fail-closed)
BEGIN;
-- Stel GEEN tenant context in
SELECT COUNT(*) FROM items;  -- Verwacht: 0 rijen
ROLLBACK;

-- Test 3: CSRF — POST zonder token → 403
-- curl -X POST http://localhost:5000/api/items -H "Content-Type: application/json" -d '{"title":"test"}'
-- Verwacht: {"error":"CSRF token mismatch","code":"AUTH_1004"}
```

---

# DEEL B: AUTHENTICATIE & SECURITY

---

## 7. Session Auth + CSRF

> **Principes (niet-onderhandelbaar)**
> - Sessions zijn httpOnly cookies, opgeslagen server-side (PostgreSQL). NOOIT JWT in localStorage.
> - CSRF token vereist op alle state-changing requests (POST/PUT/PATCH/DELETE).
> - Webhook routes registreren VOOR CSRF middleware — externe services sturen geen CSRF tokens.
> - Middleware registratie volgorde is KRITIEK: auth → webhooks → publiek → CSRF → beschermd → error handler.

### Kernprincipe
Session in httpOnly cookie, opgeslagen in PostgreSQL. CSRF via double-submit pattern. De **volgorde van middleware registratie is KRITIEK**.

**Heb je al passport.js?** Dat is prima — passport.js gebruikt ook express-session onder de hood. De sessie-structuur (`req.session.user`, `req.session.csrfToken`) en CSRF middleware werken identiek. Je kunt passport's `serializeUser`/`deserializeUser` behouden en de CSRF + middleware volgorde hieronder toevoegen.

### CSRF token generatie
```typescript
// Plaats in server/auth.ts — dezelfde file als setupAuth().
// generateCsrfToken() wordt aangeroepen BINNEN setupAuth() endpoints (login, MFA verify).
import crypto from "crypto";

function generateCsrfToken(): string {
  return crypto.randomBytes(32).toString("hex");
}
// Wordt aangemaakt bij: login, MFA verify, registratie, en session sync
```

### Express app setup (server/app.ts)

```typescript
import express from "express";
import helmet from "helmet";

export const app = express();

// === STAP 1: Security headers ===
app.use(helmet({
  contentSecurityPolicy: process.env.NODE_ENV === "production" ? {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https:"],
      frameSrc: ["'none'"],
      objectSrc: ["'none'"],
    },
  } : false,
}));

// === STAP 2: CORS (VOOR alle routes) ===
app.use((req, res, next) => {
  const origin = req.headers.origin;
  const allowedOrigins = process.env.CORS_ORIGIN?.split(",").map(o => o.trim()) || [];

  if (origin && (allowedOrigins.includes(origin) || process.env.NODE_ENV !== "production")) {
    res.header("Access-Control-Allow-Origin", origin);       // Specifiek, NOOIT "*"
    res.header("Access-Control-Allow-Credentials", "true");  // Cookies meesturen
  }
  res.header("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS");
  res.header("Access-Control-Allow-Headers", "Content-Type, X-CSRF-Token, X-Dev-Token");
  res.header("Access-Control-Expose-Headers", "X-CSRF-Token");  // Frontend kan header lezen
  if (req.method === "OPTIONS") return res.status(204).end();
  next();
});

// === STAP 3: Body parsing MET rawBody capture (voor webhook signatures) ===
// Type augmentation zodat rawBody beschikbaar is zonder `as any`
declare module 'http' {
  interface IncomingMessage { rawBody: Buffer; }
}
app.use(express.json({
  verify: (req, _res, buf) => {
    req.rawBody = buf;  // KRITIEK: bewaar originele bytes voor signature verificatie
  }
}));

// === STAP 4: Trust proxy (VERPLICHT achter Railway/Vercel/nginx) ===
app.set("trust proxy", 1);
```

### Session configuratie

De volledige session configuratie staat in `setupAuth()` hieronder (§7 "Login & Session Endpoints"). Daar vind je:
- Session store (connect-pg-simple)
- Cookie instellingen (sameSite, secure, httpOnly, maxAge)
- Deployment model keuze (Model A: same-origin vs Model B: cross-origin)

> **Deployment modellen (kies er één):**
> - **Model A: Same-origin** (aanbevolen) — backend serveert frontend, `sameSite: "lax"`, geen CORS nodig
> - **Model B: Cross-origin** — gescheiden frontend/backend domeinen, `sameSite: "none"` + `secure: true`, CORS verplicht
>
> De code in dit document toont Model A als default. Zie de comments in `setupAuth()` voor Model B instructies.

### Absolute session timeout middleware

De `maxAge` van een cookie is rolling — elke request verlengt de levensduur. Voeg een **absolute timeout** toe zodat sessies na een vaste periode verlopen, ongeacht activiteit:

```typescript
const SESSION_ABSOLUTE_TIMEOUT_MS = 30 * 24 * 60 * 60 * 1000; // 30 dagen

app.use((req, res, next) => {
  if (req.session?.user) {
    if (req.session.createdAt && Date.now() - req.session.createdAt > SESSION_ABSOLUTE_TIMEOUT_MS) {
      console.log(`Session expired (absolute timeout) for user ${req.session.user.id}`);
      req.session.destroy(() => {});
      return next();  // User moet opnieuw inloggen
    }
  }
  next();
});
```

`createdAt` wordt gezet bij login/register/MFA verify in de `session.regenerate()` callback (zie §7 login en §10 MFA).

### KRITIEKE route registratie volgorde (server/routes/index.ts)

```typescript
import { type Express } from "express";
import { createServer, type Server } from "http";
import { setupAuth } from "./auth";                    // §7: session + login/logout/session endpoints
import { stripeWebhookRouter } from "./stripe-webhook"; // §15/§22: Stripe webhook (aparte router)
import { publicAuthRouter } from "./public-auth";       // §9/§13: password reset, invitation accept
import { billingRouter, billingConfigHandler } from "./billing"; // §15: checkout, portal, status
import { itemsRouter } from "./items";                  // Jouw domein-routes
import { settingsRouter } from "./settings";
import { adminRouter } from "./admin";
import { gdprRouter } from "./gdpr";                    // §23: data export, deletion
import { errorHandler, asyncHandler } from "../utils/errors"; // §18
import { ApiError, ErrorCodes } from "../utils/errors";

export async function registerRoutes(app: Express): Promise<Server> {
  // ============================================================
  // FASE 1: Auth setup (sessions, login/logout/change-password endpoints)
  // ============================================================
  setupAuth(app);  // Exporteert uit server/auth.ts — zie hieronder

  // ============================================================
  // FASE 2: Webhook routes — VOOR CSRF middleware!
  // Externe services sturen geen CSRF tokens
  // ============================================================
  app.use(stripeWebhookRouter);   // POST /api/webhook/stripe
  // Voeg hier eigen webhook routes toe als nodig (bijv. voor externe integraties)

  // ============================================================
  // FASE 3: Publieke routes — geen auth nodig
  // ============================================================
  app.use(publicAuthRouter);      // /api/auth/accept-invitation, /api/auth/reset-password
  app.get("/api/billing/config", billingConfigHandler);

  // ============================================================
  // FASE 4: CSRF middleware — ALLE routes hierna zijn beschermd
  // ============================================================

  // 4a: Sync CSRF token van session naar response header
  app.use("/api", (req, res, next) => {
    if (req.session?.csrfToken) {
      res.setHeader("X-CSRF-Token", req.session.csrfToken);
    }
    next();
  });

  // 4b: Valideer CSRF token op state-changing requests
  app.use("/api", (req, res, next) => {
    const stateChanging = ["POST", "PUT", "PATCH", "DELETE"];
    if (!stateChanging.includes(req.method)) return next();

    const tokenFromHeader = req.headers["x-csrf-token"] as string;
    const tokenFromSession = req.session?.csrfToken;

    if (!tokenFromHeader || tokenFromHeader !== tokenFromSession) {
      return res.status(403).json({ error: "CSRF token mismatch", code: "AUTH_1004" });
    }
    next();
  });

  // 4c: Blokkeer als wachtwoord wijziging verplicht is
  app.use("/api", (req, res, next) => {
    const user = req.session?.user;
    if (!user?.mustChangePassword) return next();
    // Sta alleen essentiële endpoints toe
    const allowed = ["/api/auth/change-password", "/api/auth/logout", "/api/auth/session"];
    if (allowed.some(p => req.path.startsWith(p))) return next();
    return res.status(403).json(new ApiError(ErrorCodes.AUTH_PASSWORD_CHANGE_REQUIRED).toResponse());
  });

  // ============================================================
  // FASE 5: Beschermde routes — auth + CSRF + tenant vereist
  // ============================================================
  app.use(itemsRouter);           // Jouw domein-routes
  app.use(billingRouter);
  app.use(settingsRouter);
  app.use(adminRouter);
  app.use(gdprRouter);
  // etc.

  // ============================================================
  // FASE 6: Error handler — ALTIJD als laatste
  // ============================================================
  app.use(errorHandler);

  // ============================================================
  // FASE 7: HTTP Server aanmaken en retourneren
  // ============================================================
  const httpServer = createServer(app);
  return httpServer;
}
```

### setupAuth wrapper + Login endpoint (server/auth.ts)

De `setupAuth()` functie bundelt alle auth-gerelateerde middleware en endpoints. Dit is de functie die `registerRoutes()` aanroept.

```typescript
// server/auth.ts
import { type Express } from "express";
import session from "express-session";
import connectPgSimple from "connect-pg-simple";
import bcrypt from "bcrypt";
import crypto from "crypto";
import { storage } from "./storage";
import { loginSchema, type SessionUser, type GlobalRole, type TenantRole } from "@shared/schema";
import { logSecurityEvent, SecurityEvents } from "./utils/audit";       // §19
import { sendMfaCodeEmail } from "./services/email";                    // §20
import { ApiError, ErrorCodes } from "./utils/errors";                  // §18
import { loginLimiter, publicLimiter } from "./middleware/rate-limit";  // §11 (of stub in Fase 1)

export function setupAuth(app: Express) {
  const PgSession = connectPgSimple(session);
  const isProduction = process.env.NODE_ENV === "production";

  app.use(session({
    secret: process.env.SESSION_SECRET!,
    resave: false,
    saveUninitialized: false,
    name: "app.sid",
    cookie: isProduction
      ? { secure: true, httpOnly: true, sameSite: "lax", maxAge: 7 * 24 * 60 * 60 * 1000 }
      : { secure: false, httpOnly: true, sameSite: "lax", maxAge: 7 * 24 * 60 * 60 * 1000 },
    store: new PgSession({
      conString: process.env.DATABASE_URL,
      tableName: "session",
      createTableIfMissing: true,
    }),
  }));

  // Login endpoint
  app.post("/api/auth/login", loginLimiter, async (req, res) => {
  const parsed = loginSchema.safeParse(req.body);
  if (!parsed.success) return res.status(400).json({ error: "Ongeldige invoer" });
  const { email, password } = parsed.data;

  // 1. User opzoeken (case-insensitive)
  const user = await storage.getUserByEmail(email.toLowerCase());
  if (!user) return res.status(401).json({ error: "Ongeldig e-mailadres of wachtwoord" });

  // 2. Wachtwoord verifiëren
  const isValid = await bcrypt.compare(password, user.passwordHash);
  if (!isValid) {
    await logSecurityEvent(user.id, SecurityEvents.LOGIN_FAILURE, { success: false });
    return res.status(401).json({ error: "Ongeldig e-mailadres of wachtwoord" });
  }

  // 3. Account actief?
  if (!user.isActive) return res.status(403).json({ error: "Account is gedeactiveerd" });

  // 4. Tenants laden
  const userTenants = await storage.getUserTenants(user.id);

  // 5. SessionUser opbouwen
  const sessionUser: SessionUser = {
    id: user.id,
    username: user.username,
    email: user.email ?? undefined,
    displayName: user.displayName ?? undefined,
    globalRole: user.globalRole as GlobalRole,
    activeTenantId: userTenants.length > 0 ? userTenants[0].tenantId : undefined,
    tenants: userTenants.map(ut => ({
      id: ut.tenantId, name: ut.tenantName, slug: ut.tenantSlug, role: ut.role as TenantRole,
    })),
    mustChangePassword: user.mustChangePassword === true,
  };

  // 6. MFA check (als tenant dit vereist)
  if (sessionUser.activeTenantId) {
    const mfaSetting = await storage.getSetting("mfaRequired", sessionUser.activeTenantId);
    if (mfaSetting === "true" && user.email) {
      const code = crypto.randomInt(100000, 1000000).toString();  // Cryptografisch veilig
      req.session.mfaPending = true;
      req.session.mfaCodeHash = await bcrypt.hash(code, 10);
      req.session.mfaCodeExpiresAt = Date.now() + 10 * 60 * 1000;
      req.session.mfaUserId = user.id;
      req.session.mfaTenantId = sessionUser.activeTenantId;  // Bewaar voor verify
      const csrfToken = generateCsrfToken();
      req.session.csrfToken = csrfToken;
      await sendMfaCodeEmail({ email: user.email, code, displayName: user.displayName });
      return req.session.save(() => res.json({ mfaRequired: true, csrfToken }));
    }
  }

  // 7. Audit log fire-and-forget (VOOR session — mag nooit de login blokkeren)
  logSecurityEvent(user.id, SecurityEvents.LOGIN_SUCCESS, { success: true }).catch(() => {});

  // 8. Regenerate session — voorkomt session fixation attacks (OWASP recommendation)
  req.session.regenerate((regenErr) => {
    if (regenErr) return res.status(500).json({ error: "Sessiefout" });

    req.session.user = sessionUser;
    req.session.createdAt = Date.now();  // Absolute session timeout tracking (zie §7 middleware)
    const csrfToken = generateCsrfToken();
    req.session.csrfToken = csrfToken;

    req.session.save((err) => {
      if (err) return res.status(500).json({ error: "Sessiefout" });

      // CSRF cookie (httpOnly: false zodat JS het kan lezen)
      // sameSite matcht je session cookie — "lax" voor Model A, "none" voor Model B
      res.cookie("csrf_token", csrfToken, {
        httpOnly: false,
        secure: isProduction,
        sameSite: "lax",  // Wijzig naar "none" voor cross-origin (Model B)
        maxAge: 7 * 24 * 60 * 60 * 1000,
      });

      res.json({ success: true, user: sessionUser, csrfToken });
    });
  });
});
```

### Session endpoint

```typescript
app.get("/api/auth/session", async (req, res) => {
  const sessionUser = req.session.user;
  if (sessionUser) {
    res.json({ authenticated: true, user: sessionUser });
  } else {
    res.json({ authenticated: false });
  }
});
```

### Logout endpoint

```typescript
  // Logout: CSRF-vrij — als de CSRF cookie verlopen is na lange inactiviteit,
  // zou de user anders niet meer kunnen uitloggen. Logout is altijd veilig
  // omdat het alleen de eigen sessie vernietigt.
  app.post("/api/auth/logout", (req, res) => {
    const user = req.session.user;
    if (user) {
      logSecurityEvent(user.id, SecurityEvents.LOGOUT, {
        tenantId: user.activeTenantId, success: true,
      });
    }
    req.session.destroy((err) => {
      if (err) return res.status(500).json({ error: "Uitlogfout" });
      res.clearCookie("app.sid");
      res.clearCookie("csrf_token");
      res.json({ success: true });
    });
  });

  // Switch tenant (multi-tenant users)
  app.post("/api/auth/switch-tenant", requireAuth, async (req, res) => {
    const { tenantId } = req.body;
    const user = req.session.user!;

    // Controleer of user lid is van deze tenant
    const userTenants = await storage.getUserTenants(user.id);
    const targetTenant = userTenants.find(ut => ut.tenantId === tenantId);
    if (!targetTenant) return res.status(403).json({ error: "Geen toegang tot deze tenant" });

    // Check of tenant actief is (tenzij superadmin)
    if (user.globalRole !== "superadmin") {
      const tenant = await storage.getTenant(tenantId);
      if (!tenant?.isActive) return res.status(403).json({ error: "Tenant is niet actief" });
    }

    req.session.user = { ...user, activeTenantId: tenantId };
    req.session.save((err) => {
      if (err) return res.status(500).json({ error: "Sessiefout" });
      res.json({ success: true, activeTenantId: tenantId });
    });
  });

}  // einde setupAuth()
```

### Auth middleware helpers
```typescript
// Helper: haal user op uit session
export function getAuthUser(req: Request): SessionUser | undefined {
  return req.session?.user;
}

// Geëxporteerde middleware functies — werken op req.session.user (getypt via express-session augmentation)
export function requireAuth(req: Request, res: Response, next: NextFunction) {
  if (!req.session?.user) return res.status(401).json({ error: "Niet geauthenticeerd", code: "AUTH_1001" });
  next();
}

export function requireTenant(req: Request, res: Response, next: NextFunction) {
  if (!req.session?.user?.activeTenantId) return res.status(400).json({ error: "Geen actieve tenant" });
  next();
}

export function requireSuperAdmin(req: Request, res: Response, next: NextFunction) {
  if (req.session?.user?.globalRole !== "superadmin") return res.status(403).json({ error: "Superadmin vereist" });
  next();
}

export function requireTenantAdmin(req: Request, res: Response, next: NextFunction) {
  const user = req.session?.user;
  if (user?.globalRole === "superadmin") return next();
  const tenant = user?.tenants?.find(t => t.id === user.activeTenantId);
  if (tenant?.role !== "admin") return res.status(403).json({ error: "Admin vereist" });
  next();
}

// Helper om tenantId uit session te halen
export function getActiveTenantId(req: Request): string {
  return req.session.user!.activeTenantId!;
}
```

### Express Router bundeling patroon

Elke groep endpoints wordt gebundeld in een Express `Router`. Dit zijn de routers die `registerRoutes()` hierboven importeert:

```typescript
// server/routes/items.ts — voorbeeld domein-router
import { Router } from "express";
import { requireAuth, requireTenant, getActiveTenantId } from "../auth";
import { asyncHandler } from "../utils/errors";
import { storage } from "../storage";

export const itemsRouter = Router();

itemsRouter.get("/api/items", requireAuth, requireTenant, asyncHandler(async (req, res) => {
  const tenantId = getActiveTenantId(req);
  const items = await storage.getItems(tenantId);
  res.json(items);
}));

// Herhaal dit patroon voor: billingRouter, settingsRouter, adminRouter, gdprRouter
// Elke router exporteert een `Router()` instantie met zijn eigen endpoints
```

### CSRF mechanisme uitleg

Dit document gebruikt het **synchronous double-submit pattern**:
1. **Server** genereert een CSRF token bij login en slaat het op in `req.session.csrfToken`
2. **Server** stuurt het token naar de client via **drie kanalen** (redundantie):
   - JSON response body (`csrfToken` veld) — voor initiële opslag
   - `X-CSRF-Token` response header — voor ongoing sync
   - `csrf_token` cookie (`httpOnly: false`) — zodat JS het kan lezen
3. **Client** leest het token uit cookie (primair) of localStorage (dev fallback)
4. **Client** stuurt het token mee in de `X-CSRF-Token` request header bij elke mutatie
5. **Server** vergelijkt header-token met session-token — mismatch = 403

De drie kanalen zijn geen drie aparte mechanismen — het is één token via meerdere transportlagen voor maximale betrouwbaarheid bij cross-origin setups.

### Frontend: Complete CSRF + Query configuratie (client/src/lib/queryClient.ts)

```typescript
import { QueryClient, QueryFunction } from "@tanstack/react-query";

const CSRF_TOKEN_KEY = "app_csrf_token";

// === CSRF Token Management ===
function getCsrfToken(): string | null {
  // Cookie eerst (gezet door server bij login — veiligste methode)
  const match = document.cookie.match(/csrf_token=([^;]+)/);
  if (match) return match[1];
  // localStorage fallback ALLEEN in development (cross-domain iframe scenario)
  // In productie is localStorage kwetsbaar voor XSS — cookie MOET beschikbaar zijn
  if (import.meta.env.DEV) {
    return localStorage.getItem(CSRF_TOKEN_KEY);
  }
  return null;
}

export function setCsrfToken(token: string | undefined) {
  // localStorage alleen in development voor cross-domain fallback
  if (token && import.meta.env.DEV) {
    localStorage.setItem(CSRF_TOKEN_KEY, token);
  } else {
    localStorage.removeItem(CSRF_TOKEN_KEY);
  }
}

export function clearCsrfToken() {
  localStorage.removeItem(CSRF_TOKEN_KEY);
}

// === API Request (voor mutaties) ===
export async function apiRequest(method: string, url: string, data?: unknown): Promise<Response> {
  const headers: Record<string, string> = {};
  if (data) headers["Content-Type"] = "application/json";

  // CSRF token voor state-changing requests
  if (["POST", "PUT", "PATCH", "DELETE"].includes(method.toUpperCase())) {
    const csrf = getCsrfToken();
    if (csrf) headers["X-CSRF-Token"] = csrf;
  }

  const res = await fetch(url, {
    method,
    headers,
    body: data ? JSON.stringify(data) : undefined,
    credentials: "include",  // VERPLICHT voor cross-origin cookies
  });

  // Sync CSRF token uit response header
  const newToken = res.headers.get("X-CSRF-Token");
  if (newToken) setCsrfToken(newToken);

  if (!res.ok) throw new Error(`${res.status}: ${await res.text()}`);
  return res;
}

// === Default Query Function (voor useQuery) ===
type UnauthorizedBehavior = "throw" | "returnNull";

export const getQueryFn: <T>(options: {
  on401: UnauthorizedBehavior;
}) => QueryFunction<T> =
  ({ on401 }) =>
  async ({ queryKey }) => {
    const url = queryKey[0] as string;  // queryKey = ["/api/endpoint"]
    const res = await fetch(url, {
      credentials: "include",  // Cookies meesturen bij ELKE query
      cache: "no-store",
    });

    // Sync CSRF token ook bij GET requests
    const newCsrfToken = res.headers.get("X-CSRF-Token");
    if (newCsrfToken) setCsrfToken(newCsrfToken);

    if (on401 === "returnNull" && res.status === 401) return null;
    if (!res.ok) throw new Error(`${res.status}: ${await res.text()}`);
    return await res.json();
  };

// === QueryClient met defaults ===
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      queryFn: getQueryFn({ on401: "throw" }),
      refetchInterval: false,
      refetchOnWindowFocus: false,
      staleTime: Infinity,      // Data wordt nooit automatisch "stale" — invalidate handmatig
      retry: false,
    },
    mutations: {
      retry: false,
    },
  },
});
```

**Waarom `staleTime: Infinity`?** Data verandert alleen door expliciete acties. Na een mutatie roep je `queryClient.invalidateQueries()` aan. Dit voorkomt onnodige API calls en geeft voorspelbaar gedrag.

---

## 7b. Registration & Signup Flow

### Wanneer toepassen
Zodra je self-service onboarding nodig hebt (bezoekers die zelf een account aanmaken via de pricing pagina).

### De flow

```
Pricing pagina → Registratie → Tenant aangemaakt → Session actief → Redirect naar Stripe Checkout
                                                                      ↓
                                                               Stripe webhook → Subscription opgeslagen → Dashboard
```

**Belangrijk:** Registratie maakt een tenant + user aan maar **nog geen subscription**. De subscription wordt pas aangemaakt na Stripe checkout (§15). De tenant `isActive` start op `true` zodat de user de checkout kan bereiken.

### Zod schema (shared/schema.ts)

```typescript
export const registerSchema = z.object({
  email: z.string().email("Ongeldig emailadres"),
  password: z.string()
    .min(12, "Minimaal 12 tekens")
    .regex(/[A-Z]/, "Minimaal 1 hoofdletter")
    .regex(/[a-z]/, "Minimaal 1 kleine letter")
    .regex(/[0-9]/, "Minimaal 1 cijfer"),
  companyName: z.string().min(2, "Bedrijfsnaam is verplicht"),
  displayName: z.string().optional(),
});
```

### Endpoint (server/auth.ts)

```typescript
import { tenantProvisioning } from "../services/tenant-provisioning"; // Zie §14 voor implementatie
import { logSecurityEvent, SecurityEvents } from "../utils/audit";     // Zie §19

app.post("/api/auth/register", publicLimiter, async (req, res) => {
  const parsed = registerSchema.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({ error: parsed.error.issues[0].message });
  }
  const { email, password, companyName, displayName } = parsed.data;

  // 1. Check of email al in gebruik is
  const existing = await storage.getUserByEmail(email.toLowerCase());
  if (existing) {
    return res.status(409).json({ error: "E-mailadres al in gebruik" });
  }

  // 2. Wachtwoord hashen
  const passwordHash = await bcrypt.hash(password, 12);

  // 3. Tenant aanmaken via provisioning service (§14)
  const slug = companyName.toLowerCase().replace(/[^a-z0-9]/g, "-").replace(/-+/g, "-");
  const tenant = await tenantProvisioning.provisionTenant({
    name: companyName,
    slug: `${slug}-${Date.now().toString(36)}`,  // Uniciteit garanderen
  });

  // 4. User aanmaken
  const user = await storage.createUserWithDetails({
    username: email.toLowerCase(),
    email: email.toLowerCase(),
    passwordHash,
    displayName: displayName || companyName,
    globalRole: "standard",
    isActive: true,
    mustChangePassword: false,
  });

  // 5. User als admin aan tenant toevoegen
  await storage.addUserToTenant(user.id, tenant.id, "admin");

  // 6. Session aanmaken
  const userTenants = await storage.getUserTenants(user.id);
  const sessionUser: SessionUser = {
    id: user.id,
    username: user.username,
    email: user.email ?? undefined,
    displayName: user.displayName ?? undefined,
    globalRole: user.globalRole as GlobalRole,
    activeTenantId: tenant.id,
    tenants: userTenants.map(ut => ({
      id: ut.tenantId, name: ut.tenantName, slug: ut.tenantSlug, role: ut.role as TenantRole,
    })),
  };

  // Audit log fire-and-forget (VOOR session — nooit await in callback)
  logSecurityEvent(user.id, SecurityEvents.REGISTER, {
    tenantId: tenant.id, success: true,
  }).catch(() => {});

  // Regenerate session to prevent session fixation
  req.session.regenerate((regenErr) => {
    if (regenErr) return res.status(500).json({ error: "Sessiefout" });
    req.session.user = sessionUser;
    req.session.createdAt = Date.now();
    const csrfToken = generateCsrfToken();
    req.session.csrfToken = csrfToken;

    req.session.save((err) => {
      if (err) return res.status(500).json({ error: "Sessiefout" });
      res.json({ success: true, user: sessionUser, csrfToken, tenantId: tenant.id });
    });
  });
});
```

### Route registratie

Dit endpoint hoort bij de **publieke routes** (Fase 3 in §7) — vóór de CSRF middleware, want de user heeft nog geen sessie:

```typescript
// In registerRoutes():
// FASE 3: Publieke routes
app.use(publicAuthRouter);  // Bevat: register, accept-invitation, reset-password
```

### Frontend: na registratie → Stripe checkout

```typescript
// Na succesvolle registratie: redirect naar Stripe checkout
const registerMutation = useMutation({
  mutationFn: async (data: RegisterData) => {
    const res = await apiRequest("POST", "/api/auth/register", data);
    return res.json();
  },
  onSuccess: async (data) => {
    if (data.csrfToken) setCsrfToken(data.csrfToken);
    // Start Stripe checkout voor gekozen tier
    const checkoutRes = await apiRequest("POST", "/api/billing/checkout", {
      tier: selectedTier,      // Van pricing pagina
      interval: selectedInterval,
      successUrl: `${window.location.origin}/dashboard?welcome=true`,
      cancelUrl: `${window.location.origin}/pricing`,
    });
    const { url } = await checkoutRes.json();
    window.location.href = url;  // Redirect naar Stripe
  },
});
```

### Verschil met uitnodigingen (§13)

| | Registratie (§7b) | Uitnodiging (§13) |
|---|---|---|
| Initiatief | Gebruiker zelf | Admin van bestaande tenant |
| Tenant | Nieuwe tenant aangemaakt | Bestaande tenant |
| Rol | Altijd `admin` | Door admin bepaald (admin/member) |
| Subscription | Nog aanmaken via Stripe | Al actief (tenant bestaat) |
| Authenticatie | Sessie direct aangemaakt | Na token-validatie + account-creatie |

---

## 8. Frontend Auth

### useAuth hook (client/src/hooks/use-auth.tsx)

```tsx
// useState is nodig voor mfaPending state
import { createContext, useContext, useState, ReactNode } from "react";
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { apiRequest, setCsrfToken, clearCsrfToken } from "@/lib/queryClient";

interface AuthContextType {
  user: SessionUser | null;
  isLoading: boolean;
  isAuthenticated: boolean;
  login: (credentials: { email: string; password: string }) => Promise<{ mfaRequired?: boolean } | undefined>;
  logout: () => Promise<void>;
  switchTenant: (tenantId: string) => Promise<void>;
  mfaPending: boolean;
  verifyMfa: (code: string) => Promise<void>;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const queryClient = useQueryClient();
  const [mfaPending, setMfaPending] = useState(false);

  // Check session bij mount
  const { data: sessionData, isLoading } = useQuery({
    queryKey: ["/api/auth/session"],
    queryFn: async () => {
      const res = await fetch("/api/auth/session", { credentials: "include" });
      if (!res.ok) return null;
      return res.json();
    },
    staleTime: Infinity,
    retry: false,
  });

  const user = sessionData?.user || null;

  const loginMutation = useMutation({
    mutationFn: async (credentials: { email: string; password: string }) => {
      const res = await apiRequest("POST", "/api/auth/login", credentials);
      return res.json();
    },
    onSuccess: (data) => {
      if (data.csrfToken) setCsrfToken(data.csrfToken);
      if (data.mfaRequired) {
        setMfaPending(true);  // Toon MFA form in plaats van dashboard
        return;
      }
      // Ververs session query
      queryClient.invalidateQueries({ queryKey: ["/api/auth/session"] });
      queryClient.invalidateQueries();  // Reset alle cached data
    },
  });

  const verifyMfaMutation = useMutation({
    mutationFn: async (code: string) => {
      const res = await apiRequest("POST", "/api/auth/mfa/verify", { code });
      return res.json();
    },
    onSuccess: (data) => {
      if (data.csrfToken) setCsrfToken(data.csrfToken);
      setMfaPending(false);
      queryClient.invalidateQueries({ queryKey: ["/api/auth/session"] });
      queryClient.invalidateQueries();
    },
  });

  const logoutMutation = useMutation({
    mutationFn: () => apiRequest("POST", "/api/auth/logout"),
    onSuccess: () => {
      clearCsrfToken();
      queryClient.clear();
      queryClient.invalidateQueries({ queryKey: ["/api/auth/session"] });
    },
  });

  return (
    <AuthContext.Provider value={{
      user,
      isLoading,
      isAuthenticated: !!user,
      login: async (credentials) => { const data = await loginMutation.mutateAsync(credentials); return data; },
      logout: logoutMutation.mutateAsync,
      switchTenant: async (tenantId) => {
        await apiRequest("POST", "/api/auth/switch-tenant", { tenantId });
        queryClient.invalidateQueries();
      },
      mfaPending,
      verifyMfa: verifyMfaMutation.mutateAsync,
    }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used within AuthProvider");
  return ctx;
}
```

### Protected routes (client/src/App.tsx)

```tsx
// wouter: Redirect en useLocation worden apart geïmporteerd
import { Switch, Route, Redirect, useLocation } from "wouter";
import { useAuth } from "@/hooks/use-auth";
import { SubscriptionGuard } from "@/components/subscription-guard";

function App() {
  const { user, isLoading, isAuthenticated } = useAuth();

  if (isLoading) return <LoadingScreen />;

  // Forceer wachtwoord wijzigen als mustChangePassword = true
  const [location, setLocation] = useLocation();
  if (isAuthenticated && user?.mustChangePassword && location !== "/change-password") {
    setLocation("/change-password");  // Imperatieve redirect (wouter)
  }

  return (
    <Switch>
      {/* === PUBLIEKE ROUTES (geen auth nodig) === */}
      <Route path="/login" component={Login} />
      <Route path="/register" component={Register} />
      <Route path="/accept-invitation/:token" component={AcceptInvitation} />
      <Route path="/forgot-password" component={ForgotPassword} />
      <Route path="/reset-password/:token" component={ResetPassword} />
      <Route path="/pricing" component={Pricing} />
      <Route path="/privacy" component={PrivacyPolicy} />
      <Route path="/terms" component={TermsOfService} />

      {/* === LANDING (ongeauthenticeerd) === */}
      <Route path="/">
        {isAuthenticated ? <Redirect to="/dashboard" /> : <Landing />}
      </Route>

      {/* === WACHTWOORD WIJZIGEN (auth nodig, geen SubscriptionGuard) === */}
      <Route path="/change-password">
        {!isAuthenticated ? <Redirect to="/login" /> : <ChangePassword />}
      </Route>

      {/* === BESCHERMDE ROUTES === */}
      <Route path="/dashboard">
        {!isAuthenticated ? <Redirect to="/login" /> : (
          <SubscriptionGuard>
            <Dashboard />
          </SubscriptionGuard>
        )}
      </Route>

      {/* Herhaal voor andere beschermde routes */}
      <Route path="/settings">
        {!isAuthenticated ? <Redirect to="/login" /> : (
          <SubscriptionGuard>
            <Settings />
          </SubscriptionGuard>
        )}
      </Route>

      {/* 404 */}
      <Route>
        <NotFound />
      </Route>
    </Switch>
  );
}
```

**wouter notities:**
- `<Redirect to="/path" />` is een component (net als react-router)
- `useLocation()` retourneert `[location, setLocation]` voor imperatieve navigatie
- `setLocation("/path")` is de wouter manier om programmatisch te redirecten

### Pagina templates

Hieronder drie basissjablonen. Pas ze aan voor jouw domein — het patroon (hooks, guards, data fetching) is universeel.

#### Login pagina (client/src/pages/login.tsx)

```tsx
import { useState } from "react";
import { useAuth } from "@/hooks/use-auth";
import { useLocation } from "wouter";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default function Login() {
  const { login, mfaPending, verifyMfa } = useAuth();
  const [, setLocation] = useLocation();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [mfaCode, setMfaCode] = useState("");
  const [error, setError] = useState("");
  const [isLoading, setIsLoading] = useState(false);

  async function handleLogin(e: React.FormEvent) {
    e.preventDefault();
    setError("");
    setIsLoading(true);
    try {
      const result = await login({ email, password });
      // BELANGRIJK: check het login resultaat, NIET de mfaPending state.
      // React state updates zijn async — mfaPending kan nog false zijn op dit punt.
      if (!result?.mfaRequired) setLocation("/dashboard");
    } catch (err: any) {
      setError(err.message || "Inloggen mislukt");
    } finally {
      setIsLoading(false);
    }
  }

  async function handleMfa(e: React.FormEvent) {
    e.preventDefault();
    try {
      await verifyMfa(mfaCode);
      setLocation("/dashboard");
    } catch (err: any) {
      setError(err.message || "Ongeldige code");
    }
  }

  if (mfaPending) {
    return (
      <Card className="max-w-md mx-auto mt-20">
        <CardHeader><CardTitle>Verificatiecode</CardTitle></CardHeader>
        <CardContent>
          <form onSubmit={handleMfa} className="space-y-4">
            <p className="text-sm text-muted-foreground">Voer de 6-cijferige code in die naar je e-mail is verzonden.</p>
            <Input value={mfaCode} onChange={e => setMfaCode(e.target.value)} placeholder="123456" maxLength={6} />
            {error && <p className="text-sm text-destructive">{error}</p>}
            <Button type="submit" className="w-full">Verifiëren</Button>
          </form>
        </CardContent>
      </Card>
    );
  }

  return (
    <Card className="max-w-md mx-auto mt-20">
      <CardHeader><CardTitle>Inloggen</CardTitle></CardHeader>
      <CardContent>
        <form onSubmit={handleLogin} className="space-y-4">
          <Input type="email" value={email} onChange={e => setEmail(e.target.value)} placeholder="E-mailadres" required />
          <Input type="password" value={password} onChange={e => setPassword(e.target.value)} placeholder="Wachtwoord" required />
          {error && <p className="text-sm text-destructive">{error}</p>}
          <Button type="submit" className="w-full" disabled={isLoading}>
            {isLoading ? "Bezig..." : "Inloggen"}
          </Button>
          <div className="text-center text-sm">
            <a href="/forgot-password" className="text-primary hover:underline">Wachtwoord vergeten?</a>
          </div>
        </form>
      </CardContent>
    </Card>
  );
}
```

#### Dashboard pagina (client/src/pages/dashboard.tsx)

```tsx
import { useAuth } from "@/hooks/use-auth";
import { useQuery } from "@tanstack/react-query";
import { SubscriptionGuard } from "@/components/subscription-guard";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default function Dashboard() {
  const { user } = useAuth();

  // Data fetching via TanStack Query — queryKey = API URL
  const { data: stats, isLoading } = useQuery<{ total: number; thisMonth: number }>({
    queryKey: ["/api/stats"],  // Jouw domein-specifieke endpoint
  });

  return (
    <SubscriptionGuard>
      <div className="p-6 space-y-6">
        <h1 className="text-2xl font-bold">Dashboard</h1>
        <p className="text-muted-foreground">Welkom, {user?.displayName || user?.username}</p>

        <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
          <Card>
            <CardHeader><CardTitle className="text-sm">Totaal items</CardTitle></CardHeader>
            <CardContent>
              <p className="text-3xl font-bold">{isLoading ? "..." : stats?.total ?? 0}</p>
            </CardContent>
          </Card>
          <Card>
            <CardHeader><CardTitle className="text-sm">Deze maand</CardTitle></CardHeader>
            <CardContent>
              <p className="text-3xl font-bold">{isLoading ? "..." : stats?.thisMonth ?? 0}</p>
            </CardContent>
          </Card>
          {/* Voeg domein-specifieke kaarten toe */}
        </div>

        {/* Jouw domein-specifieke content */}
      </div>
    </SubscriptionGuard>
  );
}
```

#### Settings pagina (client/src/pages/settings.tsx)

```tsx
import { useEffect } from "react";
import { useLocation } from "wouter";
import { useAuth } from "@/hooks/use-auth";

// Sectie-ID's voor hash-navigatie (#profile, #security, #billing)
const sectionIds = ["profile", "security", "users", "billing"] as const;

export default function Settings() {
  const { user } = useAuth();
  const [location] = useLocation();

  // Scroll naar sectie bij hash-navigatie
  useEffect(() => {
    const hash = window.location.hash.slice(1);
    if (hash) document.getElementById(hash)?.scrollIntoView({ behavior: "smooth" });
  }, [location]);

  const isAdmin = user?.tenants?.find(t => t.id === user?.activeTenantId)?.role === "admin";

  return (
    <div className="p-6 space-y-8 max-w-3xl">
      <h1 className="text-2xl font-bold">Instellingen</h1>

      <section id="profile" className="space-y-4">
        <h2 className="text-xl font-semibold">Profiel</h2>
        {/* Profielvelden: displayName, email, avatar */}
      </section>

      <section id="security" className="space-y-4">
        <h2 className="text-xl font-semibold">Beveiliging</h2>
        {/* Wachtwoord wijzigen form */}
      </section>

      {isAdmin && (
        <section id="users" className="space-y-4">
          <h2 className="text-xl font-semibold">Gebruikers</h2>
          {/* Gebruikerslijst + uitnodiging form (§13) */}
        </section>
      )}

      {isAdmin && (
        <section id="billing" className="space-y-4">
          <h2 className="text-xl font-semibold">Abonnement</h2>
          {/* BillingStatus weergave + Customer Portal knop (§15-16) */}
        </section>
      )}
    </div>
  );
}
```

**Patroon voor nieuwe instellingssecties:**
1. Voeg sectie-ID toe aan `sectionIds` array
2. Voeg `<section id="nieuw">` blok toe aan de pagina
3. Voeg navigatie-item toe in je sidebar component

#### Registratie pagina (client/src/pages/register.tsx)

```tsx
import { useState } from "react";
import { useLocation } from "wouter";
import { useMutation } from "@tanstack/react-query";
import { apiRequest, setCsrfToken } from "@/lib/queryClient";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default function Register() {
  const [, setLocation] = useLocation();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [companyName, setCompanyName] = useState("");
  const [error, setError] = useState("");

  // Lees tier en interval uit URL params (vanuit pricing pagina)
  const params = new URLSearchParams(window.location.search);
  const selectedTier = params.get("tier") || "starter";
  const selectedInterval = params.get("interval") || "monthly";

  const registerMutation = useMutation({
    mutationFn: async () => {
      const res = await apiRequest("POST", "/api/auth/register", {
        email, password, companyName,
      });
      return res.json();
    },
    onSuccess: async (data) => {
      if (data.csrfToken) setCsrfToken(data.csrfToken);
      // Start Stripe checkout
      try {
        const checkoutRes = await apiRequest("POST", "/api/billing/checkout", {
          tier: selectedTier, interval: selectedInterval,
          successUrl: `${window.location.origin}/dashboard?welcome=true`,
          cancelUrl: `${window.location.origin}/pricing`,
        });
        const { url } = await checkoutRes.json();
        window.location.href = url;
      } catch {
        // Checkout mislukt maar account is aangemaakt — redirect naar dashboard
        setLocation("/dashboard");
      }
    },
    onError: (err: Error) => setError(err.message),
  });

  return (
    <Card className="max-w-md mx-auto mt-20">
      <CardHeader>
        <CardTitle>Account aanmaken</CardTitle>
        <p className="text-sm text-muted-foreground">14 dagen gratis proberen</p>
      </CardHeader>
      <CardContent>
        <form onSubmit={(e) => { e.preventDefault(); registerMutation.mutate(); }} className="space-y-4">
          <Input type="email" value={email} onChange={e => setEmail(e.target.value)}
                 placeholder="E-mailadres" required />
          <Input type="password" value={password} onChange={e => setPassword(e.target.value)}
                 placeholder="Wachtwoord (min 12 tekens)" required minLength={12} />
          <Input value={companyName} onChange={e => setCompanyName(e.target.value)}
                 placeholder="Bedrijfsnaam" required />
          {error && <p className="text-sm text-destructive">{error}</p>}
          <Button type="submit" className="w-full" disabled={registerMutation.isPending}>
            {registerMutation.isPending ? "Bezig..." : "Registreren"}
          </Button>
          <p className="text-center text-sm text-muted-foreground">
            Al een account? <a href="/login" className="text-primary hover:underline">Inloggen</a>
          </p>
        </form>
      </CardContent>
    </Card>
  );
}
```

#### Uitnodiging accepteren (client/src/pages/accept-invitation.tsx)

```tsx
import { useState } from "react";
import { useLocation, useParams } from "wouter";
import { useMutation } from "@tanstack/react-query";
import { apiRequest, setCsrfToken } from "@/lib/queryClient";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default function AcceptInvitation() {
  const { token } = useParams<{ token: string }>();
  const [, setLocation] = useLocation();
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");

  const acceptMutation = useMutation({
    mutationFn: async () => {
      const res = await apiRequest("POST", "/api/auth/accept-invitation", { token, password });
      return res.json();
    },
    onSuccess: (data) => {
      if (data.csrfToken) setCsrfToken(data.csrfToken);
      setLocation("/dashboard");
    },
    onError: (err: Error) => setError(err.message),
  });

  return (
    <Card className="max-w-md mx-auto mt-20">
      <CardHeader><CardTitle>Uitnodiging accepteren</CardTitle></CardHeader>
      <CardContent>
        <form onSubmit={(e) => { e.preventDefault(); acceptMutation.mutate(); }} className="space-y-4">
          <p className="text-sm text-muted-foreground">Kies een wachtwoord om je account te activeren.</p>
          <Input type="password" value={password} onChange={e => setPassword(e.target.value)}
                 placeholder="Wachtwoord (min 12 tekens)" required minLength={12} />
          {error && <p className="text-sm text-destructive">{error}</p>}
          <Button type="submit" className="w-full" disabled={acceptMutation.isPending}>
            {acceptMutation.isPending ? "Bezig..." : "Account activeren"}
          </Button>
        </form>
      </CardContent>
    </Card>
  );
}
```

#### Wachtwoord vergeten (client/src/pages/forgot-password.tsx)

```tsx
import { useState } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default function ForgotPassword() {
  const [email, setEmail] = useState("");
  const [submitted, setSubmitted] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    // ALTIJD 200 retourneren — voorkom user enumeration (§9 Principes)
    await fetch("/api/password-reset/request", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email }),
      credentials: "include",
    });
    setSubmitted(true);
  }

  if (submitted) {
    return (
      <Card className="max-w-md mx-auto mt-20">
        <CardHeader><CardTitle>E-mail verstuurd</CardTitle></CardHeader>
        <CardContent>
          <p className="text-sm text-muted-foreground">
            Als er een account bestaat met dit e-mailadres, ontvang je een link om je wachtwoord te resetten.
          </p>
          <a href="/login" className="text-primary hover:underline text-sm mt-4 block">Terug naar inloggen</a>
        </CardContent>
      </Card>
    );
  }

  return (
    <Card className="max-w-md mx-auto mt-20">
      <CardHeader><CardTitle>Wachtwoord vergeten</CardTitle></CardHeader>
      <CardContent>
        <form onSubmit={handleSubmit} className="space-y-4">
          <p className="text-sm text-muted-foreground">Vul je e-mailadres in om een resetlink te ontvangen.</p>
          <Input type="email" value={email} onChange={e => setEmail(e.target.value)} placeholder="E-mailadres" required />
          <Button type="submit" className="w-full">Verstuur resetlink</Button>
          <a href="/login" className="text-primary hover:underline text-sm block text-center">Terug naar inloggen</a>
        </form>
      </CardContent>
    </Card>
  );
}
```

#### Wachtwoord resetten (client/src/pages/reset-password.tsx)

```tsx
import { useState } from "react";
import { useLocation, useParams } from "wouter";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default function ResetPassword() {
  const { token } = useParams<{ token: string }>();
  const [, setLocation] = useLocation();
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");
  const [success, setSuccess] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    try {
      const res = await fetch(`/api/password-reset/${token}/complete`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ password }),
        credentials: "include",
      });
      if (!res.ok) {
        const data = await res.json();
        throw new Error(data.error || "Reset mislukt");
      }
      setSuccess(true);
      setTimeout(() => setLocation("/login"), 3000);
    } catch (err: any) {
      setError(err.message);
    }
  }

  if (success) {
    return (
      <Card className="max-w-md mx-auto mt-20">
        <CardHeader><CardTitle>Wachtwoord gewijzigd</CardTitle></CardHeader>
        <CardContent>
          <p className="text-sm text-muted-foreground">Je wachtwoord is succesvol gewijzigd. Je wordt doorgestuurd naar de inlogpagina...</p>
        </CardContent>
      </Card>
    );
  }

  return (
    <Card className="max-w-md mx-auto mt-20">
      <CardHeader><CardTitle>Nieuw wachtwoord instellen</CardTitle></CardHeader>
      <CardContent>
        <form onSubmit={handleSubmit} className="space-y-4">
          <Input type="password" value={password} onChange={e => setPassword(e.target.value)}
                 placeholder="Nieuw wachtwoord (min 12 tekens)" required minLength={12} />
          {error && <p className="text-sm text-destructive">{error}</p>}
          <Button type="submit" className="w-full">Wachtwoord resetten</Button>
        </form>
      </CardContent>
    </Card>
  );
}
```

#### Wachtwoord wijzigen (client/src/pages/change-password.tsx)

Wordt getoond wanneer `mustChangePassword = true` (bijv. na seed of admin-reset).

```tsx
import { useState } from "react";
import { useLocation } from "wouter";
import { useAuth } from "@/hooks/use-auth";
import { apiRequest } from "@/lib/queryClient";
import { queryClient } from "@/lib/queryClient";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default function ChangePassword() {
  const { user } = useAuth();
  const [, setLocation] = useLocation();
  const [currentPassword, setCurrentPassword] = useState("");
  const [newPassword, setNewPassword] = useState("");
  const [error, setError] = useState("");
  const [isLoading, setIsLoading] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError("");
    setIsLoading(true);
    try {
      await apiRequest("POST", "/api/auth/change-password", { currentPassword, newPassword });
      // Ververs session (mustChangePassword = false nu)
      queryClient.invalidateQueries({ queryKey: ["/api/auth/session"] });
      setLocation("/dashboard");
    } catch (err: any) {
      setError(err.message || "Wachtwoord wijzigen mislukt");
    } finally {
      setIsLoading(false);
    }
  }

  return (
    <Card className="max-w-md mx-auto mt-20">
      <CardHeader>
        <CardTitle>Wachtwoord wijzigen</CardTitle>
        {user?.mustChangePassword && (
          <p className="text-sm text-muted-foreground">Je moet je wachtwoord wijzigen voordat je verder kunt.</p>
        )}
      </CardHeader>
      <CardContent>
        <form onSubmit={handleSubmit} className="space-y-4">
          <Input type="password" value={currentPassword} onChange={e => setCurrentPassword(e.target.value)}
                 placeholder="Huidig wachtwoord" required />
          <Input type="password" value={newPassword} onChange={e => setNewPassword(e.target.value)}
                 placeholder="Nieuw wachtwoord (min 12 tekens)" required minLength={12} />
          {error && <p className="text-sm text-destructive">{error}</p>}
          <Button type="submit" className="w-full" disabled={isLoading}>
            {isLoading ? "Bezig..." : "Wachtwoord wijzigen"}
          </Button>
        </form>
      </CardContent>
    </Card>
  );
}
```

#### Landing pagina (client/src/pages/landing.tsx)

```tsx
import { useLocation } from "wouter";
import { Button } from "@/components/ui/button";

export default function Landing() {
  const [, setLocation] = useLocation();

  return (
    <div className="min-h-screen flex flex-col">
      {/* Header */}
      <header className="border-b px-6 py-4 flex items-center justify-between">
        <h1 className="text-xl font-bold">App Naam</h1>
        <div className="flex gap-2">
          <Button variant="ghost" onClick={() => setLocation("/login")}>Inloggen</Button>
          <Button onClick={() => setLocation("/pricing")}>Starten</Button>
        </div>
      </header>

      {/* Hero */}
      <main className="flex-1 flex items-center justify-center px-6">
        <div className="max-w-2xl text-center space-y-6">
          <h2 className="text-4xl md:text-5xl font-bold tracking-tight">
            Jouw product headline hier
          </h2>
          <p className="text-lg text-muted-foreground">
            Beschrijf in 1-2 zinnen wat je product doet en voor wie het is.
          </p>
          <div className="flex gap-4 justify-center">
            <Button size="lg" onClick={() => setLocation("/pricing")}>Start gratis proefperiode</Button>
            <Button size="lg" variant="outline" onClick={() => setLocation("/login")}>Inloggen</Button>
          </div>
        </div>
      </main>

      {/* Footer */}
      <footer className="border-t px-6 py-4 text-center text-sm text-muted-foreground">
        <div className="flex gap-4 justify-center">
          <a href="/privacy" className="hover:underline">Privacy</a>
          <a href="/terms" className="hover:underline">Voorwaarden</a>
        </div>
      </footer>
    </div>
  );
}
```

#### Layout component (client/src/components/layout.tsx)

Beschermde pagina's gebruiken een layout met sidebar + main content area. Wrap je beschermde routes in `<Layout>` zodat de sidebar consistent is. Publieke routes (login, register, etc.) staan BUITEN de layout:

```tsx
import { ReactNode } from "react";
import { AppSidebar } from "@/components/app-sidebar";

export function Layout({ children }: { children: ReactNode }) {
  return (
    <div className="flex min-h-screen">
      <AppSidebar />
      <main className="flex-1 overflow-auto">
        {children}
      </main>
    </div>
  );
}
```

Gebruik in `App.tsx`:
```tsx
// Beschermde routes met layout
<Route path="/dashboard">
  {!isAuthenticated ? <Redirect to="/login" /> : (
    <Layout>
      <SubscriptionGuard>
        <Dashboard />
      </SubscriptionGuard>
    </Layout>
  )}
</Route>
```

### Utility componenten (herbruikbaar)

Deze componenten worden door meerdere pagina's gebruikt. Plaats ze in `client/src/components/`.

```tsx
// === client/src/components/loading-screen.tsx ===
import { Spinner } from "@/components/spinner";

export function LoadingScreen() {
  return (
    <div className="flex items-center justify-center min-h-screen">
      <Spinner size="lg" />
    </div>
  );
}

// === client/src/components/spinner.tsx ===
import { cn } from "@/lib/utils";

export function Spinner({ size = "md", className }: { size?: "sm" | "md" | "lg"; className?: string }) {
  const sizeClasses = { sm: "h-4 w-4", md: "h-6 w-6", lg: "h-10 w-10" };
  return (
    <div className={cn("animate-spin rounded-full border-2 border-muted border-t-primary", sizeClasses[size], className)} />
  );
}

// === client/src/components/not-found.tsx ===
import { Button } from "@/components/ui/button";
import { useLocation } from "wouter";

export function NotFound() {
  const [, setLocation] = useLocation();
  return (
    <div className="flex flex-col items-center justify-center min-h-[60vh] gap-4">
      <h1 className="text-4xl font-bold">404</h1>
      <p className="text-muted-foreground">Pagina niet gevonden</p>
      <Button onClick={() => setLocation("/")}>Terug naar home</Button>
    </div>
  );
}

// === client/src/components/subscription-blocked.tsx ===
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { apiRequest } from "@/lib/queryClient";

export function SubscriptionBlocked() {
  async function openPortal() {
    const res = await apiRequest("POST", "/api/billing/portal");
    const { url } = await res.json();
    window.location.href = url;
  }
  return (
    <div className="flex items-center justify-center min-h-[60vh]">
      <Card className="max-w-md">
        <CardHeader><CardTitle>Abonnement verlopen</CardTitle></CardHeader>
        <CardContent className="space-y-4">
          <p className="text-muted-foreground">Je abonnement is niet meer actief. Vernieuw je abonnement om verder te gaan.</p>
          <Button onClick={openPortal} className="w-full">Abonnement vernieuwen</Button>
        </CardContent>
      </Card>
    </div>
  );
}

// === client/src/components/warning-banner.tsx ===
import { AlertTriangle } from "lucide-react";

export function WarningBanner({ message }: { message?: string }) {
  if (!message) return null;
  return (
    <div className="bg-yellow-50 dark:bg-yellow-900/20 border border-yellow-200 dark:border-yellow-800 rounded-lg p-3 mb-4 flex items-center gap-2">
      <AlertTriangle className="h-4 w-4 text-yellow-600 shrink-0" />
      <p className="text-sm text-yellow-800 dark:text-yellow-200">{message}</p>
    </div>
  );
}

// === client/src/components/error-boundary.tsx ===
import { Component, ReactNode } from "react";

interface Props { children: ReactNode; fallback?: ReactNode; }
interface State { hasError: boolean; error?: Error; }

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error("[ErrorBoundary]", error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="flex flex-col items-center justify-center min-h-[40vh] gap-4">
          <h2 className="text-xl font-semibold">Er ging iets mis</h2>
          <p className="text-muted-foreground">{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false })}
                  className="text-primary underline">Opnieuw proberen</button>
        </div>
      );
    }
    return this.props.children;
  }
}
```

Gebruik ErrorBoundary in `App.tsx`:
```tsx
// Wrap de hele app (of per route)
<ErrorBoundary>
  <AuthProvider>
    <Switch>{/* routes */}</Switch>
  </AuthProvider>
</ErrorBoundary>
```

### Sidebar navigatie (client/src/components/app-sidebar.tsx)

```tsx
import { useAuth } from "@/hooks/use-auth";
import { useLocation } from "wouter";
import { cn } from "@/lib/utils";
import { LayoutDashboard, Settings, Users, CreditCard, LogOut } from "lucide-react";

const navItems = [
  { label: "Dashboard", href: "/dashboard", icon: LayoutDashboard },
  { label: "Instellingen", href: "/settings", icon: Settings },
  // Voeg domein-specifieke items toe
];

export function AppSidebar() {
  const { user, logout } = useAuth();
  const [location, setLocation] = useLocation();
  const isAdmin = user?.tenants?.find(t => t.id === user?.activeTenantId)?.role === "admin";

  return (
    <aside className="w-64 border-r bg-card min-h-screen p-4 flex flex-col">
      <div className="mb-8">
        <h1 className="text-lg font-bold">App Naam</h1>
        <p className="text-sm text-muted-foreground">{user?.displayName || user?.username}</p>
      </div>
      <nav className="flex-1 space-y-1">
        {navItems.map(item => (
          <button key={item.href}
            onClick={() => setLocation(item.href)}
            className={cn(
              "flex items-center gap-3 w-full px-3 py-2 rounded-md text-sm transition-colors",
              location === item.href ? "bg-accent text-accent-foreground" : "hover:bg-muted"
            )}>
            <item.icon className="h-4 w-4" />
            {item.label}
          </button>
        ))}
      </nav>
      <button onClick={() => logout()} className="flex items-center gap-3 px-3 py-2 text-sm text-muted-foreground hover:text-foreground">
        <LogOut className="h-4 w-4" />
        Uitloggen
      </button>
    </aside>
  );
}
```

### Timezone utilities (client/src/lib/timezone.ts)

Sla alle datums op in UTC (TIMESTAMPTZ). Toon ze in de lokale tijdzone van de browser.

```typescript
// client/src/lib/timezone.ts
import { formatInTimeZone } from "date-fns-tz";
import { nl } from "date-fns/locale";

// Detecteer browser tijdzone
export function getUserTimezone(): string {
  return Intl.DateTimeFormat().resolvedOptions().timeZone;  // Bijv. "Europe/Amsterdam"
}

// Formatteer UTC datum voor weergave
export function formatDate(date: string | Date, format = "d MMM yyyy HH:mm"): string {
  const tz = getUserTimezone();
  return formatInTimeZone(new Date(date), tz, format, { locale: nl });
}

// Relatieve tijd ("2 uur geleden")
export function timeAgo(date: string | Date): string {
  const ms = Date.now() - new Date(date).getTime();
  const minutes = Math.floor(ms / 60000);
  if (minutes < 1) return "Zojuist";
  if (minutes < 60) return `${minutes} min geleden`;
  const hours = Math.floor(minutes / 60);
  if (hours < 24) return `${hours} uur geleden`;
  const days = Math.floor(hours / 24);
  return `${days} ${days === 1 ? "dag" : "dagen"} geleden`;
}
```

---

## 9. Password Security & Password Reset

> **Principes (niet-onderhandelbaar)**
> - Wachtwoorden worden ALTIJD gehasht met bcrypt (minimaal 12 rounds). NOOIT plain text opslaan.
> - Minimale complexiteit: 12 tekens, 1 hoofdletter, 1 kleine letter, 1 cijfer.
> - Laatste 5 wachtwoorden bijhouden — hergebruik geblokkeerd.
> - Password reset: ALTIJD 200 retourneren (ook als email niet bestaat) — voorkom user enumeration.
> - Reset tokens worden gehasht opgeslagen (bcrypt) — NOOIT plain text in database.

### Implementatie
```typescript
const BCRYPT_ROUNDS = 12;
const PASSWORD_HISTORY_COUNT = 5;

// === Hashing ===
const hash = await bcrypt.hash(password, BCRYPT_ROUNDS);
const isValid = await bcrypt.compare(password, user.passwordHash);

// === Complexiteitseisen ===
function validatePassword(password: string): { valid: boolean; error?: string } {
  if (password.length < 12) return { valid: false, error: "Minimaal 12 tekens" };
  if (!/[A-Z]/.test(password)) return { valid: false, error: "Minimaal 1 hoofdletter" };
  if (!/[a-z]/.test(password)) return { valid: false, error: "Minimaal 1 kleine letter" };
  if (!/[0-9]/.test(password)) return { valid: false, error: "Minimaal 1 cijfer" };
  return { valid: true };
}

// === Wachtwoordgeschiedenis (voorkom hergebruik) ===
async function checkPasswordHistory(userId: string, newPassword: string): Promise<boolean> {
  const history = await storage.getPasswordHistory(userId, PASSWORD_HISTORY_COUNT);
  for (const oldHash of history) {
    if (await bcrypt.compare(newPassword, oldHash)) return false; // Hergebruik!
  }
  return true; // Veilig
}

// === Change-password endpoint (achter auth + CSRF) ===
app.post("/api/auth/change-password", requireAuth, asyncHandler(async (req, res) => {
  const user = getAuthUser(req)!;
  const { currentPassword, newPassword } = req.body;

  // 1. Huidig wachtwoord verifiëren
  const dbUser = await storage.getUser(user.id);
  if (!dbUser || !(await bcrypt.compare(currentPassword, dbUser.passwordHash))) {
    return res.status(401).json({ error: "Huidig wachtwoord is onjuist" });
  }

  // 2. Nieuw wachtwoord valideren
  const validation = validatePassword(newPassword);
  if (!validation.valid) return res.status(400).json({ error: validation.error });

  // 3. Check tegen geschiedenis
  const historyOk = await checkPasswordHistory(user.id, newPassword);
  if (!historyOk) return res.status(400).json({ error: "Wachtwoord recent gebruikt" });

  // 4. Hash + opslaan + geschiedenis + mustChangePassword reset
  const hash = await bcrypt.hash(newPassword, BCRYPT_ROUNDS);
  await storage.updateUserDetails(user.id, { passwordHash: hash, mustChangePassword: false });
  await storage.addPasswordToHistory(user.id, hash);

  res.json({ success: true });
}));
```

### Gedeelde utility: `validateToken()` (gebruikt door §9 EN §13)

**IMPLEMENTEER DIT EERST** — zowel password reset (§9) als invitation acceptance (§13) gebruiken deze functie.

Plaats in `server/utils/tokens.ts` (of in `server/auth.ts`):

```typescript
// server/utils/tokens.ts
import bcrypt from "bcrypt";
import { storage } from "../storage";

export async function validateToken(
  rawToken: string,
  type: Token["type"]
): Promise<{ valid: boolean; error?: string; token?: Token }> {
  // STAP 1: O(1) prefix-lookup (als token_prefix kolom beschikbaar is)
  const prefix = rawToken.substring(0, 8);
  const prefixMatches = await storage.getPendingTokensByPrefix(prefix, type);

  // Als prefix-lookup resultaten geeft, check alleen die (O(1) → O(k) met k << n)
  const candidates = prefixMatches.length > 0
    ? prefixMatches
    : await storage.getPendingTokensByType(type);  // Fallback: O(n) voor oude tokens zonder prefix

  for (const record of candidates) {
    if (await bcrypt.compare(rawToken, record.tokenHash)) {
      if (new Date() > record.expiresAt) {
        return { valid: false, error: "Token verlopen" };
      }
      return { valid: true, token: record };
    }
  }
  return { valid: false, error: "Ongeldig token" };
}

// Performance: Met token_prefix is lookup O(1) + bcrypt verify.
// Zonder prefix (legacy tokens) is het O(n) over alle pending tokens van dat type.
// Bij >1000 pending tokens: voeg een cleanup job toe die expired tokens verwijdert.
// Voorbeeld: DELETE FROM tokens WHERE status = 'pending' AND expires_at < NOW();
```

### Password Reset flow (3 endpoints)

```typescript
// === 1. POST /api/password-reset/request ===
// Altijd 200 retourneren (voorkom email enumeration)
app.post("/api/password-reset/request", publicLimiter, async (req, res) => {
  const { email } = req.body;
  const user = await storage.getUserByEmail(email.toLowerCase());
  if (user) {
    const rawToken = crypto.randomBytes(32).toString("base64url");
    const tokenHash = await bcrypt.hash(rawToken, 12);
    await storage.revokePendingTokens(email.toLowerCase(), "", "password_reset");
    await storage.createToken({
      tokenHash, type: "password_reset", status: "pending",
      tokenPrefix: rawToken.substring(0, 8),  // O(1) prefix-lookup (zelfde als invitations)
      email: email.toLowerCase(), userId: user.id,
      expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000), // 24 uur
    });
    const resetUrl = `${process.env.FRONTEND_URL}/reset-password/${rawToken}`;
    await sendPasswordResetEmail({ email, resetUrl, expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000) });
  }
  // ALTIJD success (ook als user niet bestaat)
  res.json({ success: true, message: "Als dit e-mailadres bekend is, ontvang je een reset-link." });
});

// === 2. GET /api/password-reset/:token/validate ===
// Frontend checkt of token geldig is voordat formulier getoond wordt
// validateToken() — zie hierboven (gedeeld door password resets + uitnodigingen §13)
app.get("/api/password-reset/:token/validate", async (req, res) => {
  const result = await validateToken(req.params.token, "password_reset");
  res.json({ valid: result.valid, email: result.token?.email });
});

// === 3. POST /api/password-reset/:token/complete ===
app.post("/api/password-reset/:token/complete", async (req, res) => {
  const { password } = req.body;
  const result = await validateToken(req.params.token, "password_reset");
  if (!result.valid) return res.status(400).json({ error: result.error });

  // Complexiteit + geschiedenis check
  const validation = validatePassword(password);
  if (!validation.valid) return res.status(400).json({ error: validation.error });
  const historyOk = await checkPasswordHistory(result.token!.userId!, password);
  if (!historyOk) return res.status(400).json({ error: "Wachtwoord recent gebruikt" });

  // Update
  const hash = await bcrypt.hash(password, 12);
  await storage.addPasswordToHistory(result.token!.userId!, hash);
  await storage.updateUserDetails(result.token!.userId!, { passwordHash: hash, mustChangePassword: false });
  await storage.updateTokenStatus(result.token!.id, "accepted", new Date());

  res.json({ success: true, redirectTo: "/login" });
});
```

**Belangrijk:** Password reset tokens worden met bcrypt gehasht (net als uitnodigings-tokens). Validatie is O(n) — je kunt niet zoeken op een bcrypt hash.

---

## 10. Email-Based MFA

### Flow
```
Login → Wachtwoord OK → Check tenant MFA setting → Genereer 6-cijferige code
→ Hash met bcrypt → Sla hash op in session → Stuur code per email
→ User stuurt code terug → Vergelijk met bcrypt.compare → Sessie aanmaken
```

### Implementatie
```typescript
const MFA_CODE_VALIDITY_MS = 10 * 60 * 1000; // 10 minuten

function generateMfaCode(): string {
  return crypto.randomInt(100000, 1000000).toString();  // Cryptografisch veilig
}

// Na succesvolle wachtwoord check:
if (tenantSettings.mfaRequired && user.email) {
  const code = generateMfaCode();
  req.session.mfaPending = true;
  req.session.mfaCodeHash = await bcrypt.hash(code, 10);
  req.session.mfaCodeExpiresAt = Date.now() + MFA_CODE_VALIDITY_MS;
  req.session.mfaUserId = user.id;
  req.session.mfaTenantId = sessionUser.activeTenantId;  // Bewaar voor verify
  await sendMfaCodeEmail({ email: user.email, code, displayName: user.displayName });
  return res.json({ mfaRequired: true, csrfToken: req.session.csrfToken });
}

// Bij verificatie (mfaVerifyLimiter uit §11):
app.post("/api/auth/mfa/verify", mfaVerifyLimiter, async (req, res) => {
  const { code } = req.body;
  if (!req.session.mfaPending) return res.status(400).json({ error: "Geen MFA actief" });
  if (Date.now() > req.session.mfaCodeExpiresAt!) return res.status(401).json({ error: "Code verlopen" });
  if (!await bcrypt.compare(code.trim(), req.session.mfaCodeHash!)) {
    return res.status(401).json({ error: "Ongeldige code" });
  }

  // User ophalen en session opbouwen
  const user = await storage.getUser(req.session.mfaUserId!);
  if (!user) return res.status(400).json({ error: "Gebruiker niet gevonden" });
  const userTenants = await storage.getUserTenants(user.id);
  const sessionUser: SessionUser = {
    id: user.id, username: user.username, email: user.email ?? undefined,
    displayName: user.displayName ?? undefined,
    globalRole: user.globalRole as GlobalRole,
    activeTenantId: req.session.mfaTenantId || userTenants[0]?.tenantId,
    tenants: userTenants.map(ut => ({
      id: ut.tenantId, name: ut.tenantName, slug: ut.tenantSlug, role: ut.role as TenantRole,
    })),
    mustChangePassword: user.mustChangePassword === true,
  };

  // Regenerate session — voorkomt session fixation (MFA state is automatisch weg)
  req.session.regenerate((regenErr) => {
    if (regenErr) return res.status(500).json({ error: "Sessiefout" });
    req.session.user = sessionUser;
    req.session.createdAt = Date.now();
    const csrfToken = generateCsrfToken();
    req.session.csrfToken = csrfToken;
    req.session.save(() => res.json({ success: true, user: sessionUser, csrfToken }));
  });
});
```

### Multi-tenant MFA gedrag

Als een gebruiker lid is van meerdere tenants en slechts één tenant MFA vereist:
- **Login**: MFA wordt gecheckt op basis van `activeTenantId` (de eerste/standaard tenant).
- **Tenant switch naar MFA-tenant**: Als de gebruiker switcht naar een tenant die MFA vereist en al eerder MFA heeft doorlopen in dezelfde sessie, hoeft dit niet opnieuw. De sessie is al "sterk" geauthenticeerd.
- **Tenant switch naar niet-MFA-tenant**: Geen extra actie nodig.
- **Optioneel strenger**: Als je wilt dat elke switch naar een MFA-tenant opnieuw MFA vereist, voeg een `mfaVerifiedAt` timestamp toe aan de session en controleer of deze recenter is dan bijv. 1 uur.

### Cross-domain fallback
Wanneer cookies niet werken (development, iframes), gebruik een in-memory token store:
```typescript
const mfaTokenStore = new Map<string, { userId: string; codeHash: string; expiresAt: number }>();
// Genereer token bij MFA start, stuur mee in response
// Frontend stuurt token terug bij verify (ipv cookie-based session)
// Cleanup elke 5 minuten (voorkom memory leak)
```

---

## 11. Rate Limiting

```typescript
import rateLimit from "express-rate-limit";

// Login: alleen mislukte pogingen tellen
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,        // 15 minuten
  max: 10,                          // Max 10 failures
  skipSuccessfulRequests: true,     // Succesvolle logins tellen NIET mee
  message: { error: "Te veel pogingen. Probeer het over 15 minuten opnieuw." },
  standardHeaders: true,
  legacyHeaders: false,
});

// MFA verify: voorkom brute-force op 6-cijferige codes (10^6 = 1M mogelijkheden)
const mfaVerifyLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,        // 15 minuten
  max: 5,                           // Max 5 pogingen — strenger dan login
  message: { error: "Te veel verificatiepogingen. Probeer het over 15 minuten opnieuw." },
  standardHeaders: true,
  legacyHeaders: false,
});

// Publieke endpoints (registratie, uitnodiging)
const publicLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 20,
});

// Admin endpoints
const adminLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: process.env.NODE_ENV === "production" ? 100 : 500,
});

app.post("/api/auth/login", loginLimiter, loginHandler);
app.post("/api/auth/mfa/verify", mfaVerifyLimiter, mfaVerifyHandler);
```

### Account Lockout (persistent, overleeft herstart)

Rate limiting beschermt tegen snelle brute-force, maar een aanvaller kan na elke 15 minuten opnieuw proberen. Voeg daarom **persistent account lockout** toe op database-niveau:

```typescript
const MAX_FAILED_ATTEMPTS = 5;
const LOCKOUT_DURATION_MS = 15 * 60 * 1000; // 15 minuten

// In login handler, NA user ophalen:
if (user.lockedUntil && new Date() < new Date(user.lockedUntil)) {
  const remainingMin = Math.ceil((new Date(user.lockedUntil).getTime() - Date.now()) / 60000);
  return res.status(423).json({
    error: `Account vergrendeld. Probeer het over ${remainingMin} minuten opnieuw.`
  });
}

// Bij fout wachtwoord:
const newAttempts = (user.failedLoginAttempts || 0) + 1;
const updates: { failedLoginAttempts: number; lockedUntil?: Date | null } = {
  failedLoginAttempts: newAttempts,
};
if (newAttempts >= MAX_FAILED_ATTEMPTS) {
  updates.lockedUntil = new Date(Date.now() + LOCKOUT_DURATION_MS);
}
await storage.updateUserDetails(user.id, updates);

// Bij succesvolle login: reset counters
await storage.updateUserDetails(user.id, { failedLoginAttempts: 0, lockedUntil: null });
```

**Database kolommen** (al opgenomen in de initiële CREATE TABLE in §2. Voor brownfield/bestaande databases):
```sql
ALTER TABLE users ADD COLUMN IF NOT EXISTS failed_login_attempts INTEGER DEFAULT 0;
ALTER TABLE users ADD COLUMN IF NOT EXISTS locked_until TIMESTAMP;
```

---

## 12. Superadmin Impersonation

### Implementatie
```typescript
app.post("/api/auth/impersonate", async (req, res) => {
  const currentUser = req.session.user;

  // === GUARDS (alle 5 zijn verplicht) ===
  if (!currentUser) return res.status(401);
  if (currentUser.isImpersonating) return res.status(400).json({ error: "Al bezig" });
  if (currentUser.globalRole !== "superadmin") return res.status(403);
  if (req.body.userId === currentUser.id) return res.status(400).json({ error: "Kan jezelf niet impersoneren" });
  const target = await storage.getUser(req.body.userId);
  if (!target) return res.status(404).json({ error: "Gebruiker niet gevonden" });
  if (target.globalRole === "superadmin") return res.status(403).json({ error: "Kan geen superadmins impersoneren" });

  // === TARGET SESSIE OPBOUWEN ===
  const targetTenants = await storage.getUserTenants(target.id);
  const targetSessionData: SessionUser = {
    id: target.id,
    username: target.username,
    email: target.email ?? undefined,
    displayName: target.displayName ?? undefined,
    globalRole: target.globalRole as GlobalRole,
    activeTenantId: targetTenants.length > 0 ? targetTenants[0].tenantId : undefined,
    tenants: targetTenants.map(ut => ({
      id: ut.tenantId, name: ut.tenantName, slug: ut.tenantSlug, role: ut.role as TenantRole,
    })),
  };

  // === SESSIE VERVANGEN ===
  req.session.user = {
    ...targetSessionData,
    mustChangePassword: false,        // Bypass zodat superadmin niet vastloopt
    isImpersonating: true,
    impersonationStartedAt: Date.now(),  // Auto-expire na 1 uur
    originalUser: { id: currentUser.id, username: currentUser.username, globalRole: currentUser.globalRole },
  };
  req.session.save(() => res.json({ success: true }));
});

app.post("/api/auth/stop-impersonate", async (req, res) => {
  const current = req.session.user;
  if (!current?.isImpersonating) return res.status(400);

  // BELANGRIJK: Hervalideer dat originele user nog steeds superadmin is
  const original = await storage.getUser(current.originalUser!.id);
  if (!original || original.globalRole !== "superadmin") {
    req.session.destroy(() => {});
    return res.status(403).json({ error: "Geen superadmin meer" });
  }

  // Herstel de originele superadmin sessie
  const originalTenants = await storage.getUserTenants(original.id);
  req.session.user = {
    id: original.id,
    username: original.username,
    email: original.email ?? undefined,
    displayName: original.displayName ?? undefined,
    globalRole: original.globalRole as GlobalRole,
    activeTenantId: originalTenants.length > 0 ? originalTenants[0].tenantId : undefined,
    tenants: originalTenants.map(ut => ({
      id: ut.tenantId, name: ut.tenantName, slug: ut.tenantSlug, role: ut.role as TenantRole,
    })),
  };
  req.session.save(() => res.json({ success: true }));
});
```

### Auto-expire middleware

Impersonation sessies verlopen automatisch na 1 uur. Voeg dit toe aan je session middleware (zie §7):

```typescript
// In session middleware, na absolute timeout check:
if (req.session.user?.isImpersonating && req.session.user.impersonationStartedAt) {
  const IMPERSONATION_TIMEOUT_MS = 60 * 60 * 1000; // 1 uur
  if (Date.now() - req.session.user.impersonationStartedAt > IMPERSONATION_TIMEOUT_MS) {
    req.session.destroy(() => {});
    return next();  // User moet opnieuw inloggen
  }
}
```

---

# DEEL C: BILLING & ONBOARDING

---

## 13. Invitation System

> **Principes (niet-onderhandelbaar)**
> - Tokens worden met bcrypt gehasht opgeslagen — NOOIT plain text in de database.
> - Validatie gebruikt O(1) prefix-lookup (`token_prefix` kolom, eerste 8 chars raw token) met bcrypt verify op matches. Fallback naar O(n) voor oude tokens zonder prefix.
> - Bij nieuwe uitnodiging: revoke alle bestaande pending tokens voor zelfde email+tenant.
> - Token expiry is maximaal 72 uur.
> - Bestaande users worden aan de tenant toegevoegd, NIET opnieuw aangemaakt.

### Token lifecycle
```
Admin stuurt uitnodiging → Token aangemaakt (bcrypt-hashed)
→ Email met link (raw token in URL) → User klikt link
→ Token gevalideerd (bcrypt.compare tegen alle pending tokens)
→ Account aangemaakt OF tenant-toegang toegevoegd → Token gemarkeerd als "used"
```

### Kerncode
```typescript
import { validateToken } from "../utils/tokens"; // Gedefinieerd in §9 — gedeeld met password resets
const TOKEN_EXPIRY_HOURS = 72;

async function createInvitationToken(params: { email: string; tenantId: string; role: TenantRole; invitedById: string }) {
  const rawToken = crypto.randomBytes(32).toString("base64url");
  const tokenHash = await bcrypt.hash(rawToken, 12);
  const tokenPrefix = rawToken.substring(0, 8);  // O(1) prefix lookup
  const expiresAt = new Date(Date.now() + TOKEN_EXPIRY_HOURS * 60 * 60 * 1000);
  // Revoke bestaande pending uitnodigingen voor zelfde email+tenant
  await storage.revokePendingTokens(params.email.toLowerCase(), params.tenantId, "invitation");
  await storage.createToken({
    tokenHash, tokenPrefix, type: "invitation", status: "pending",
    email: params.email.toLowerCase(), tenantId: params.tenantId,
    tenantRole: params.role, invitedById: params.invitedById, expiresAt,
  });
  return rawToken;  // Alleen nu beschikbaar als plain text!
}

// validateToken() staat in server/utils/tokens.ts (zie §9 voor de volledige implementatie)
// Gebruik: const result = await validateToken(rawToken, "invitation");

async function acceptInvitation(token: Token, passwordHash: string) {
  const existing = await storage.getUserByEmail(token.email);
  if (existing) {
    // Bestaande user → voeg toe aan tenant
    await storage.addUserToTenant(existing.id, token.tenantId!, token.tenantRole);
  } else {
    // Nieuwe user → maak account aan
    const user = await storage.createUserWithDetails({
      username: token.email,
      email: token.email,
      passwordHash,
      displayName: token.displayName || token.email.split("@")[0],
      globalRole: "standard",
      isActive: true,
      mustChangePassword: false,
    });
    await storage.addUserToTenant(user.id, token.tenantId!, token.tenantRole);
  }
  await storage.updateTokenStatus(token.id, "accepted", new Date());
}
```

### Route: uitnodiging accepteren (server/routes/public-auth.ts)

```typescript
// POST /api/auth/accept-invitation — publieke route (geen auth nodig)
publicAuthRouter.post("/api/auth/accept-invitation", publicLimiter, async (req, res) => {
  const parsed = acceptInvitationSchema.safeParse(req.body);
  if (!parsed.success) return res.status(400).json({ error: parsed.error.issues[0].message });

  const result = await validateToken(parsed.data.token, "invitation");
  if (!result.valid) return res.status(400).json({ error: result.error });

  const passwordHash = await bcrypt.hash(parsed.data.password, 12);
  await acceptInvitation(result.token!, passwordHash);

  await logSecurityEvent(null, SecurityEvents.INVITATION_ACCEPTED, {
    success: true, additionalContext: { email: result.token!.email },
  }).catch(() => {});

  res.json({ success: true, message: "Account aangemaakt. Je kunt nu inloggen." });
});
```

### Route: uitnodiging versturen (server/routes/admin.ts)

```typescript
// POST /api/admin/invitations — admin-only, vereist auth + tenant
router.post("/api/admin/invitations", requireAuth, requireTenant, requireTenantAdmin, asyncHandler(async (req, res) => {
  const { email, role } = req.body;
  if (!email || !z.string().email().safeParse(email).success) {
    return res.status(400).json({ error: "Ongeldig emailadres" });
  }
  const tenantId = getActiveTenantId(req)!;
  const userId = req.session.user!.id;

  // Check tier limiet (§17)
  const userCheck = await checkUserLimit(tenantId, { isSuperadmin: req.session.user!.globalRole === "superadmin" });
  if (!userCheck.allowed) return res.status(429).json({ error: userCheck.reason });

  const rawToken = await createInvitationToken({ email, tenantId, role: role || "member", invitedById: userId });
  const invitationUrl = `${process.env.FRONTEND_URL}/accept-invitation/${rawToken}`;
  const tenant = await storage.getTenant(tenantId);
  await sendInvitationEmail({ email, invitationUrl, tenantName: tenant?.name, expiresAt: new Date(Date.now() + 72 * 60 * 60 * 1000) });

  await logSecurityEvent(userId, SecurityEvents.INVITATION_SENT, {
    tenantId, success: true, additionalContext: { invitedEmail: email, role },
  }).catch(() => {});

  res.json({ success: true, message: "Uitnodiging verstuurd" });
}));
```

---

## 14. Tenant Provisioning & Bootstrap

### Standaard data voor nieuwe tenants

```typescript
const DEFAULT_SYSTEM_SETTINGS: Record<string, string> = {
  language: "nl",
  timezone: "Europe/Amsterdam",
  mfaRequired: "false",
  // Voeg domein-specifieke defaults toe
};
```

### Bij Stripe checkout of handmatig door superadmin:
```typescript
class TenantProvisioningService {
  async provisionTenant(data: InsertTenant): Promise<Tenant> {
    const tenant = await storage.createTenant(data);
    // Standaard instellingen
    await storage.updateSystemSettings(DEFAULT_SYSTEM_SETTINGS, tenant.id);
    // Voeg hier domein-specifieke bootstrap toe, bijv.:
    // await storage.createItem({ tenantId: tenant.id, title: "Welkom", status: "active" });
    return tenant;
  }

  async addUserToTenant(userId: string, tenantId: string, role: TenantRole = "member") {
    await storage.addUserToTenant(userId, tenantId, role);
  }
}

// Singleton export — importeer als: import { tenantProvisioning } from "../services/tenant-provisioning";
export const tenantProvisioning = new TenantProvisioningService();
```

### Bootstrap: eerste superadmin aanmaken (server/seed.ts)

De allereerste tenant en superadmin worden aangemaakt via een seed script. Dit draai je **eenmalig** bij eerste deployment.

```typescript
// server/seed.ts — draai apart: npx tsx server/seed.ts
import crypto from "crypto";
import bcrypt from "bcrypt";
import { storage } from "./storage";

async function seed() {
  // 1. Eerste tenant aanmaken
  let tenant = await storage.getTenantBySlug("mijn-bedrijf");
  if (!tenant) {
    tenant = await storage.createTenant({ name: "Mijn Bedrijf", slug: "mijn-bedrijf" });
    console.log("Tenant aangemaakt:", tenant.id);
  }

  // 2. Superadmin user aanmaken
  const adminEmail = process.env.ADMIN_EMAIL || "admin@example.com";
  let adminPassword = process.env.ADMIN_PASSWORD;
  let generatedPassword = false;

  if (!adminPassword) {
    if (process.env.NODE_ENV === "production") {
      console.error("FATAL: ADMIN_PASSWORD env var is vereist in productie.");
      process.exit(1);
    }
    // Development: genereer veilig random wachtwoord
    adminPassword = crypto.randomBytes(16).toString("base64url");
    generatedPassword = true;
  }

  let admin = await storage.getUserByEmail(adminEmail);
  if (!admin) {
    const hash = await bcrypt.hash(adminPassword, 12);
    admin = await storage.createUserWithDetails({
      username: adminEmail, email: adminEmail, passwordHash: hash,
      displayName: "Admin", globalRole: "superadmin",
      isActive: true, mustChangePassword: true,
    });
    await storage.addUserToTenant(admin.id, tenant.id, "admin");
    console.log("Superadmin aangemaakt:", admin.id);
  }

  if (generatedPassword) {
    console.log(`\nLogin: ${adminEmail} / ${adminPassword} (auto-generated)`);
  } else {
    console.log(`\nLogin: ${adminEmail} / (uit ADMIN_PASSWORD env var)`);
  }
}

seed();
```

**Gebruik:** `ADMIN_EMAIL=admin@mijndomein.nl ADMIN_PASSWORD=VeiligWachtwoord123 npx tsx server/seed.ts`

**Na de seed:** De superadmin logt in, maakt tenants aan via het admin panel, en nodigt gebruikers uit via het invitation system (§13). Alle volgende tenants worden automatisch aangemaakt via Stripe checkout (§15).

---

## 15. Stripe Billing

> **Principes (niet-onderhandelbaar)**
> - `tenantId` ALTIJD meegeven in `subscription_data.metadata` bij checkout — webhook moet weten bij welke tenant de betaling hoort.
> - Webhook endpoint registreren VOOR CSRF middleware (§7 route volgorde).
> - Webhook signature verificatie via Stripe SDK is VERPLICHT — accepteer nooit ongetekende payloads.
> - Idempotency: `upsertStripeSubscription()` is safe bij dubbele webhook events.
> - Welkomstmail: `welcomeEmailSentAt` timestamp voorkomt dubbele verzending.

### Stripe Price ID mapping

```typescript
// server/services/stripe.ts
function getPriceIds() {
  return {
    starter_monthly: process.env.STRIPE_PRICE_STARTER_MONTHLY,
    starter_annual: process.env.STRIPE_PRICE_STARTER_ANNUAL,
    pro_monthly: process.env.STRIPE_PRICE_PRO_MONTHLY,
    pro_annual: process.env.STRIPE_PRICE_PRO_ANNUAL,
  } as const;
}

// Gebruik: const priceId = getPriceIds()[`${tier}_${interval}`];
// Enterprise is uitgesloten van self-service checkout (vereist sales contact)
```

### Checkout flow
```typescript
import express from "express";
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: "2025-01-27.acacia" });
export const billingRouter = express.Router();
const router = billingRouter;  // Alias voor leesbaarheid

// Service functie — aangeroepen vanuit de route handler hieronder
async function createCheckoutSession(tenantId: string, tier: string, interval: string, email: string, name: string, successUrl: string, cancelUrl: string) {
  // 1. Price ID ophalen uit env vars
  const priceIds = getPriceIds();
  const priceKey = `${tier}_${interval}` as keyof ReturnType<typeof getPriceIds>;
  const priceId = priceIds[priceKey];
  if (!priceId) throw new Error(`Price niet geconfigureerd voor ${tier} ${interval}`);

  // 2. Customer aanmaken/ophalen (idempotent)
  const customers = await stripe.customers.list({ email, limit: 1 });
  const customerId = customers.data[0]?.id || (await stripe.customers.create({ email, name, metadata: { tenantId } })).id;

  // 3. Checkout session
  return (await stripe.checkout.sessions.create({
    customer: customerId,
    mode: "subscription",
    payment_method_types: ["card", "ideal", "sepa_debit"],
    line_items: [{ price: priceId, quantity: 1 }],
    subscription_data: {
      trial_period_days: 14,
      metadata: { tenantId, tier },  // KRITIEK: tenantId in metadata voor webhook
    },
    success_url: successUrl,
    cancel_url: cancelUrl,
    locale: "nl",
    allow_promotion_codes: true,
  })).url;
}

// Billing config (publiek — geen auth nodig, geregistreerd apart in §7 route registratie)
export function billingConfigHandler(req: express.Request, res: express.Response) {
  res.json({
    trialDays: 14,
    tiers: ["starter", "pro", "enterprise"],
    currency: "eur",
  });
}

// Route handler die de service functie aanroept
router.post("/api/billing/checkout", requireAuth, requireTenant, asyncHandler(async (req, res) => {
  const { tier, interval } = req.body;
  const tenantId = getActiveTenantId(req)!;
  const user = getAuthUser(req)!;
  const url = await createCheckoutSession(
    tenantId, tier, interval,
    user.email || "", user.displayName || "",
    `${process.env.FRONTEND_URL}/dashboard?checkout=success`,
    `${process.env.FRONTEND_URL}/pricing?checkout=canceled`,
  );
  res.json({ url });
}));
```

### Webhook (VOOR CSRF middleware!)
```typescript
// server/routes/stripe-webhook.ts — APARTE router (geïmporteerd in §7 route registratie)
export const stripeWebhookRouter = express.Router();

stripeWebhookRouter.post("/api/webhook/stripe", async (req, res) => {
  const signature = req.headers["stripe-signature"] as string;
  const rawBody = req.rawBody as Buffer;  // Van express.json verify (type augmentation in app.ts)

  // 1. Signature verificatie
  const event = stripe.webhooks.constructEvent(rawBody, signature, process.env.STRIPE_WEBHOOK_SECRET!);

  // 2. Deduplicatie (gebruik processedWebhooks uit §22, of inline Map)
  if (await isWebhookDuplicate(event.id)) return res.json({ received: true, duplicate: true });
  markWebhookProcessed(event.id);

  // 3. Verwerken
  switch (event.type) {
    case "customer.subscription.created":
    case "customer.subscription.updated": {
      const sub = event.data.object;
      const tenantId = sub.metadata.tenantId;
      await storage.upsertStripeSubscription({
        tenantId, stripeCustomerId: sub.customer, stripeSubscriptionId: sub.id,
        status: sub.status, tier: sub.metadata.tier || "starter",
        billingInterval: sub.items.data[0]?.price?.recurring?.interval === "year" ? "annual" : "monthly",
        // Stripe SDK v20+: current_period verplaatst naar SubscriptionItem
        currentPeriodStart: sub.items.data[0]?.current_period_start
          ? new Date(sub.items.data[0].current_period_start * 1000) : null,
        currentPeriodEnd: sub.items.data[0]?.current_period_end
          ? new Date(sub.items.data[0].current_period_end * 1000) : null,
        cancelAtPeriodEnd: sub.cancel_at_period_end,
      });
      // Heractiveer tenant bij active/trialing
      if (["active", "trialing"].includes(sub.status)) {
        await storage.updateTenant(tenantId, { isActive: true });
      }
      break;
    }
    case "customer.subscription.deleted": {
      const sub = event.data.object;
      const tenantId = sub.metadata.tenantId;
      await storage.updateStripeSubscription(tenantId, { status: "canceled" });
      await storage.updateTenant(tenantId, { isActive: false });
      break;
    }
  }
  res.json({ received: true });
});
```

### Customer Portal (self-service abonnementsbeheer)

```typescript
// POST /api/billing/portal
router.post("/api/billing/portal", requireAuth, requireTenant, asyncHandler(async (req, res) => {
  const tenantId = getActiveTenantId(req);
  const returnUrl = `${process.env.FRONTEND_URL}/settings#billing`;

  // Stripe customer ophalen via subscription
  const sub = await storage.getStripeSubscription(tenantId);
  if (!sub) return res.status(404).json({ error: "Geen abonnement gevonden" });

  const session = await stripe.billingPortal.sessions.create({
    customer: sub.stripeCustomerId,
    return_url: returnUrl,
  });

  res.json({ url: session.url });
}));

// GET /api/billing/status — huidige subscription status voor frontend (SubscriptionGuard)
router.get("/api/billing/status", requireAuth, requireTenant, asyncHandler(async (req, res) => {
  const tenantId = getActiveTenantId(req)!;
  const status = await getBillingStatus(tenantId);
  res.json(status);
}));
```

### Post-checkout flow (wat gebeurt er na betaling?)

```
Stripe checkout compleet → Stripe stuurt webhook → handleSubscriptionCreated()
  ↓
1. subscription.metadata.tenantId uitlezen (gezet bij checkout)
2. Subscription opslaan in stripe_subscriptions tabel
3. Tenant activeren (isActive = true)
4. Welkomstmail sturen (idempotent via welcomeEmailSentAt)
```

**De tenant is al aangemaakt VOOR de checkout** (bij registratie of door superadmin). De `tenantId` wordt meegegeven in `subscription_data.metadata` zodat de webhook weet bij welke tenant de subscription hoort.

**Idempotency:** De webhook kan meerdere keren dezelfde event ontvangen. `welcomeEmailSentAt` timestamp voorkomt dubbele welkomstmails. `upsertStripeSubscription()` is idempotent per definitie.

### getBillingStatus (voor frontend)
```typescript
export async function getBillingStatus(tenantId: string): Promise<BillingStatus> {
  const tenant = await storage.getTenant(tenantId);
  if (tenant?.billingExempt) {
    return { hasSubscription: true, tier: "enterprise", accessLevel: "full" as const, billingExempt: true };
  }
  const sub = await storage.getStripeSubscription(tenantId);
  if (!sub) return { hasSubscription: false, tier: "starter", accessLevel: "blocked" as const };

  let accessLevel: "full" | "warning" | "blocked";
  let accessMessage: string | undefined;
  const PAST_DUE_GRACE_DAYS = 14;

  switch (sub.status) {
    case "trialing": accessLevel = "full"; break;
    case "active":
      if (sub.cancelAtPeriodEnd && sub.currentPeriodEnd) {
        accessLevel = "warning";
        accessMessage = `Abonnement eindigt op ${sub.currentPeriodEnd.toLocaleDateString("nl-NL")}.`;
      } else {
        accessLevel = "full";
      }
      break;
    case "past_due": {
      const daysPastDue = Math.floor((Date.now() - sub.updatedAt.getTime()) / (1000 * 60 * 60 * 24));
      if (daysPastDue >= PAST_DUE_GRACE_DAYS) {
        accessLevel = "blocked";
        accessMessage = "Betaling al meer dan 14 dagen mislukt.";
      } else {
        accessLevel = "warning";
        accessMessage = `Betaling mislukt. Nog ${PAST_DUE_GRACE_DAYS - daysPastDue} dagen.`;
      }
      break;
    }
    case "unpaid": case "canceled": accessLevel = "blocked"; break;
    default: accessLevel = "blocked";
  }
  return { hasSubscription: true, tier: sub.tier, status: sub.status, accessLevel, accessMessage };
}
```

---

## 16. Subscription Access Control

### Frontend SubscriptionGuard
```tsx
import { Spinner } from "@/components/spinner";
import { useAuth } from "@/hooks/use-auth";
import { useQuery } from "@tanstack/react-query";
import type { BillingStatus } from "@shared/schema";

export function SubscriptionGuard({ children }: { children: React.ReactNode }) {
  const { user } = useAuth();
  const { data: billing, isLoading, error } = useQuery<BillingStatus>({
    queryKey: ["/api/billing/status"],
    enabled: !!user?.activeTenantId,
    refetchInterval: 5 * 60 * 1000,
  });

  if (user?.globalRole === "superadmin") return <>{children}</>;
  if (isLoading) return <Spinner />;
  // BEWUSTE KEUZE: bij technische fouten (billing API down) laten we users door.
  // Alternatief: `return <SubscriptionBlocked />` — maar dat blokkeert alle users bij een API outage.
  // Voor de meeste B2B SaaS is "fail open" beter dan "fail closed" bij billing checks.
  if (error || !billing) return <>{children}</>;
  if (billing.accessLevel === "blocked") return <SubscriptionBlocked />;
  return (
    <>
      {billing.accessLevel === "warning" && <WarningBanner message={billing.accessMessage} />}
      {children}
    </>
  );
}
```

### Pricing pagina (client/src/pages/pricing.tsx)

Publieke route — geen auth nodig. Toont tiers en redirect naar registratie + Stripe checkout.

```tsx
import { useState } from "react";
import { useLocation } from "wouter";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { SUBSCRIPTION_TIERS } from "@shared/schema";
import { Check } from "lucide-react";

type Interval = "monthly" | "annual";

export default function Pricing() {
  const [interval, setInterval] = useState<Interval>("monthly");
  const [, setLocation] = useLocation();

  const tiers = [
    { key: "starter" as const, ...SUBSCRIPTION_TIERS.starter, cta: "Start gratis proefperiode" },
    { key: "pro" as const, ...SUBSCRIPTION_TIERS.pro, cta: "Start gratis proefperiode", popular: true },
    { key: "enterprise" as const, ...SUBSCRIPTION_TIERS.enterprise, cta: "Neem contact op" },
  ];

  return (
    <div className="max-w-5xl mx-auto px-4 py-16">
      <h1 className="text-3xl font-bold text-center mb-4">Kies je abonnement</h1>
      <p className="text-center text-muted-foreground mb-8">14 dagen gratis proberen, geen creditcard nodig.</p>

      {/* Interval toggle */}
      <div className="flex justify-center gap-2 mb-10">
        <Button variant={interval === "monthly" ? "default" : "outline"} onClick={() => setInterval("monthly")}>
          Maandelijks
        </Button>
        <Button variant={interval === "annual" ? "default" : "outline"} onClick={() => setInterval("annual")}>
          Jaarlijks (20% korting)
        </Button>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {tiers.map(tier => {
          const price = interval === "monthly" ? tier.priceMonthly : tier.priceAnnual;
          return (
            <Card key={tier.key} className={tier.popular ? "border-primary ring-2 ring-primary" : ""}>
              <CardHeader>
                {tier.popular && <span className="text-xs font-semibold text-primary uppercase">Populair</span>}
                <CardTitle>{tier.name}</CardTitle>
                <div className="text-3xl font-bold">
                  {price ? `€${interval === "annual" ? Math.round(price / 12) : price}` : "Op maat"}
                  {price && <span className="text-sm font-normal text-muted-foreground">/maand</span>}
                </div>
              </CardHeader>
              <CardContent className="space-y-4">
                <ul className="space-y-2">
                  {tier.features.map(f => (
                    <li key={f} className="flex items-center gap-2 text-sm">
                      <Check className="h-4 w-4 text-green-500" /> {f}
                    </li>
                  ))}
                </ul>
                <Button className="w-full" variant={tier.popular ? "default" : "outline"}
                  onClick={() => tier.key === "enterprise"
                    ? window.location.href = "mailto:sales@yourdomain.com"
                    : setLocation(`/register?tier=${tier.key}&interval=${interval}`)
                  }>
                  {tier.cta}
                </Button>
              </CardContent>
            </Card>
          );
        })}
      </div>
    </div>
  );
}
```

---

## 17. Tier Limits

```typescript
import { storage } from "../storage";
import { getBillingStatus } from "../services/stripe";  // Standalone functie (§15)
import { SUBSCRIPTION_TIERS, type TierLimitResult } from "@shared/schema";

// Generiek: pas entityType en storage-methode aan voor jouw domein
export async function checkEntityLimit(tenantId: string, opts?: { isSuperadmin?: boolean }): Promise<TierLimitResult> {
  if (opts?.isSuperadmin) return { allowed: true };
  const tenant = await storage.getTenant(tenantId);
  if (tenant?.billingExempt) return { allowed: true };
  const billing = await getBillingStatus(tenantId);
  const tierConfig = SUBSCRIPTION_TIERS[billing.tier];
  if (!tierConfig || tierConfig.limits.itemsPerMonth === Infinity) return { allowed: true };
  const count = await storage.getEntityCountForPeriod(tenantId);  // Implementeer per domein
  if (count >= tierConfig.limits.itemsPerMonth) {
    return { allowed: false, reason: `Limiet bereikt: ${count}/${tierConfig.limits.itemsPerMonth}` };
  }
  return { allowed: true };
}

// Gebruik in route:
const check = await checkEntityLimit(tenantId, { isSuperadmin: user.globalRole === "superadmin" });
if (!check.allowed) return res.status(429).json({ error: check.reason });
```

---

# DEEL D: PLATFORM KWALITEIT

---

## 18. Centralized Error Handling

```typescript
// === Error codes ===
export const ErrorCodes = {
  AUTH_NOT_AUTHENTICATED: "AUTH_1001",   // 401
  AUTH_INVALID_CREDENTIALS: "AUTH_1002", // 401
  AUTH_CSRF_MISMATCH: "AUTH_1004",       // 403
  AUTH_MFA_REQUIRED: "AUTH_1005",        // 401
  AUTH_PASSWORD_CHANGE_REQUIRED: "AUTH_1009", // 403
  AUTHZ_FORBIDDEN: "AUTHZ_2001",        // 403
  AUTHZ_NOT_TENANT_MEMBER: "AUTHZ_2002",// 403
  VALIDATION_INVALID_INPUT: "VAL_3001", // 400
  RESOURCE_NOT_FOUND: "RES_4001",       // 404
  RESOURCE_ALREADY_EXISTS: "RES_4002",  // 409
  EXTERNAL_SERVICE_ERROR: "EXT_5001",   // 502
  RATE_LIMIT_EXCEEDED: "RATE_6001",     // 429
  SERVER_INTERNAL_ERROR: "SRV_9001",    // 500
} as const;

// === Status codes per error code ===
const errorStatusCodes: Record<string, number> = {
  AUTH_1001: 401, AUTH_1002: 401, AUTH_1004: 403, AUTH_1005: 401, AUTH_1009: 403,
  AUTHZ_2001: 403, AUTHZ_2002: 403,
  VAL_3001: 400,
  RES_4001: 404, RES_4002: 409,
  EXT_5001: 502,
  RATE_6001: 429,
  SRV_9001: 500,
};

// === Error messages per error code ===
const errorMessages: Record<string, string> = {
  AUTH_1001: "Niet geauthenticeerd",
  AUTH_1002: "Ongeldige inloggegevens",
  AUTH_1004: "CSRF token mismatch",
  AUTH_1005: "MFA verificatie vereist",
  AUTH_1009: "Wachtwoord wijzigen verplicht",
  AUTHZ_2001: "Geen toegang",
  AUTHZ_2002: "Geen lid van deze tenant",
  VAL_3001: "Ongeldige invoer",
  RES_4001: "Niet gevonden",
  RES_4002: "Bestaat al",
  EXT_5001: "Externe service fout",
  RATE_6001: "Te veel verzoeken",
  SRV_9001: "Interne serverfout",
};

// === ApiError class ===
export class ApiError extends Error {
  public statusCode: number;
  constructor(public code: string, public details?: string, public field?: string) {
    super(errorMessages[code] || "Onbekende fout");
    this.statusCode = errorStatusCodes[code] || 500;
  }
  toResponse(requestId?: string) {
    return { error: this.message, code: this.code, details: this.details, timestamp: new Date().toISOString(), requestId };
  }
}

// === Express error handler (registreer als LAATSTE middleware) ===
export function errorHandler(err: Error, req: any, res: any, next: any) {
  const requestId = req.headers["x-request-id"] || `req_${Date.now().toString(36)}`;
  if (err instanceof ApiError) return res.status(err.statusCode).json(err.toResponse(requestId));
  if ("issues" in err && err.name === "ZodError") {
    const zodErr = err as Error & { issues: Array<{ message: string }> };
    const apiErr = new ApiError(ErrorCodes.VALIDATION_INVALID_INPUT, zodErr.issues[0]?.message);
    return res.status(400).json(apiErr.toResponse(requestId));
  }
  // NOOIT stack trace naar client sturen!
  console.error("[Error]", { requestId, error: err.message, stack: err.stack });
  return res.status(500).json(new ApiError(ErrorCodes.SERVER_INTERNAL_ERROR).toResponse(requestId));
}

// === Async handler (voorkomt unhandled promise rejections) ===
export function asyncHandler(fn: (req: any, res: any, next: any) => Promise<any>) {
  return (req: any, res: any, next: any) => Promise.resolve(fn(req, res, next)).catch(next);
}

// BELANGRIJK: Elke async route handler MOET in asyncHandler gewrapped worden.
// Zonder wrapper worden promise rejections niet doorgestuurd naar de error handler.
// Voorbeeld: router.get("/api/x", requireAuth, asyncHandler(async (req, res) => { ... }));
```

---

## 19. Audit Logging

```typescript
// === SECURITY EVENTS (constanten voor audit logging) ===
// Gebruik deze constanten in ELKE logSecurityEvent() aanroep — voorkomt typos en maakt zoeken mogelijk
export const SecurityEvents = {
  LOGIN_SUCCESS: "login_success",
  LOGIN_FAILURE: "login_failure",
  LOGOUT: "logout",
  REGISTER: "register",
  MFA_SENT: "mfa_sent",
  MFA_SUCCESS: "mfa_success",
  MFA_FAILURE: "mfa_failure",
  PASSWORD_CHANGE: "password_change",
  PASSWORD_RESET_REQUEST: "password_reset_request",
  PASSWORD_RESET_COMPLETE: "password_reset_complete",
  ACCOUNT_LOCKED: "account_locked",
  INVITATION_SENT: "invitation_sent",
  INVITATION_ACCEPTED: "invitation_accepted",
  IMPERSONATION_START: "impersonation_start",
  IMPERSONATION_STOP: "impersonation_stop",
  TENANT_SWITCH: "tenant_switch",
  DATA_EXPORT: "data_export",
  ACCOUNT_DELETION_REQUEST: "account_deletion_request",
  ACCOUNT_DELETED: "account_deleted",
} as const;

// Gouden regel: audit logging mag NOOIT de hoofdoperatie laten falen
export async function logSecurityEvent(userId: string | null, event: string, details: {
  tenantId?: string | null; success?: boolean; additionalContext?: Record<string, unknown>;
} = {}) {
  try {
    await storage.createActivityLog({
      tenantId: details.tenantId || "system",
      userId: userId || "system", action: event, field: event,
      context: details.additionalContext || {},
    });
    console[details.success === false ? "warn" : "info"](`[Security] ${event}: user=${userId}`);
  } catch (error) {
    console.error("[Security] Failed to log:", error);  // Log failure, maar crash NIET
  }
}

export function getClientIp(req: any): string {
  return req.headers["x-forwarded-for"]?.split(",")[0]?.trim() || req.connection?.remoteAddress || "unknown";
}
```

---

## 20. Email Service

### Wanneer toepassen
Zodra je MFA, uitnodigingen, password resets, of welkomstmails nodig hebt.

### Setup (server/services/email.ts)

```typescript
import { MailerSend, EmailParams, Sender, Recipient } from "mailersend";

let mailerSendInstance: MailerSend | null = null;

function getMailerSend(): MailerSend | null {
  const apiKey = process.env.MAILERSEND_API_KEY;
  if (!apiKey) {
    console.log("[EmailService] MAILERSEND_API_KEY niet gezet — emails worden niet verstuurd");
    return null;
  }
  if (!mailerSendInstance) {
    mailerSendInstance = new MailerSend({ apiKey });
  }
  return mailerSendInstance;
}

// XSS preventie in emails
function escapeHtml(text: string): string {
  return text.replace(/&/g, "&amp;").replace(/</g, "&lt;")
    .replace(/>/g, "&gt;").replace(/"/g, "&quot;");
}
```

### Email template functies

Elke functie retourneert `{ success: boolean; error?: string }`. Bij ontbrekende API key wordt `{ success: true }` geretourneerd (graceful degradation in development).

```typescript
// 1. Uitnodiging (§13)
export async function sendInvitationEmail(params: {
  email: string; displayName?: string; invitationUrl: string;
  tenantName?: string; expiresAt: Date;
}): Promise<{ success: boolean; error?: string }> {
  const mailer = getMailerSend();
  if (!mailer) return { success: true };  // Dev mode

  const emailParams = new EmailParams()
    .setFrom(new Sender(process.env.MAILERSEND_FROM_EMAIL || "noreply@yourdomain.com", "App"))
    .setTo([new Recipient(params.email, params.displayName)])
    .setSubject(`Uitnodiging voor ${escapeHtml(params.tenantName || "het platform")}`)
    .setHtml(buildInvitationHtml(params));  // HTML template met CTA button + fallback link

  try {
    await mailer.email.send(emailParams);
    return { success: true };
  } catch (error: any) {
    console.error("[EmailService] Uitnodiging mislukt:", error.message);
    return { success: false, error: error.message };
  }
}

// 2. MFA Code (§10)
export async function sendMfaCodeEmail(params: {
  email: string; code: string; displayName?: string;
}): Promise<{ success: boolean; error?: string }> {
  const mailer = getMailerSend();
  if (!mailer) return { success: true };
  const name = escapeHtml(params.displayName || "daar");
  const html = `
  <div style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
              max-width: 560px; margin: 0 auto; padding: 40px 20px;">
    <div style="background: #f8f9fa; border-radius: 8px; padding: 32px;">
      <h1 style="font-size: 20px; color: #1a1a1a; margin: 0 0 16px;">Verificatiecode</h1>
      <p style="font-size: 15px; color: #4a4a4a; line-height: 1.6;">Hallo ${name},</p>
      <p style="font-size: 15px; color: #4a4a4a; line-height: 1.6;">Je verificatiecode is:</p>
      <div style="background: #ffffff; border: 2px solid #e5e5e5; border-radius: 8px;
                  padding: 16px; text-align: center; margin: 16px 0;">
        <span style="font-size: 32px; font-weight: bold; letter-spacing: 8px; color: #1a1a1a;">
          ${escapeHtml(params.code)}
        </span>
      </div>
      <p style="font-size: 13px; color: #888;">Deze code is 10 minuten geldig.
         Als je niet hebt geprobeerd in te loggen, kun je dit bericht negeren.</p>
    </div>
  </div>`;
  const emailParams = new EmailParams()
    .setFrom(new Sender(process.env.MAILERSEND_FROM_EMAIL || "noreply@yourdomain.com", "App"))
    .setTo([new Recipient(params.email)])
    .setSubject("Je verificatiecode")
    .setHtml(html);
  try { await mailer.email.send(emailParams); return { success: true }; }
  catch (error: any) { console.error("[EmailService] MFA code mislukt:", error.message); return { success: false, error: error.message }; }
}

// 3. Password Reset (§9)
export async function sendPasswordResetEmail(params: {
  email: string; resetUrl: string; expiresAt: Date; displayName?: string;
}): Promise<{ success: boolean; error?: string }> {
  const mailer = getMailerSend();
  if (!mailer) return { success: true };
  const name = escapeHtml(params.displayName || "daar");
  const expiry = params.expiresAt.toLocaleDateString("nl-NL", { day: "numeric", month: "long", year: "numeric" });
  const html = `
  <div style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
              max-width: 560px; margin: 0 auto; padding: 40px 20px;">
    <div style="background: #f8f9fa; border-radius: 8px; padding: 32px;">
      <h1 style="font-size: 20px; color: #1a1a1a; margin: 0 0 16px;">Wachtwoord resetten</h1>
      <p style="font-size: 15px; color: #4a4a4a; line-height: 1.6;">Hallo ${name},</p>
      <p style="font-size: 15px; color: #4a4a4a; line-height: 1.6;">
        Er is een wachtwoord reset aangevraagd voor je account.
      </p>
      <table cellpadding="0" cellspacing="0" border="0" style="margin: 16px auto 24px;">
        <tr><td style="background: #2563eb; border-radius: 6px; padding: 12px 32px;">
          <a href="${escapeHtml(params.resetUrl)}"
             style="color: #ffffff; text-decoration: none; font-size: 15px; font-weight: 600;">
            Wachtwoord resetten
          </a>
        </td></tr>
      </table>
      <p style="font-size: 13px; color: #888; word-break: break-all;">
        Link niet zichtbaar? ${escapeHtml(params.resetUrl)}
      </p>
      <hr style="border: none; border-top: 1px solid #e5e5e5; margin: 24px 0;" />
      <p style="font-size: 12px; color: #999;">
        Deze link verloopt op ${expiry}. Heb je geen reset aangevraagd? Negeer dit bericht.
      </p>
    </div>
  </div>`;
  const emailParams = new EmailParams()
    .setFrom(new Sender(process.env.MAILERSEND_FROM_EMAIL || "noreply@yourdomain.com", "App"))
    .setTo([new Recipient(params.email)])
    .setSubject("Wachtwoord resetten")
    .setHtml(html);
  try { await mailer.email.send(emailParams); return { success: true }; }
  catch (error: any) { console.error("[EmailService] Password reset mislukt:", error.message); return { success: false, error: error.message }; }
}

// 4. Welkomstmail (§15 post-checkout)
export async function sendWelcomeEmail(params: {
  email: string; tenantName?: string; tier: SubscriptionTier;
  trialEndsAt?: Date; dashboardUrl: string; displayName?: string;
}): Promise<{ success: boolean; error?: string }> {
  const mailer = getMailerSend();
  if (!mailer) return { success: true };
  const name = escapeHtml(params.displayName || "daar");
  const tenant = escapeHtml(params.tenantName || "je account");
  const tierName = escapeHtml(params.tier.charAt(0).toUpperCase() + params.tier.slice(1));
  const trialInfo = params.trialEndsAt
    ? `<p style="font-size: 15px; color: #4a4a4a;">Je proefperiode loopt tot
       ${params.trialEndsAt.toLocaleDateString("nl-NL", { day: "numeric", month: "long", year: "numeric" })}.</p>`
    : "";
  const html = `
  <div style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
              max-width: 560px; margin: 0 auto; padding: 40px 20px;">
    <div style="background: #f8f9fa; border-radius: 8px; padding: 32px;">
      <h1 style="font-size: 20px; color: #1a1a1a; margin: 0 0 16px;">Welkom bij ${tenant}!</h1>
      <p style="font-size: 15px; color: #4a4a4a; line-height: 1.6;">Hallo ${name},</p>
      <p style="font-size: 15px; color: #4a4a4a; line-height: 1.6;">
        Je ${tierName}-abonnement is actief. Je kunt nu aan de slag.
      </p>
      ${trialInfo}
      <table cellpadding="0" cellspacing="0" border="0" style="margin: 16px auto 24px;">
        <tr><td style="background: #2563eb; border-radius: 6px; padding: 12px 32px;">
          <a href="${escapeHtml(params.dashboardUrl)}"
             style="color: #ffffff; text-decoration: none; font-size: 15px; font-weight: 600;">
            Naar je dashboard
          </a>
        </td></tr>
      </table>
    </div>
  </div>`;
  const emailParams = new EmailParams()
    .setFrom(new Sender(process.env.MAILERSEND_FROM_EMAIL || "noreply@yourdomain.com", "App"))
    .setTo([new Recipient(params.email)])
    .setSubject(`Welkom bij ${params.tenantName || "het platform"}`)
    .setHtml(html);
  try { await mailer.email.send(emailParams); return { success: true }; }
  catch (error: any) { console.error("[EmailService] Welcome email mislukt:", error.message); return { success: false, error: error.message }; }
}

// 5. Account Verwijdering Bevestiging (§23 GDPR)
export async function sendDeletionConfirmEmail(params: {
  email: string; confirmUrl: string; displayName?: string;
}): Promise<{ success: boolean; error?: string }> {
  const mailer = getMailerSend();
  if (!mailer) return { success: true };
  const name = escapeHtml(params.displayName || "daar");
  const html = `
  <div style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
              max-width: 560px; margin: 0 auto; padding: 40px 20px;">
    <div style="background: #f8f9fa; border-radius: 8px; padding: 32px;">
      <h1 style="font-size: 20px; color: #1a1a1a; margin: 0 0 16px;">Account verwijderen</h1>
      <p style="font-size: 15px; color: #4a4a4a; line-height: 1.6;">Hallo ${name},</p>
      <p style="font-size: 15px; color: #4a4a4a; line-height: 1.6;">
        Je hebt een verzoek ingediend om je account te verwijderen. Dit is <strong>onomkeerbaar</strong>.
        Al je data wordt permanent verwijderd.
      </p>
      <table cellpadding="0" cellspacing="0" border="0" style="margin: 16px auto 24px;">
        <tr><td style="background: #EF4444; border-radius: 6px; padding: 12px 32px;">
          <a href="${escapeHtml(params.confirmUrl)}"
             style="color: #ffffff; text-decoration: none; font-size: 15px; font-weight: 600;">
            Account definitief verwijderen
          </a>
        </td></tr>
      </table>
      <hr style="border: none; border-top: 1px solid #e5e5e5; margin: 24px 0;" />
      <p style="font-size: 12px; color: #999;">
        Deze link is 24 uur geldig. Heb je dit niet aangevraagd? Negeer dit bericht — er gebeurt niets.
      </p>
    </div>
  </div>`;
  const emailParams = new EmailParams()
    .setFrom(new Sender(process.env.MAILERSEND_FROM_EMAIL || "noreply@yourdomain.com", "App"))
    .setTo([new Recipient(params.email)])
    .setSubject("Bevestig account verwijdering")
    .setHtml(html);
  try { await mailer.email.send(emailParams); return { success: true }; }
  catch (error: any) { console.error("[EmailService] Deletion confirm mislukt:", error.message); return { success: false, error: error.message }; }
}
```

### HTML Email Template (voorbeeld)

Alle email functies hierboven gebruiken een `buildXxxHtml()` helper voor de email body. Hier is het template-patroon (pas branding aan voor jouw app):

```typescript
function buildInvitationHtml(params: {
  invitationUrl: string; tenantName?: string; displayName?: string; expiresAt: Date;
}): string {
  const name = escapeHtml(params.displayName || "daar");
  const tenant = escapeHtml(params.tenantName || "het platform");
  const expiry = params.expiresAt.toLocaleDateString("nl-NL", {
    day: "numeric", month: "long", year: "numeric",
  });

  // Inline CSS — email clients strippen <style> tags
  return `
  <div style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
              max-width: 560px; margin: 0 auto; padding: 40px 20px;">
    <div style="background: #f8f9fa; border-radius: 8px; padding: 32px;">
      <h1 style="font-size: 20px; color: #1a1a1a; margin: 0 0 16px;">
        Hallo ${name},
      </h1>
      <p style="font-size: 15px; color: #4a4a4a; line-height: 1.6; margin: 0 0 24px;">
        Je bent uitgenodigd om lid te worden van <strong>${tenant}</strong>.
        Klik op de knop hieronder om je account in te stellen.
      </p>

      <!-- CTA Button (table-based voor Outlook compatibiliteit) -->
      <table cellpadding="0" cellspacing="0" border="0" style="margin: 0 auto 24px;">
        <tr>
          <td style="background: #2563eb; border-radius: 6px; padding: 12px 32px;">
            <a href="${escapeHtml(params.invitationUrl)}"
               style="color: #ffffff; text-decoration: none; font-size: 15px; font-weight: 600;">
              Uitnodiging accepteren
            </a>
          </td>
        </tr>
      </table>

      <!-- Fallback link (als knop niet werkt) -->
      <p style="font-size: 13px; color: #888; word-break: break-all;">
        Link niet zichtbaar? Kopieer deze URL:<br/>
        ${escapeHtml(params.invitationUrl)}
      </p>

      <hr style="border: none; border-top: 1px solid #e5e5e5; margin: 24px 0;" />

      <p style="font-size: 12px; color: #999; margin: 0;">
        Deze uitnodiging verloopt op ${expiry}.
        Als je deze uitnodiging niet verwacht, kun je dit bericht negeren.
      </p>
    </div>
  </div>`;
}
```

**Template-regels:**
1. **Inline CSS** — `<style>` tags worden door Gmail, Outlook verwijderd
2. **Table-based buttons** — `<a>` styling werkt niet overal, `<table>` wel
3. **`escapeHtml()` op ALLE user input** — voorkom XSS via displayName, tenantName, URLs
4. **Fallback link** — sommige clients blokkeren buttons
5. **Max-width 560px** — leesbaar op mobiel en desktop

Alle 5 email functies (uitnodiging, MFA, password reset, welkom, account verwijdering) zijn volledig geïmplementeerd hierboven. De structuur (header, body, CTA, footer) is gelijk — alleen de inhoud verschilt.

### Belangrijke patronen

1. **Graceful degradation:** Geen API key = geen crash, alleen een log. Development werkt altijd.
2. **HTML escaping:** Alle user input door `escapeHtml()` voor XSS preventie in emails.
3. **Idempotency:** Welkomstmails checken `welcomeEmailSentAt` timestamp (§15).
4. **Non-blocking:** Email failures worden gelogd maar laten de hoofdoperatie NOOIT falen.
5. **Vervangbaar:** MailerSend → Resend/Postmark/SendGrid = alleen deze file aanpassen.

---

## 21. OAuth Token Encryption

Bestandslocatie: `server/utils/encryption.ts` — geïmporteerd door §4 (storage) en §14 (settings).

```typescript
// server/utils/encryption.ts
import crypto from "crypto";

const ALGORITHM = "aes-256-gcm";
let keyCache: Buffer | null = null;

function getKey(): Buffer {
  if (keyCache) return keyCache;
  const isProduction = process.env.NODE_ENV === "production";
  const key = process.env.ENCRYPTION_KEY || process.env.SESSION_SECRET;
  if (!key || key.length < 16) throw new Error("ENCRYPTION_KEY required (min 16 chars)");
  if (isProduction && process.env.ENCRYPTION_KEY === process.env.SESSION_SECRET) {
    throw new Error("ENCRYPTION_KEY must differ from SESSION_SECRET in production");
  }
  // Configureerbare salt via ENCRYPTION_SALT env var.
  // Bestaande installaties: stel ENCRYPTION_SALT=salt in voor backwards compatibility.
  // Nieuwe installaties: gebruik een sterke random waarde.
  const salt = process.env.ENCRYPTION_SALT || "salt";
  if (salt === "salt" && isProduction) {
    console.warn("[Encryption] WARNING: Using default KDF salt. Set ENCRYPTION_SALT for new installations.");
  }
  keyCache = crypto.scryptSync(key, salt, 32);
  return keyCache;
}

export function encrypt(text: string): string {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(ALGORITHM, getKey(), iv);
  let encrypted = cipher.update(text, "utf8", "hex") + cipher.final("hex");
  return `${iv.toString("hex")}:${cipher.getAuthTag().toString("hex")}:${encrypted}`;
}

function isEncrypted(value: string): boolean {
  const parts = value.split(":");
  return parts.length === 3 && parts[0].length === 32;  // 16 bytes IV = 32 hex chars
}

export function decrypt(text: string): string {
  try {
    const [ivHex, tagHex, ciphertext] = text.split(":");
    if (!tagHex) return text;  // Niet geëncrypt → return as-is (migratie-compat)
    const decipher = crypto.createDecipheriv(ALGORITHM, getKey(), Buffer.from(ivHex, "hex"));
    decipher.setAuthTag(Buffer.from(tagHex, "hex"));
    return decipher.update(ciphertext, "hex", "utf8") + decipher.final("utf8");
  } catch {
    // Plaintext dat niet encrypted lijkt → return as-is (migratie-modus)
    if (!isEncrypted(text)) {
      console.warn("[Encryption] Value appears unencrypted, returning as-is (migration mode)");
      return text;
    }
    // Encrypted data die faalt → THROW, geef NOOIT ciphertext door als plaintext
    throw new Error("Decryption failed — check ENCRYPTION_KEY and ENCRYPTION_SALT configuration");
  }
}
```

---

## 22. Webhook Handling

> **Principes (niet-onderhandelbaar)**
> - Webhook signature verificatie is VERPLICHT — accepteer nooit ongetekende payloads.
> - Deduplicatie cache met TTL — voorkom replay attacks en dubbele verwerking.
> - Webhook routes registreren VOOR CSRF middleware in de route volgorde (§7).
> - Gebruik `rawBody` (buffer) voor signature verificatie, NOOIT geparsed JSON.
> - Altijd 200/json retourneren (ook bij fouten) — voorkom webhook retries door de provider.

### Drie verdedigingslagen: signature + deduplicatie + cache limit

```typescript
// === 1. Svix/HMAC signature verificatie (Recall.ai, etc.) ===
function verifyWebhookSignature(payload: string, headers: any, secret: string) {
  const msgId = headers["webhook-id"] || headers["svix-id"];
  const msgTimestamp = headers["webhook-timestamp"] || headers["svix-timestamp"];
  const msgSignature = headers["webhook-signature"] || headers["svix-signature"];
  if (!msgId || !msgTimestamp || !msgSignature) return { valid: false };

  // Timestamp check: max 5 minuten
  if (Math.abs(Math.floor(Date.now() / 1000) - parseInt(msgTimestamp)) > 300) return { valid: false };

  const secretBytes = secret.startsWith("whsec_")
    ? Buffer.from(secret.substring(6), "base64") : Buffer.from(secret, "utf8");
  const expected = crypto.createHmac("sha256", secretBytes)
    .update(`${msgId}.${msgTimestamp}.${payload}`, "utf8").digest("base64");

  // Meerdere signatures mogelijk (key rotation)
  for (const sig of msgSignature.split(" ")) {
    const [version, signature] = sig.split(",");
    if (version === "v1" && signature === expected) return { valid: true };
  }
  return { valid: false };
}

// === 2. Stripe signature (via SDK) ===
const event = stripe.webhooks.constructEvent(rawBody, signature, webhookSecret);

// === 3. Deduplicatie: L1 (in-memory) + L2 (database) ===
// L1: snelle check binnen hetzelfde process
const processedWebhooks = new Map<string, number>();
const CACHE_MAX = 10000;
const TTL = 60 * 60 * 1000;

function evictIfNeeded() {
  if (processedWebhooks.size >= CACHE_MAX) {
    const keys = Array.from(processedWebhooks.keys());
    for (let i = 0; i < Math.ceil(CACHE_MAX * 0.1); i++) processedWebhooks.delete(keys[i]);
  }
}
setInterval(() => {
  const now = Date.now();
  for (const [id, ts] of processedWebhooks) { if (now - ts > TTL) processedWebhooks.delete(id); }
}, 10 * 60 * 1000);

// L2: persistent across restarts — voorkomt dubbele verwerking na deploys
// Tabel: processed_webhooks (webhook_id VARCHAR PRIMARY KEY, processed_at TIMESTAMPTZ)
async function isWebhookDuplicate(webhookId: string): Promise<boolean> {
  if (processedWebhooks.has(webhookId)) return true;  // L1 (snelst)
  try {
    return await storage.isWebhookProcessed(webhookId);  // L2 (persistent)
  } catch {
    return false;  // L2 failure is non-fatal — L1 beschermt nog steeds
  }
}

function markWebhookProcessed(webhookId: string) {
  evictIfNeeded();
  processedWebhooks.set(webhookId, Date.now());  // L1
  // L2 write is non-blocking — failure breekt de flow niet
  storage.markWebhookProcessed(webhookId).catch(err =>
    console.warn("[Webhook] L2 dedup write failed:", err)
  );
}

// SQL migratie: zie §2 voor de volledige tabel definitie (processed_webhooks)
// Bevat: id UUID PK, webhook_id TEXT UNIQUE, source TEXT, processed_at TIMESTAMPTZ
// Cleanup: DELETE FROM processed_webhooks WHERE processed_at < NOW() - INTERVAL '30 days';
```

### BELANGRIJK: webhook routes registreren VOOR CSRF middleware (zie sectie 7).

---

## 23. GDPR Compliance

> **Principes (niet-onderhandelbaar)**
> - Data export (Artikel 15/20): Gebruiker heeft recht op AL hun persoonlijke data in machine-leesbaar formaat.
> - Account verwijdering (Artikel 17): Altijd 2-staps (email bevestiging) — voorkom accidentele verwijdering.
> - Geen verwijdering als gebruiker enige admin is van een tenant — eerst rol overdragen.
> - Essentiële cookies (sessie, CSRF) vereisen geen expliciete consent, maar je MOET de gebruiker informeren.
> - Geen tracking cookies, analytics, of marketing pixels zonder expliciete opt-in consent.

### Drie verplichte endpoints

```typescript
// server/routes/gdpr.ts
import crypto from "crypto";
import bcrypt from "bcrypt";
import express from "express";
import { requireAuth } from "../middleware/auth";
import { storage } from "../storage";
import { logSecurityEvent, SecurityEvents } from "../utils/audit";
import { sendDeletionConfirmEmail } from "../services/email";
import { validateToken } from "../utils/tokens";

export const gdprRouter = express.Router();

// 1. GET /api/user/data-export (Artikel 15 + 20)
gdprRouter.get("/api/user/data-export", requireAuth, async (req, res) => {
  const userId = req.session.user!.id;
  const user = await storage.getUser(userId);
  if (!user) return res.status(404).json({ error: "Gebruiker niet gevonden" });

  const tenants = await storage.getUserTenants(userId);
  const logs = await storage.getActivityLogs(null, { userId });

  const exportData = {
    exportDate: new Date().toISOString(),
    format: "GDPR Article 15/20 Data Export",
    dataSubject: {
      id: user.id, username: user.username, email: user.email,
      displayName: user.displayName, createdAt: user.createdAt, lastLoginAt: user.lastLoginAt,
    },
    tenantMemberships: tenants.map(t => ({ tenantId: t.tenantId, tenantName: t.tenantName, role: t.role })),
    activityHistory: logs,
    yourRights: {
      rectification: "Neem contact op met support om gegevens te corrigeren",
      erasure: "POST /api/user/delete-account om verwijdering te starten",
      restriction: "Neem contact op met support",
      objection: "Neem contact op met de verwerkingsverantwoordelijke",
    },
  };

  await logSecurityEvent(userId, SecurityEvents.DATA_EXPORT, { success: true }).catch(() => {});
  res.setHeader("Content-Type", "application/json");
  res.setHeader("Content-Disposition", `attachment; filename="data-export-${new Date().toISOString().slice(0, 10)}.json"`);
  res.json(exportData);
});

// 2. GET /api/user/data-summary (overzicht zonder download)
gdprRouter.get("/api/user/data-summary", requireAuth, async (req, res) => {
  const userId = req.session.user!.id;
  const tenants = await storage.getUserTenants(userId);
  const logs = await storage.getActivityLogs(null, { userId });
  res.json({
    tenantCount: tenants.length,
    activityLogCount: logs.length,
    dataCategories: ["Profiel", "Tenant-lidmaatschappen", "Activiteitenlog", "Chat-gesprekken"],
  });
});

// 3. POST /api/user/delete-account (Artikel 17 — stap 1)
gdprRouter.post("/api/user/delete-account", requireAuth, async (req, res) => {
  const userId = req.session.user!.id;
  const user = await storage.getUser(userId);
  if (!user) return res.status(404).json({ error: "Gebruiker niet gevonden" });
  if (user.globalRole === "superadmin") {
    return res.status(403).json({ error: "Superadmins kunnen hun account niet zelf verwijderen" });
  }

  // Check: niet enige admin van een tenant
  const tenants = await storage.getUserTenants(userId);
  for (const t of tenants) {
    if (t.role === "admin") {
      const members = await storage.getUsersForTenant(t.tenantId);
      const otherAdmins = members.filter(m => m.id !== userId && m.role === "admin");
      if (otherAdmins.length === 0) {
        return res.status(409).json({
          error: `Je bent de enige admin van "${t.tenantName}". Draag eerst de admin-rol over.`,
        });
      }
    }
  }

  // Maak 24-uur bevestigingstoken (zelfde patroon als password reset §9)
  const rawToken = crypto.randomBytes(32).toString("base64url");
  const tokenHash = await bcrypt.hash(rawToken, 12);
  const tokenPrefix = rawToken.substring(0, 8);
  await storage.revokePendingTokens(user.email!, "", "account_deletion");
  await storage.createToken({
    tokenHash, tokenPrefix, type: "account_deletion", status: "pending",
    email: user.email!, expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000),
  });
  const confirmUrl = `${process.env.FRONTEND_URL}/delete-account/confirm?token=${rawToken}`;
  await sendDeletionConfirmEmail({ email: user.email!, confirmUrl });
  res.json({ message: "Bevestigingsmail verstuurd. Check je inbox." });
});

// 4. DELETE /api/user/delete-account/confirm (Artikel 17 — stap 2)
gdprRouter.delete("/api/user/delete-account/confirm", requireAuth, async (req, res) => {
  const { token } = req.body;
  if (!token) return res.status(400).json({ error: "Token is verplicht" });

  const result = await validateToken(token, "account_deletion");
  if (!result.valid) return res.status(400).json({ error: "Ongeldig of verlopen token" });

  const userId = req.session.user!.id;
  const user = await storage.getUser(userId);
  if (!user || user.email !== result.token!.email) {
    return res.status(403).json({ error: "Token hoort niet bij dit account" });
  }

  // Verwijder alle data
  const tenants = await storage.getUserTenants(userId);
  for (const t of tenants) {
    await storage.removeUserFromTenant(userId, t.tenantId);
  }
  await storage.deleteUser(userId);
  await storage.updateTokenStatus(result.token!.id, "used", new Date());

  await logSecurityEvent(userId, SecurityEvents.ACCOUNT_DELETED, { success: true }).catch(() => {});
  // Optioneel: bevestigingsmail na verwijdering (gebruiker krijgt al een bevestiging via de response)
  console.info(`[GDPR] Account verwijderd: ${user.email}`);

  // Sessie vernietigen
  req.session.destroy((err) => {
    res.clearCookie("app.sid");
    res.json({ message: "Account en alle data verwijderd." });
  });
});
```

### Cookie consent (essentiële cookies = geen keuze, maar wél informeren)
```tsx
function CookieConsent() {
  const [show, setShow] = useState(!localStorage.getItem("cookie-consent"));
  if (!show) return null;
  return (
    <div className="fixed bottom-0 w-full bg-background border-t p-4">
      <p>Wij gebruiken alleen essentiële cookies (sessie, beveiliging).</p>
      <Button onClick={() => { localStorage.setItem("cookie-consent", "true"); setShow(false); }}>
        Begrepen
      </Button>
    </div>
  );
}
```

---

## 24. Database-Backed Queue Processing

```typescript
import postgres from "postgres";

class JobQueue {
  private sql: ReturnType<typeof postgres>;
  private processingLock: Promise<void> = Promise.resolve();
  private maxRetries = 3;
  private baseDelayMs = 60000;

  constructor(databaseUrl: string) {
    this.sql = postgres(databaseUrl, { ssl: { rejectUnauthorized: false }, prepare: false });
  }

  // Startup: crashed items herstellen
  async initialize() {
    await this.sql`UPDATE job_queue SET status = 'pending', started_at = NULL WHERE status = 'processing'`;
  }

  async enqueue(entityId: string, tenantId: string, jobType?: string) {
    // Check of al in queue
    const existing = await this.sql`SELECT id FROM job_queue WHERE entity_id = ${entityId} AND status IN ('pending', 'processing')`;
    if (existing.length > 0) return;
    await this.sql`INSERT INTO job_queue (entity_id, tenant_id, job_type, status) VALUES (${entityId}, ${tenantId}, ${jobType || 'default'}, 'pending')`;
    this.startProcessing();
  }

  private startProcessing() {
    this.processingLock = this.processingLock.then(() => this.processAll());
  }

  private async processAll() {
    let hasMore = true;
    while (hasMore) {
      hasMore = await this.processNext();
      if (hasMore) await new Promise(r => setTimeout(r, 2000));  // Pauze tussen items
    }
  }

  private isRateLimitError(error: unknown): boolean {
    if (error instanceof Error) {
      return error.message.includes("429") || error.message.toLowerCase().includes("rate limit");
    }
    return false;
  }

  private async processNext(): Promise<boolean> {
    // Atomic: SELECT + UPDATE (concurrent-safe)
    const jobs = await this.sql`
      UPDATE job_queue SET status = 'processing', started_at = NOW()
      WHERE id = (
        SELECT id FROM job_queue WHERE status = 'pending'
        ORDER BY created_at ASC LIMIT 1
        FOR UPDATE SKIP LOCKED
      ) RETURNING *`;
    if (jobs.length === 0) return false;

    try {
      await this.processJob(jobs[0]);  // Implementeer per applicatie
      await this.sql`UPDATE job_queue SET status = 'completed', completed_at = NOW() WHERE id = ${jobs[0].id}`;
    } catch (error) {
      if (this.isRateLimitError(error) && jobs[0].retry_count < this.maxRetries) {
        await this.sql`UPDATE job_queue SET status = 'pending', retry_count = retry_count + 1 WHERE id = ${jobs[0].id}`;
        await new Promise(r => setTimeout(r, this.baseDelayMs * Math.pow(2, jobs[0].retry_count)));
      } else {
        await this.sql`UPDATE job_queue SET status = 'failed', error_message = ${(error as Error).message} WHERE id = ${jobs[0].id}`;
      }
    }
    return true;
  }

  // Override per applicatie: verwerk een job op basis van job_type
  private async processJob(job: { entity_id: string; tenant_id: string; job_type: string }) {
    // Voorbeeld:
    // if (job.job_type === "generate_report") await generateReport(job.entity_id);
    // if (job.job_type === "send_notification") await sendNotification(job.entity_id);
    throw new Error(`Unknown job type: ${job.job_type}`);
  }
}
```

---

# DEEL E: BUILD & DEPLOY

---

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
