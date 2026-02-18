# §13-17 Billing & Onboarding

> Onderdeel van de B2B SaaS Infrastructure Skills — Deel C: Billing & Onboarding
>
> Zie [skills/README.md](README.md) voor het overzicht van alle secties.

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

