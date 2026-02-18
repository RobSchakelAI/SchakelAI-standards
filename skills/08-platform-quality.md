# §18-24 Platform Kwaliteit

> Onderdeel van de B2B SaaS Infrastructure Skills — Deel D: Platform Kwaliteit
>
> Zie [skills/README.md](README.md) voor het overzicht van alle secties.

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

