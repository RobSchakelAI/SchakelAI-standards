# Schakel AI - AI Assistant Instructions

> This file provides context for AI assistants (Claude, Copilot, etc.) working on this codebase.

## Project Overview

Schakel AI is a **B2B SaaS meeting automation platform** that:
- Records meetings via Recall.ai bots (Teams, Zoom, Meet, Webex)
- Fetches transcripts from Fireflies.ai (legacy) or Recall.ai
- Generates AI summaries using Claude (Anthropic)
- Categorizes meetings and assigns owners
- Uploads documents to SharePoint
- Creates tasks in Productive.io
- Provides an "Inzichten" (Insights) chat agent for querying meeting data

## Deployment

- **Planned URL**: `map.schakel.ai` (Meeting Automation Platform)
- **Frontend**: Vercel
- **Backend**: Railway
- **Database**: Supabase (PostgreSQL)

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | React 18, TypeScript, Vite, TailwindCSS, shadcn/ui |
| Backend | Node.js, Express, TypeScript |
| Database | PostgreSQL (Supabase), Row Level Security |
| Auth | Session-based with passport.js, optional email MFA |
| AI | Anthropic Claude API |
| Payments | Stripe (subscriptions, checkout, webhooks) |
| Integrations | Microsoft Graph (SharePoint, Outlook, Calendar), Productive.io, Fireflies.ai, Recall.ai |

## Architecture Principles

### 1. Multi-Tenant Isolation
Every operation is scoped to a tenant. Security is defense-in-depth:
- Application layer: All queries require `tenantId` (not optional)
- Database layer: PostgreSQL Row Level Security (RLS)
- API layer: Session validates tenant membership
- File storage: Tenant-prefixed paths (`logos/{tenantId}/...`)
- Tier limits: Meeting/user counts enforced per subscription tier

```typescript
// Pattern: Always pass tenantId from session context
// tenantId is string | null — null = explicit superadmin/system bypass
const meeting = await storage.getMeeting(id, req.user.activeTenantId);

// Superadmin pattern (cross-tenant access)
const meeting = await storage.getMeeting(id, user.globalRole === "superadmin" ? null : tenantId);
```

### 2. Capabilities System
All actions (pipeline processing, agent queries, mutations) go through the capability system.
- Location: `/server/capabilities/`
- Types: Pipeline (automated processing), Query (agent read-only), Mutation (agent write, requires approval), Security
- Agent capabilities = Query (6) + Mutation (8) = 14 tools total (incl. search_documents, create_werkruimte_document)
- Mutations always use `requiresApproval: true` (human-in-the-loop)
- Pipeline capabilities are NOT exposed to the agent (only their agent wrappers are)

### 3. Session-Based Auth with CSRF
- Cookies for session (httpOnly, secure, sameSite)
- CSRF tokens synced via response headers
- MFA support (email-based, tenant-configurable)

### 4. Subscription Access Control
Access is determined in this order:
1. Superadmin → full access
2. Tenant `billingExempt = true` → full access (enterprise)
3. Tenant `isActive = false` → blocked (subscription canceled)
4. Stripe subscription status → full/warning/blocked

```typescript
// Pattern: Check subscription access for feature gating
const isPro = isSuperadmin || billingStatus?.tier === "pro" ||
              billingStatus?.tier === "enterprise" || billingStatus?.billingExempt;
```

## Key Directories

