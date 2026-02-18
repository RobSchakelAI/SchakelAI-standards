# Security Checklist — Schakel AI

> Non-negotiable security requirements. Elk Schakel-project moet aan AL deze punten voldoen.

## Authenticatie

- [ ] Session-based auth (NOOIT JWT in localStorage)
- [ ] Cookies: httpOnly, secure, sameSite=lax
- [ ] CSRF tokens gesynchroniseerd via response headers
- [ ] MFA support (email-based, tenant-configureerbaar)
- [ ] Wachtwoord minimum 12 karakters, bcrypt met 12+ rounds
- [ ] Account lockout na 5 mislukte pogingen (persistent, niet alleen rate limit)
- [ ] Password history (voorkom hergebruik van laatste 5)

## Multi-Tenant Isolatie

- [ ] Alle storage calls vereisen `tenantId` (type: `string | null`, NOOIT optioneel)
- [ ] `null` = bewuste superadmin/system bypass
- [ ] PostgreSQL Row Level Security (RLS) op alle tenant-scoped tabellen
- [ ] RLS policies gebruiken `current_setting('app.tenant_id')` via `set_tenant_context()`
- [ ] File storage paths bevatten tenant prefix (`{tenantId}/...`)
- [ ] Tier limits gehandhaafd per subscription (items, users)

## Input Validatie

- [ ] Alle user input via Zod schemas
- [ ] Geen raw SQL met user input — altijd parameterized queries
- [ ] File uploads: MIME whitelist (geen SVG/executables), grootte limiet
- [ ] Webhook payloads: signature verificatie + deduplicatie

## Data Bescherming

- [ ] API keys en OAuth tokens encrypted at rest (AES-256-GCM)
- [ ] Geen secrets in logs of error responses
- [ ] Private Supabase buckets met signed URLs (1 uur expiry) voor documenten
- [ ] GDPR: data export, account deletion, cookie consent endpoints

## Audit & Monitoring

- [ ] Security events gelogd via audit utility (login, logout, MFA, data export, role changes)
- [ ] Rate limiting op auth endpoints
- [ ] Gecentraliseerde error codes (AUTH_xxxx, AUTHZ_xxxx, VAL_xxxx, etc.)

## Webhook Security

- [ ] Signature verificatie op alle inkomende webhooks (Stripe, Recall.ai)
- [ ] Deduplicatie met TTL cache (voorkom replay attacks)
- [ ] Webhook routes VOOR body parser middleware (raw body nodig voor signature)

## Security Headers

- [ ] Helmet middleware actief
- [ ] CORS geconfigureerd per environment (geen wildcard in productie)
- [ ] Content-Security-Policy ingesteld

---

*Bij twijfel: strenger is beter. Security fouten zijn duurder dan development-tijd.*
