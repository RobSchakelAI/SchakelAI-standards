# §2 Database Schema

> Onderdeel van de B2B SaaS Infrastructure Skills — Deel A: Fundament
>
> Zie [skills/README.md](README.md) voor het overzicht van alle secties.

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