```
/client                          # React frontend (Vite)
  /src
    /components                  # Reusable UI components (shadcn/ui based)
      /ui                        # Base shadcn components
      werkruimte-dialog.tsx       # Reusable create/edit werkruimte dialog
      werkruimte-logo-upload.tsx # Logo upload with crop/zoom (react-easy-crop)
      werkruimte-documents.tsx   # Document list + file uploads
      werkruimte-timeline.tsx    # Timeline with meetings, documents, events
      werkruimte-chat.tsx        # Werkruimte-scoped AI agent chat
      werkruimte-panel.tsx       # Sidebar popover werkruimte list
      /onboarding               # Onboarding wizard components
      /quick-actions            # Dashboard quick action widgets
    /pages                       # Route pages
      dashboard.tsx              # Dashboard with KPIs and charts
      meetings.tsx               # Vergaderingen (processed meetings list)
      meeting-detail.tsx         # Meeting detail with tabs (notes, tasks, transcript, audio, logs)
      kalender.tsx               # Kalender (scheduled recordings)
      werkruimtes.tsx            # Werkruimtes overview (card grid)
      werkruimte-detail.tsx      # Werkruimte detail (tabs: meetings, documents, insights, timeline, members)
      settings.tsx               # Settings (single scrollable page with sections)
      inzichten.tsx              # AI chat agent (Inzichten)
    /hooks                       # React hooks (useAuth, useToast, useMeetings, etc.)
    /lib                         # Utilities (queryClient, timezone, api helpers)
    /contexts                    # React contexts (theme, auth, filters)

/server                          # Express backend
  /routes                        # API route handlers
    admin.ts                     # Admin/superadmin endpoints
    billing.ts                   # Stripe billing endpoints
    config.ts                    # Tenant settings/config
    gdpr.ts                      # GDPR compliance (export, delete)
    meetings.ts                  # Meeting CRUD
    pipeline.ts                  # Pipeline processing endpoints
    recordings.ts                # Recall.ai bot scheduling
    recall-webhook.ts            # Recall.ai webhook handler
    stripe-webhook.ts            # Stripe webhook handler
    public-auth.ts               # Auth routes (login, register, etc.)
    werkruimtes.ts               # Werkruimte CRUD, membership, meeting assignment, logo, timeline events
    documents.ts                 # Document CRUD + file upload (requireWerkruimteMember)
  /services                      # Business logic
    agent.ts                     # Inzichten chat agent
    claude.ts                    # Claude API wrapper
    email.ts                     # MailerSend email service
    microsoft-graph.ts           # MS365 integration (Calendar, SharePoint, Outlook)
    pipeline.ts                  # Meeting processing pipeline
    recall.ts                    # Recall.ai bot service
    recall-scheduler.ts          # Bot scheduling from calendar
    stripe.ts                    # Stripe subscription management
    oauth.ts                     # OAuth token management
    supabase-file-storage.ts     # Supabase Storage (logos, documents) — signed URLs, private buckets
  /capabilities                  # Agent capability definitions
    categorize.ts                # Meeting categorization
    determineOwner.ts            # Owner assignment
    generateNotes.ts             # AI summary generation
    searchMeetings.ts            # Meeting search for agent
    searchDocuments.ts           # Cross-werkruimte document search for agent
    createWerkruimteDocument.ts  # Agent document creation (requires approval)
    security.ts                  # Security capabilities
  /utils                         # Shared utilities
    errors.ts                    # Centralized error codes
    audit.ts                     # Security event logging
    tier-limits.ts               # Subscription tier enforcement (meetings/users/werkruimtes)
  /middleware                    # Express middleware
  supabase-storage.ts            # Database access layer (with RLS)
  storage.ts                     # Storage interface definitions

/shared                          # Shared TypeScript types/schemas
  schema.ts                      # Zod schemas, Drizzle tables, TypeScript types

/migrations                      # SQL migration files (Drizzle)
  0000_*.sql                     # Initial schema
  0010_password_history.sql      # Password history for security
  0011_stripe_subscriptions.sql  # Stripe integration
  0012_recall_integration.sql    # Recall.ai tables
  0023_werkruimtes.sql           # Werkruimtes (workspaces) with RLS
  0024_primary_werkruimte.sql    # primary_werkruimte_id on meetings
  0025_documents.sql             # Documents table with RLS
  0026_chat_werkruimte_scope.sql # werkruimte_id on chat_threads
  0027_document_uploads.sql      # File upload columns on documents
  0028_werkruimte_events.sql     # Timeline events table with RLS
  0029_werkruimte_logo.sql       # logo_url column on werkruimtes
  0013_billing_exempt.sql        # Enterprise billing exempt flag

/reviews                         # Quality review reports (agent teams output)
```

## Available Commands

```bash
# Development
npm run dev          # Start development server (frontend + backend)
npm run check        # TypeScript type checking (tsc)
npm run build        # Production build (Vite + esbuild)
npm run start        # Start production server

# Testing
npm test             # Run all tests once (vitest)
npm run test:watch   # Run tests in watch mode

# Database
npm run db:push      # Push schema changes to database (drizzle-kit)

# Type checking (recommended before commits)
npx tsc --noEmit     # Full type check without emitting
```

## Critical Integration Points

### 1. Recall.ai Bot Pipeline
```
Calendar Sync → Bot Scheduling → Meeting Recording → Webhook → Pipeline Processing
     ↓              ↓                  ↓                ↓            ↓
microsoft-graph.ts  recall.ts      Recall.ai       recall-webhook.ts  pipeline.ts
                    recall-scheduler.ts                              ↓
                                                              Claude API (categorize, notes)
                                                                     ↓
                                                              SharePoint + Outlook + Tasks
```

### 2. Stripe Billing Flow
```
Pricing Page → Checkout → Webhook → Tenant Provisioning → Access Control
     ↓            ↓          ↓              ↓                   ↓
  /pricing    billing.ts  stripe-webhook.ts  tenant-provisioning.ts  SubscriptionGuard
```

### 3. Multi-Tenant Data Flow
```
Request → Session Auth → Tenant Context → RLS Query → Response
    ↓          ↓              ↓               ↓
  CSRF     passport.js   activeTenantId   withTenant()
```

## Common Patterns

