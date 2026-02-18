# §3 Shared TypeScript Types

> Onderdeel van de B2B SaaS Infrastructure Skills — Deel A: Fundament
>
> Zie [skills/README.md](README.md) voor het overzicht van alle secties.

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

