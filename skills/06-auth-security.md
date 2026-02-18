# §7-12 Authenticatie & Security

> Onderdeel van de B2B SaaS Infrastructure Skills — Deel B: Auth & Security
>
> Zie [skills/README.md](README.md) voor het overzicht van alle secties.

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