### API Route Pattern
```typescript
router.get("/api/resource/:id", async (req, res) => {
  if (!req.isAuthenticated()) {
    return res.status(401).json({ error: "Niet geauthenticeerd" });
  }
  const tenantId = req.user.activeTenantId;
  const resource = await storage.getResource(req.params.id, tenantId);
  // ...
});
```

### Storage Layer Pattern (with RLS)
```typescript
// For tenant-scoped operations — tenantId is required (string | null)
// null = superadmin/system bypass, string = tenant-scoped with RLS
const result = await this.withTenant(tenantId, async (sql) => {
  return await sql`SELECT * FROM table WHERE id = ${id}`;
});
```

### Tier Limit Enforcement
```typescript
import { checkMeetingLimit, checkUserLimit } from "../utils/tier-limits";

// Before creating a meeting
const limitCheck = await checkMeetingLimit(tenantId, { isSuperadmin });
if (!limitCheck.allowed) {
  return res.status(429).json({ error: limitCheck.reason });
}

// Before inviting a user
const userCheck = await checkUserLimit(tenantId, { isSuperadmin });
if (!userCheck.allowed) {
  return res.status(429).json({ error: userCheck.reason });
}
// Before creating a werkruimte
const werkruimteCheck = await checkWerkruimteLimit(tenantId, { isSuperadmin });
if (!werkruimteCheck.allowed) {
  return res.status(402).json({ error: werkruimteCheck.reason });
}
// Superadmins and billingExempt tenants always bypass limits
```

### Frontend Data Fetching
```typescript
const { data, isLoading } = useQuery({
  queryKey: ["/api/resource"],
  // Uses getQueryFn from queryClient.ts with CSRF handling
});

const mutation = useMutation({
  mutationFn: async (data) => apiRequest("/api/resource", { method: "POST", body: data }),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ["/api/resource"] }),
});
```

## Error Handling

Use the centralized error system in `server/utils/errors.ts`:

```typescript
import { ApiError, ErrorCodes } from "../utils/errors";

// Throw standardized errors
throw new ApiError(ErrorCodes.RESOURCE_NOT_FOUND, "Meeting not found");
throw new ApiError(ErrorCodes.AUTHZ_FORBIDDEN, "No access to this tenant");

// Error codes by category:
// AUTH_xxxx - Authentication errors (1xxx)
// AUTHZ_xxxx - Authorization errors (2xxx)
// VAL_xxxx - Validation errors (3xxx)
// RES_xxxx - Resource errors (4xxx)
// EXT_xxxx - External service errors (5xxx)
// RATE_xxxx - Rate limiting errors (6xxx)
// SRV_xxxx - Server errors (9xxx)
```

## Audit Logging

Use `server/utils/audit.ts` for security-sensitive events:

```typescript
import { logSecurityEvent, SecurityEvents } from "../utils/audit";

await logSecurityEvent(userId, SecurityEvents.LOGIN_SUCCESS, {
  tenantId: user.activeTenantId,
  success: true,
  additionalContext: { email, ipAddress: getClientIp(req) },
});
```

## Security Checklist

When making changes, ensure:
- [ ] All storage calls pass `tenantId` (required — use `string | null`, never omit)
- [ ] User input is validated with Zod schemas
- [ ] Sensitive operations check user roles
- [ ] CSRF protection is maintained for mutations
- [ ] No secrets are logged or exposed in errors
- [ ] Rate limiting is appropriate for the endpoint
- [ ] Security events are logged via audit utility
- [ ] Errors use standardized error codes
- [ ] File uploads include tenant prefix in storage path (`{tenantId}/{werkruimteId}/...`)
- [ ] Document/file uploads use private Supabase bucket with signed URLs (not public URLs)
- [ ] Tier limits checked before creating meetings, inviting users, or creating werkruimtes
- [ ] Werkruimte operations verify tenant ownership before mutations
- [ ] Werkruimte-scoped endpoints use `requireWerkruimteMember` middleware

## Language & Localization

The UI is in **Dutch (Nederlands)**. Key translations:
- Dashboard = Dashboard
- Meetings = Vergaderingen
- Settings = Instellingen
- Users = Gebruikers
- Security = Beveiliging
- Save = Opslaan
- Delete = Verwijderen

## Testing

Automated test suite using **Vitest** with 69 tests across 4 test files.

### Setup
- **Config**: `vitest.config.ts`
- **Run once**: `npm test`
- **Watch mode**: `npm run test:watch`

### Test Files

| File | Tests | Coverage |
|------|-------|----------|
| `server/storage.test.ts` | 23 | Tenant isolation: meetings, settings, categories, chat threads, scheduled recordings. Meeting creation defaults. |
| `server/utils/tier-limits.test.ts` | 16 | Tier limit enforcement: meeting/user limits per tier, superadmin bypass, billingExempt bypass, SUBSCRIPTION_TIERS constants. |
| `server/auth-security.test.ts` | 10 | Session regeneration, MFA crypto, CSRF, rate limiting, account lockout. |
| `server/utils/encryption.test.ts` | 20 | AES-256-GCM encryption/decryption, salt configuration, backwards compatibility. |

