# §5-6 Multi-Tenancy & Row Level Security

> Onderdeel van de B2B SaaS Infrastructure Skills — Deel A: Fundament
>
> Zie [skills/README.md](README.md) voor het overzicht van alle secties.

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

