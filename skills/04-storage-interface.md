# §4 Storage Interface Pattern

> Onderdeel van de B2B SaaS Infrastructure Skills — Deel A: Fundament
>
> Zie [skills/README.md](README.md) voor het overzicht van alle secties.

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