### Writing New Tests
- **Storage tests**: Use `MemStorage` directly (no database needed). Create tenants/users, then test operations are scoped correctly.
- **Service/utility tests**: Use `vi.mock()` for external dependencies (storage, APIs).
- E2E tests for auth flows (recommended, not yet implemented)

## Environment Variables

Required for production:
```
DATABASE_URL          # PostgreSQL connection string
SESSION_SECRET        # Secure random string for sessions
ANTHROPIC_API_KEY     # Claude API key
ENCRYPTION_KEY        # 32-byte hex for encrypting secrets
FRONTEND_URL          # For email links (e.g., https://map.schakel.ai)

# Stripe (required for billing)
STRIPE_SECRET_KEY           # sk_live_... or sk_test_...
STRIPE_WEBHOOK_SECRET       # whsec_...
STRIPE_PRICE_STARTER_MONTHLY  # price_...
STRIPE_PRICE_STARTER_ANNUAL   # price_...
STRIPE_PRICE_PRO_MONTHLY      # price_...
STRIPE_PRICE_PRO_ANNUAL       # price_...

# Recall.ai (required for meeting bot)
RECALL_API_KEY              # API key from Recall.ai dashboard
RECALL_WEBHOOK_SECRET       # Webhook secret for signature verification
RECALL_REGION               # Optional: us-east-1, us-west-2, eu-central-1, ap-northeast-1

# Email (required for notifications)
MAILERSEND_API_KEY          # API key from MailerSend
MAILERSEND_FROM_EMAIL       # Default: noreply@schakel.ai
```

## Common Tasks

### Adding a New Settings Section
1. Add to `settingsSubItems` in `/client/src/components/app-sidebar.tsx`
2. Add section ID to `sectionIds` in `/client/src/pages/settings.tsx`
3. Add section component with `id={sectionId}` attribute
**Note:** Werkruimte management was moved OUT of settings into dedicated werkruimte pages (Feb 2026). Use `WerkruimteDialog` for create/edit.

### Adding a New API Endpoint
1. Create route in appropriate file in `/server/routes/`
2. Add types/schemas to `/shared/schema.ts`
3. Register router in `/server/routes/index.ts`

### Adding a New Capability (for Agent)
1. Create capability file in `/server/capabilities/` using `createAgenticCapability()`
   - Include: `displayName`, `icon`, `tags`, `whenToUse`, `whenNotToUse`
   - Set `requiresApproval: true` for write operations
   - Set `isReadOnly: false` for mutations
2. Register in `/server/capabilities/index.ts`:
   - Add to `mutationCapabilities` (or `queryCapabilities` for read-only)
   - Add import + export statements
   - Use `agent`-prefixed keys if wrapping a pipeline capability (avoid key conflicts)
3. Update system prompt in `server/services/agent.ts` if workflow guidance needed
4. Frontend popover updates automatically via `/api/chat/tools` API

**Wrapping pipeline capabilities:** Use thin wrappers in `agentMutations.ts` that take simple input
(e.g., `meetingId`) and build the full pipeline input. See existing wrappers as pattern.

## Recent Changes (February 2026)

### Werkruimte Intelligence Hub + Feedback Fixes (15-16 Feb 2026 - Complete ✅)

Werkruimtes evolved into intelligence hubs with documents, timeline, scoped AI agent, and cross-werkruimte search. Feedback fixes added: full-width layout, meeting filters, document uploads, timeline events, logo upload with crop/zoom, CRUD management moved from settings to werkruimte pages, and destructive delete confirmation.

**Key new components:**
- `werkruimte-dialog.tsx` — Reusable create/edit dialog (used in 3 places: overview page, sidebar panel, detail page)
- `werkruimte-logo-upload.tsx` — Logo upload with react-easy-crop (256×256 PNG output)
- `werkruimte-documents.tsx` — Document list + file upload (private bucket, signed URLs)
- `werkruimte-timeline.tsx` — Combined timeline: meetings + documents + manual events
- `werkruimte-chat.tsx` — Werkruimte-scoped AI agent chat

**Key patterns:**
- Document files stored in **private** Supabase bucket (`werkruimte-documents`), accessed via signed URLs (1 hour expiry)
- Werkruimte logos stored in **public** `branding` bucket at `werkruimte-logos/{tenantId}/{werkruimteId}/`
- Timeline uses UNION ALL query combining 3 sources (meetings, documents, werkruimte_events)
- `requireWerkruimteMember` middleware on all werkruimte-scoped endpoints (admin bypass built-in)
- Delete werkruimte shows destructive AlertDialog with explicit cascade warning (documents, timeline, chats deleted; meetings detached)

### Werkruimtes — Intra-Tenant Access Control (13 Feb 2026 - Complete ✅)
Werkruimtes (workspaces) add per-person access control within a tenant. Meetings without a werkruimte remain visible to everyone (backwards compatible).

**Concepts:**
- **Werkruimte**: Container for meetings + determines access via membership. Types: client, project, internal, general.
- **Categorie** (existing): Cross-cutting label, orthogonal to werkruimtes, NOT a security boundary.

**6 phases implemented:**
1. **Backend Foundation**: 3 tables (werkruimtes, werkruimte_leden, meeting_werkruimtes) with RLS, 16 IStorage methods, werkruimtes.ts routes (12 endpoints), tier limits
2. **Settings UI**: CRUD + member management in settings page (Pro+ tier-gated)
3. **Meeting Assignment**: Werkruimte badges on meeting detail, assign/remove via dropdown
4. **Pages + Navigation**: `/werkruimtes` overview (card grid), `/werkruimtes/:id` detail page, sidebar nav
5. **Access Filtering**: `getMeetingsLight` scoped by werkruimte membership for non-admins
6. **Agent Scoping**: `searchMeetings` capability respects werkruimte access via `CapabilityContext.isAdmin`

**Key files:**
- `migrations/0023_werkruimtes.sql` — Tables, RLS, indexes
- `server/routes/werkruimtes.ts` — 12 API endpoints
- `client/src/pages/werkruimtes.tsx` — Overview page
- `client/src/pages/werkruimte-detail.tsx` — Detail page
- `shared/schema.ts` — Types, schemas, tier limits

**Tier limits:** Starter=0 (disabled), Pro=10, Enterprise=unlimited

### Agent Capabilities Expansion (9 Feb 2026 - Complete ✅)
All platform actions are now available as agent capabilities ("alles is een capability").

**12 agent capabilities total:**
- **Query (5):** search_meetings, get_meeting_details, list_categories, list_clients, get_meeting_stats
- **Mutation (7):** update_meeting_summary, update_meeting_details, schedule_recording, create_outlook_draft, generate_document, upload_sharepoint, create_tasks

**New files:**
- `server/capabilities/scheduleRecording.ts` — Notulist inplannen (Recall.ai bot via agent)
- `server/capabilities/agentMutations.ts` — 4 thin wrappers for pipeline capabilities

**Key patterns:**
- Agent wrappers take simple input (e.g., `{ meetingId }`) and build full pipeline input
- `getCapabilityByName()` looks up by `.name` property (snake_case), not just object key
- `getAgentCapabilities()` filters from `agentCapabilities` only (not pipeline capabilities)
- Frontend popover is dynamic based on `/api/chat/tools` API response

### Security Hardening (9 Feb 2026 - Complete ✅)
Four-phase security hardening based on quality review findings:
- **Fase 1**: Inzichten Agent fix — replaced `services: null as any` with real service instances
- **Fase 2**: tenantId required — 19 IStorage methods changed from `tenantId?` to `tenantId: string | null`, 40+ callers updated. `null` = explicit superadmin bypass
- **Fase 3**: File storage tenant isolation — logo/avatar paths now include tenant prefix
- **Fase 4**: Tier limit enforcement — `server/utils/tier-limits.ts` enforces meetings/month and users/tenant limits per subscription tier

Key patterns:
- `getActiveTenantId(req)` returns `string | null` — use `!` after `requireTenant` middleware
- Superadmin bypass: pass `null` as tenantId for cross-tenant access
- `checkMeetingLimit()` / `checkUserLimit()` — bypass for superadmin and `billingExempt`
- `getMeetingCountForMonth(tenantId)` — efficient SQL COUNT for tier enforcement

### Recall.ai Meeting Bot Integration (Complete & Tested ✅)
- **Meeting Recording**: Bots join Teams, Zoom, Meet, Webex meetings automatically
- **Transcript Processing**: Webhook receives transcripts and creates meetings
- **AI Pipeline**: Full processing (categorization, notes, documents, SharePoint, tasks)
- **Calendar Sync**: Microsoft 365 calendar integration for automatic scheduling
- **Manual Scheduling**: Users can manually add bot to meetings via URL
- **Early Join**: `botJoinLeadTimeMinutes` setting (default: 5 minutes before meeting)
- **Bot Detection**: Automatic leave when only bots remain (no humans)
- **Files**: `server/services/recall.ts`, `server/routes/recall-webhook.ts`, `server/services/pipeline.ts`

#### Key Technical Details (5 Feb 2026 fixes)
- **JSONB Storage**: Use `sql.json()` not `JSON.stringify()` for PostgreSQL JSONB columns
- **Array Safety**: Always use `Array.isArray()` checks for API data that should be arrays
- **Transcript Context**: Load transcript into pipeline context when `fetch_transcript` is skipped
- **Webhook Deduplication**: Cache with 1-hour TTL prevents replay attacks; dev endpoint to clear

#### CRITICAL: EU Region (eu-central-1) Compatibility
**We use the EU region (`RECALL_REGION=eu-central-1`). Some API features differ!**

| Feature | US Region | EU Region | Notes |
|---------|-----------|-----------|-------|
| `transcription_options` | Supported | **NOT SUPPORTED** | Don't send this field |
| `/bot/{id}/transcript` | Works | **DEPRECATED** | Use bot recordings method |
| `bot_image` | Supported | Supported | For bot avatar |
| `bot_detection` | Supported | Supported | For auto-leave when only bots remain |

**Transcript Retrieval in EU:**
Instead of `/bot/{id}/transcript`, use:
1. GET `/bot/{id}` to get bot details
2. Extract `recordings[].media_shortcuts.transcript.data.download_url`
3. Download transcript from that URL

**Implementation in `recall.ts`:**
- `createBot()`: Conditionally excludes `transcription_options` for EU
- `getTranscript()`: Uses `getTranscriptViaBot()` method for EU region
- Always verify endpoints work in EU before using!

**Reference**: [Recall.ai EU Docs](https://docs.recall.ai/reference/transcript_retrieve?region=eu-central-1)

### Simplified Meeting Recordings UI (Complete)
- **Single List**: "Komende Meetings" displays all upcoming meetings with toggle buttons
- **Per-Meeting Toggle**: Users can add/remove bot from any meeting unlimited times
- **Auto-Join Modes**:
  - Auto-join ON: All meetings get bot automatically, user can exclude specific ones
  - Auto-join OFF: No meetings get bot by default, user can include specific ones
- **New Endpoints**: `POST /api/recordings/toggle-bot/:eventId`, `POST /api/recordings/refresh`
- **Response Fields**: `hasBotId`, `withBotCount`, `scheduledButNoBotCount`, `warning`
- **Files**: `client/src/components/upcoming-recordings.tsx`, `server/routes/recordings.ts`

### Timezone Handling (Fixed)
- **Storage**: All times stored in UTC (ISO 8601)
- **Display**: Uses `formatInTimeZone()` from date-fns-tz with browser's timezone
- **Detection**: Dynamic via `Intl.DateTimeFormat().resolvedOptions().timeZone`
- **File**: `client/src/lib/timezone.ts`

### Billing-Exempt Feature (Complete)
- **Enterprise Support**: `billing_exempt` flag on tenants bypasses Stripe checks
- **Tenant Lifecycle**: `is_active` flag controls access (synced with Stripe subscription)
- **Database Migration**: `migrations/0013_billing_exempt.sql`

### Welcome & Onboarding Flow (Complete)
- **Welcome Modal**: Shows after Stripe checkout with tier-specific features
- **Per-User Onboarding**: LocalStorage key includes userId (`schakel-onboarding-{userId}`)
- **10 Steps**: Removed redundant welcome step (was handled by modal)
- **Files**: `client/src/components/welcome-modal.tsx`, `client/src/components/onboarding/onboarding-guide.tsx`

---

## Recent Changes (January 2026)

### Stripe Billing (Complete)
- **Subscription Tiers**: Essentials (€29/mo), Professional (€79/mo), Enterprise (custom)
- **Payment Methods**: Credit cards, iDEAL, SEPA Direct Debit
- **Trial Period**: 14 days free trial on all plans
- **Customer Portal**: Self-service subscription management via Stripe
- **Webhooks**: Automatic subscription status sync
- **Environment Variables**: `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_*`

### SaaS Readiness (Complete)
- **Legal Pages**: Privacy Policy (`/privacy`), Terms of Service (`/terms`), DPA (`/dpa`)
- **Cookie Consent**: GDPR cookie banner (essential cookies only)
- **GDPR Endpoints**: Data export, data summary, account deletion with email confirmation
- **Error Handling**: Centralized error codes (AUTH_xxxx, AUTHZ_xxxx, etc.)
- **Audit Logging**: Security events (login, logout, MFA, data export)
- **Password History**: Prevents reuse of last 5 passwords

### Authentication
- **MFA Implementation**: Email-based MFA with cross-domain support (mfaTokenStore)
- **Password Security**: Bcrypt hashing, minimum 12 chars, complexity requirements
- **CSRF Protection**: Synchronous double-submit pattern with header sync

### Data Security
- **Row Level Security**: PostgreSQL RLS for tenant isolation
- **Multi-tenant**: All queries scoped by tenantId
- **Encryption**: API keys encrypted at rest

## Public Routes (No Auth Required)

| Route | Page | Purpose |
|-------|------|---------|
| `/` | Landing | Marketing page for unauthenticated users |
| `/pricing` | Pricing | Subscription tier selection and checkout |
| `/privacy` | Privacy Policy | GDPR-compliant privacy policy (Dutch) |
| `/terms` | Terms of Service | Legal terms and conditions (Dutch) |
| `/dpa` | Data Processing Agreement | B2B Verwerkersovereenkomst (Dutch) |
| `/login` | Login | Authentication |
| `/accept-invitation/*` | Accept Invitation | User onboarding |
| `/forgot-password` | Forgot Password | Password reset request |
| `/reset-password/*` | Reset Password | Password reset completion |

## Billing API Endpoints

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/billing/status` | GET | Yes | Get current subscription status |
| `/api/billing/checkout` | POST | Yes | Create Stripe Checkout session, returns URL |
| `/api/billing/portal` | POST | Yes | Create Stripe Customer Portal session |
| `/api/billing/config` | GET | No | Get billing configuration (trial days, etc.) |
| `/api/webhook/stripe` | POST | No* | Stripe webhook endpoint (*verified via signature) |

## GDPR API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/user/data-export` | GET | Download all personal data as JSON |
| `/api/user/data-summary` | GET | View summary of stored data |
| `/api/user/delete-account` | POST | Request account deletion (sends email) |
| `/api/user/delete-account/confirm` | DELETE | Confirm and execute deletion |

## Werkruimtes API Endpoints

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/werkruimtes` | GET | Yes | List all werkruimtes for current tenant |
| `/api/werkruimtes` | POST | Admin | Create werkruimte (tier-limited) |
| `/api/werkruimtes/:id` | GET | Yes | Get werkruimte with member count |
| `/api/werkruimtes/:id` | PATCH | Admin | Update werkruimte |
| `/api/werkruimtes/:id` | DELETE | Admin | Delete werkruimte (not default) |
| `/api/werkruimtes/:id/leden` | GET | Yes | List werkruimte members |
| `/api/werkruimtes/:id/leden` | POST | Admin | Add member to werkruimte |
| `/api/werkruimtes/:id/leden/:userId` | DELETE | Admin | Remove member from werkruimte |
| `/api/werkruimtes/:id/meetings` | GET | Yes | List meetings in werkruimte |
| `/api/meetings/:meetingId/werkruimtes` | GET | Yes | Get werkruimtes for a meeting |
| `/api/meetings/:meetingId/werkruimtes` | POST | Yes | Assign meeting to werkruimte |
| `/api/meetings/:meetingId/werkruimtes/:werkruimteId` | DELETE | Yes | Remove meeting from werkruimte |
| `/api/werkruimtes/:id/logo` | POST | Admin | Upload werkruimte logo (Multer, crop-ready) |
| `/api/werkruimtes/:id/logo` | DELETE | Admin | Remove werkruimte logo |
| `/api/werkruimtes/:id/timeline` | GET | Member | Get combined timeline (meetings + documents + events) |
| `/api/werkruimtes/:id/events` | POST | Member | Create timeline event |
| `/api/werkruimtes/:id/events/:eventId` | PATCH | Member | Update timeline event |
| `/api/werkruimtes/:id/events/:eventId` | DELETE | Member | Delete timeline event |

## Documents API Endpoints

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/werkruimtes/:werkruimteId/documents` | GET | Member | List documents in werkruimte |
| `/api/werkruimtes/:werkruimteId/documents` | POST | Member | Create text document |
| `/api/werkruimtes/:werkruimteId/documents/upload` | POST | Member | Upload file document (Multer + signed URL) |
| `/api/werkruimtes/:werkruimteId/documents/:id` | GET | Member | Get single document |
| `/api/werkruimtes/:werkruimteId/documents/:id` | PATCH | Member | Update document |
| `/api/werkruimtes/:werkruimteId/documents/:id` | DELETE | Member | Delete document (+ file from storage) |
| `/api/werkruimtes/:werkruimteId/documents/:id/download` | GET | Member | Get signed download URL |

### Security Model
- All endpoints require authentication (`requireAuth`) and tenant context (`requireTenant`)
- Write operations on werkruimtes require tenant admin (`requireTenantAdmin`)
- Werkruimte-scoped resources (documents, events, timeline) use `requireWerkruimteMember` (admin bypass built-in)
- Meeting assignment/removal requires authentication only (any tenant member)
- All operations verify resource ownership (werkruimte/meeting belongs to tenant)
- Default "Algemeen" werkruimte cannot be deleted or renamed
- Member additions verify user belongs to tenant
- Tier limits: Starter=0 (disabled), Pro=10, Enterprise=unlimited
- Document uploads: private Supabase bucket, signed URLs (1 hour), MIME whitelist (no SVG/executables)
- Logo uploads: public branding bucket, MIME whitelist (PNG/JPEG/GIF/WebP), 5MB max client-side

## Recordings API Endpoints (Recall.ai)

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/recordings` | GET | Yes | List scheduled recordings for tenant |
| `/api/recordings` | POST | Yes | Schedule a new recording (manual or calendar) |
| `/api/recordings/upcoming` | GET | Yes | Get upcoming recordings for dashboard widget |
| `/api/recordings/status` | GET | Yes | Get recording status summary |
| `/api/recordings/importable` | GET | Yes | List recordings that can be imported (have transcript, no meeting) |
| `/api/recordings/import` | POST | Yes | Batch import recordings and create meetings |
| `/api/recordings/calendar-events` | GET | Yes | Get calendar events with bot status fields |
| `/api/recordings/toggle-bot/:eventId` | POST | Yes | Toggle bot on/off for a specific meeting |
| `/api/recordings/refresh` | POST | Yes | Combined calendar fetch + bot scheduling |
| `/api/recordings/:id` | GET | Yes | Get specific recording details |
| `/api/recordings/:id` | DELETE | Yes | Cancel a scheduled recording |
| `/api/recordings/:id/reprocess` | POST | Yes | Reprocess recording (fetch transcript, create/update meeting) |
| `/api/webhook/recall` | POST | No* | Recall.ai webhook endpoint (*verified via signature) |

### Calendar Events Response Fields
The `/api/recordings/calendar-events` endpoint returns events with:
- `hasBotId`: Whether a bot is currently scheduled for this meeting
- `withBotCount`: Number of meetings with bots scheduled
- `scheduledButNoBotCount`: Number of scheduled recordings without bot IDs
- `warning`: Optional warning message about sync issues

### Bot Toggle Behavior
- **Auto-join ON**: All meetings get bot automatically; user can remove specific ones
- **Auto-join OFF**: No meetings get bot by default; user can add specific ones
- Users can toggle bots on/off unlimited times for any meeting

## Files to Always Check

When making changes:
- `CONTEXT.md` - Current project status and TODOs
- `CLAUDE.md` - This file, AI assistant instructions
- `shared/schema.ts` - Type definitions
- `server/routes/index.ts` - Route registration
- `client/src/App.tsx` - Route definitions

---

## Agent Teams Richtlijnen

Deze sectie is voor Claude Code agent teammates die parallel werken aan dit project.

### Werkafspraken

1. **Rapportage**: Schrijf bevindingen als markdown in de `/reviews` directory
   - Bestandsnaam: `{agent-naam}-review.md` (bijv. `tenant-guardian-review.md`)
   - Gebruik duidelijke secties: Samenvatting, Bevindingen, Aanbevelingen
   - Prioriteer bevindingen: CRITICAL, HIGH, MEDIUM, LOW

2. **Zelfstandig uitvoerbaar**: Taken moeten zelfstandig af te ronden zijn
   - Lees eerst relevante code voordat je conclusies trekt
   - Gebruik `npm run check` om type errors te valideren
   - Maak GEEN code wijzigingen zonder expliciete opdracht

3. **Bij twijfel**: Rapporteer aan de lead
   - Als je iets vindt dat buiten je scope valt, noteer het
   - Als je blokkerende issues tegenkomt, communiceer via task list
   - Vraag niet om gebruikersinput - rapporteer bevindingen

4. **Scope bewaken**: Blijf binnen je toegewezen reviewgebied
   - Noteer cross-cutting concerns voor andere teammates
   - Dupliceer geen werk van andere agents

### Review Output Format

```markdown
# [Agent Naam] Security Review

## Samenvatting
[2-3 zinnen over de algemene staat]

## Bevindingen

### CRITICAL
- [ ] **[Titel]**: [Beschrijving + bestandslocatie + aanbeveling]

### HIGH
- [ ] **[Titel]**: [Beschrijving]

### MEDIUM
- [ ] **[Titel]**: [Beschrijving]

### LOW
- [ ] **[Titel]**: [Beschrijving]

## Positieve Observaties
[Wat is goed geïmplementeerd]

## Aanbevelingen
[Geprioriteerde actielijst]
```

### Belangrijke Bestanden per Review Area

**Multi-Tenant Security:**
- `server/supabase-storage.ts` - Database queries met RLS
- `server/routes/*.ts` - API endpoints met tenantId checks
- `migrations/005_rls_tenant_isolation.sql` - RLS policies
- `server/capabilities/security.ts` - Security capabilities

**Pipeline & Processing:**
- `server/services/pipeline.ts` - Pipeline orchestration
- `server/services/pipelineQueue.ts` - Queue management
- `server/routes/recall-webhook.ts` - Webhook processing
- `server/capabilities/*.ts` - Pipeline stage implementations

**Stripe & Billing:**
- `server/services/stripe.ts` - Stripe API integration
- `server/routes/billing.ts` - Billing endpoints
- `server/routes/stripe-webhook.ts` - Webhook handler
- `server/services/tenant-provisioning.ts` - New tenant setup
- `client/src/components/subscription-guard.tsx` - Access control

## Questions?

Check `CONTEXT.md` for current status and planned work.
