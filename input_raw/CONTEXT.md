# Project Context - Schakel AI

> **Laatst bijgewerkt:** 2026-02-17 (Dev testing ronde 2 — alle feedback verwerkt)
> **Huidige focus:** Testen op dev, daarna merge naar productie
> **Deployment:** Push naar `dev` branch → Railway auto-deploy, Vercel auto-deploy

---

## DEV → PRODUCTIE DEPLOYMENT PLAN

### Status
- **233 commits** op `dev` die niet op `main` staan
- **200 bestanden** gewijzigd (~58.600 regels toegevoegd)
- **21 nieuwe database migraties** (0010–0030)
- **Fase:** Eerst testen op dev, daarna merge

### Risico-inschatting

| Risico | Impact | Kans | Mitigatie |
|--------|--------|------|-----------|
| Migraties falen op prod DB | Hoog | Laag | Backup + handmatig testen |
| Frontend/backend mismatch | Hoog | Laag | Backend eerst deployen |
| Stripe tier checks blokkeren gebruikers | Medium | Medium | billingExempt flag checken |
| Soft-delete breekt bestaande flows | Medium | Laag | Alleen deleteUser gewijzigd |
| Recall webhook status mapping | Medium | Laag | recording_done → recording i.p.v. processing |
| RLS policies blokkeren queries | Hoog | Laag | Al getest op dev |

### Stap 1: Testen op Dev (NU)

Volledige test-checklist voor alle ontwikkelingen:

#### Basis functionaliteit
- [ ] Login/logout werkt
- [ ] Dashboard laadt met correcte data
- [ ] Meetings lijst toont vergaderingen
- [ ] Meeting detail pagina opent zonder errors

#### Kalender (punt 21)
- [x] Kalender pagina toont events **3 weken** vooruit (504 uur)
- [x] Events chronologisch gesorteerd (was: ongesorteerd object grouping)
- [ ] "Handmatig toevoegen" knop opent dialoog
- [ ] Bot scheduling via URL werkt (Teams/Zoom/Meet)

#### Categorieën (punten 2, 3, 6)
- [x] Categorie aanmaken: beschrijving is verplicht (foutmelding bij leeg)
- [x] Mappad veld is VERBORGEN als geen SharePoint connectie
- [x] Mappad veld is ZICHTBAAR als SharePoint wel geconfigureerd
- [x] Zekerheidsdrempel is volledig VERBORGEN (interne instelling)
- [x] "Geen cloud opslag" melding verborgen als geen SharePoint
- [x] Categorie popup: 3 knoppen passen netjes (geen overflow)
- [x] AI instructies generator: "Genereer met AI" knop per categorie-prompt
- [x] AI instructies: optioneel extra context veld voor sturing

#### Sidebar & navigatie (punten 7, 8)
- [x] Productive sidebar item is VERBORGEN als niet geconfigureerd
- [x] Productive sidebar item is ZICHTBAAR als geconfigureerd
- [x] Onboarding: start navigeert naar #meeting-bot (eerste stap)
- [x] Onboarding: "Volgende" navigeert naar de juiste volgende stap
- [ ] Onboarding: terug-knop werkt correct

#### Admin pagina (punten 15, 16, 17)
- [x] Platform logging: data laadt (was kapot door ontbrekende getFullUrl)
- [x] Platform logging: statistieken tiles/cards tonen
- [x] Superadmin promoten: AlertDialog i.p.v. browser confirm
- [x] Superadmin degraderen: AlertDialog i.p.v. browser confirm
- [x] Uitnodiging intrekken: AlertDialog i.p.v. browser confirm
- [x] Gebruikers tabel: "Organisatie(s)" kolom zichtbaar
- [x] Gebruikers tabel: tenant filter dropdown werkt
- [x] Tenant beheer: Bot Personalisatie toggle toegevoegd

#### Bot & opnames (punten 19, 20)
- [x] Bot personalisatie (naam/avatar): geblokkeerd voor niet-Pro tenants (server-side)
- [x] Bot personalisatie: werkt voor Pro/billingExempt tenants
- [x] Bot personalisatie: ook ontgrendeld via admin toggle (botPersonalizationEnabled)
- [ ] Actieve opname toont "Opnemen..." (niet "Verwerken...")
- [ ] Na afloop opname: status wisselt correct naar "Verwerken..."

#### Abonnement (punt 14)
- [x] "Abonnement beheren" knop zichtbaar voor billingExempt tenants

#### Account & beveiliging (punt 18)
- [ ] Account verwijderen: stuurt bevestigingsmail
- [x] Na bevestiging: user wordt gedeactiveerd (isActive=false), niet hard-deleted
- [ ] Gedeactiveerde user kan niet meer inloggen

#### Meetings tabel (punt 10)
- [x] Task items in popover zijn klikbaar → navigeert naar meeting detail

#### Logging pagina (tenant-level)
- [x] Dashboard tiles met iconen (blauw/groen/amber/paars)
- [x] Totaal acties, First Time Right %, Notitie bewerkingen, Categorie wijzigingen

#### Werkruimtes
- [ ] Werkruimte aanmaken/bewerken/verwijderen
- [ ] Documenten uploaden + preview (afbeelding, PDF)
- [ ] Tijdlijn events aanmaken/bewerken/verwijderen
- [x] Tijdlijn: vernieuwen-knop (invalidateQueries)
- [ ] Logo uploaden + bijsnijden
- [x] Floating panel: visueel gealigneerd met app sidebar (sidebar-accent, sidebar-border)

#### In-person opnames (Deepgram)
- [ ] Audio upload → transcriptie werkt
- [ ] Live opname via microfoon werkt
- [ ] Transcript wordt correct verwerkt door pipeline

### Stap 2: Database Backup (VOOR merge)

```bash
# Via Supabase Dashboard:
# Project Settings > Database > Backups > Create backup
# OF via CLI:
pg_dump $PROD_DATABASE_URL > backup_pre_merge_$(date +%Y%m%d_%H%M%S).sql
```

- Noteer exact tijdstip als rollback punt
- Bewaar backup minimaal 7 dagen

### Stap 3: Migraties Reviewen

21 migraties die draaien bij eerste startup na merge:

| # | Migratie | Wat het doet | Risico |
|---|----------|-------------|--------|
| 0010 | password_history | Wachtwoord historie tabel | Laag |
| 0011 | stripe_subscriptions | Stripe subscription tracking | Laag |
| 0012 | recall_integration | Recall.ai bot/recording tabellen | Medium |
| 0013 | billing_exempt | billing_exempt flag op tenants | Laag |
| 0014 | pipeline_queue | Pipeline queue tabel | Laag |
| 0015 | welcome_email_tracking | Email tracking kolom | Laag |
| 0016 | audio_storage | Audio opslag bucket setup | Laag |
| 0017 | token_prefix | Token prefix kolom | Laag |
| 0018 | webhook_dedup | Webhook deduplicatie tabel | Laag |
| 0019 | account_lockout | Account lockout velden | Laag |
| 0020 | performance_indexes | Performance indexes | Laag |
| 0021 | webhook_and_queue_indexes | Extra indexes | Laag |
| 0022 | force_rls_scheduled_recordings | RLS op recordings | Medium |
| 0023 | werkruimtes | Werkruimtes + leden tabellen | Medium |
| 0024 | primary_werkruimte | Primary werkruimte kolom | Laag |
| 0025 | documents | Documenten tabel | Laag |
| 0026 | chat_werkruimte_scope | Chat werkruimte scope | Laag |
| 0027 | document_uploads | Document uploads bucket | Laag |
| 0028 | werkruimte_events | Werkruimte events tabel | Laag |
| 0029 | werkruimte_logo | Werkruimte logo kolom | Laag |
| 0030 | document_date | Document datum kolom + backfill | Laag |

**Let op:** Migraties 0022 en 0023 wijzigen RLS policies en maken nieuwe tabellen met foreign keys. Deze zijn het meest kritisch.

### Stap 4: Merge Uitvoeren

```bash
# 1. PR aanmaken (voor overzicht)
gh pr create --base main --head dev --title "Release: dev → production" --body "..."

# 2. Merge (na goedkeuring)
gh pr merge --merge

# 3. Of handmatig:
git checkout main
git merge dev
git push origin main
```

### Stap 5: Post-Deploy Verificatie

Dezelfde checklist als Stap 1, maar nu op productie:
- [ ] Alle migraties succesvol gedraaid (check Railway logs)
- [ ] Login werkt
- [ ] Dashboard laadt
- [ ] Meetings tonen
- [ ] Kalender 14 dagen
- [ ] Admin logging werkt
- [ ] Geen console errors

### Stap 6: Rollback Plan

```bash
# Optie A: Revert merge commit
git checkout main
git revert HEAD
git push origin main

# Optie B: Hard reset naar pre-merge (DESTRUCTIEF)
git checkout main
git reset --hard <pre-merge-commit-hash>
git push --force origin main

# Optie C: Database restore (als migraties data braken)
# Via Supabase Dashboard: restore backup uit Stap 2
```

**Voorkeur:** Optie A (revert) — veiligst, behoudt geschiedenis.

---

## HUIDIGE SESSIE: Pre-Productie Voorbereiding (17 Feb 2026)

### Overzicht

21 punten voor pre-productie — geanalyseerd, geprioriteerd, en ingedeeld in: **Direct fixen**, **Quick wins**, en **Backlog**.

### Pricing Model — Beslispunt

**Huidige staat:** Per-tenant pricing met user-caps (Starter: 3, Pro: 15, Enterprise: onbeperkt).
**Overwegen:** Per-user pricing (bijv. €X/user/maand) — schaalt beter met adoptie.
**Tiers:**
- **Starter** — automatisering, branding
- **Pro** — werkruimtes, bot aanpassing, inzichten, logging
- **Enterprise** — TBD

**Actie:** Pricing model herzien. Per-user pricing vereist Stripe `per_seat` billing aanpassing. Backlog.

---

### DIRECT FIXEN (bugs + kleine UX issues)

| # | Punt | Probleem | Oplossing | Status |
|---|------|----------|-----------|--------|
| 2 | Categorie beschrijving | "(optioneel)" maar nodig voor AI | Verplicht maken in schema + UI | ✅ |
| 3 | Mappad | Altijd zichtbaar, ook zonder SharePoint | Conditioneel tonen op basis van SharePoint connectie | ✅ |
| 6 | Vertrouwensdrempel | Zichtbaar voor eindgebruiker | Volledig verborgen (interne instelling) | ✅ |
| 7 | Productive sidebar | Altijd zichtbaar | Conditioneel op integratie-status | ✅ |
| 8 | Onboarding navigatie | Start navigeert niet naar eerste stap | Fix: navigeer naar #meeting-bot bij start + auto-start | ✅ |
| 12 | Categorie popup | 3 knoppen overflow in 500px dialog | Responsive layout | ✅ |
| 15 | Platform logging | `fetch()` zonder `getFullUrl()` + `getAuthHeaders()` | Fix API calls in admin.tsx | ✅ |
| 16 | Admin promotie | `window.confirm()` i.p.v. dialoog | Vervang door AlertDialog | ✅ |
| 17 | Admin gebruikers | Geen filter op tenant | Tenant filter dropdown toevoegen | ✅ |
| 20 | Recording status | "Verwerken" i.p.v. "Opnemen" | Webhook mapping: recording_done → recording | ✅ |
| 21 | Kalender bereik | 7 dagen vooruit | Verhoogd naar **3 weken** (504 uur) + chronologische sortering | ✅ |

### QUICK WINS (implementatie ~1-2 bestanden)

| # | Punt | Beschrijving | Status |
|---|------|-------------|--------|
| 9 | Logging dashboard | Dashboard tiles met gekleurde iconen (blauw/groen/amber/paars) | ✅ |
| 10 | Inzichten acties | Badges klikbaar → navigeert naar meeting detail met tab=tasks | ✅ |
| 18 | Account verwijdering | Soft-delete (isActive=false, displayName='[verwijderd]', email=null) | ✅ |
| 19 | Bot personalisatie | Server-side tier check + admin toggle (botPersonalizationEnabled) | ✅ |

### EXTRA FIXES (uit dev testing ronde 1 en 2)

| # | Punt | Beschrijving | Status |
|---|------|-------------|--------|
| - | Cloud opslag melding | "Geen cloud opslag" verborgen als geen SharePoint | ✅ |
| - | Abonnement knop | "Abonnement beheren" ook voor billingExempt tenants | ✅ |
| 5 | AI-generated prompts | "Genereer met AI" knop per categorie + optioneel extra context + best-practice prompt engineering | ✅ |
| - | Tenant bot toggle | botPersonalizationEnabled toggle in admin tenant dialog | ✅ |
| - | Tijdlijn vernieuwen | Vernieuwen-knop met invalidateQueries | ✅ |
| - | Werkruimte panel | Visueel gealigneerd met app sidebar (sidebar-accent, sidebar-border, grotere iconen) | ✅ |

### BACKLOG (niet nu implementeren, documenteren voor later)

| # | Punt | Beschrijving |
|---|------|-------------|
| 1 | Pricing/tiers | Per-user pricing overwegen, Stripe checkout aanpassen |
| 4 | Google Drive/Gmail | Geen bestaande integratie, significante effort |
| 13 | PMS integraties | Monday.com, Asana — parkeren |
| - | Data export | JSON volledigheid checken + Excel export toevoegen |
| - | Operationeel | Logging, monitoring, maintenance voor productie |

---

### Technische Details per Fix

#### Punt 2: Categorie beschrijving verplicht
- **Schema:** `shared/schema.ts` — `insertCategoryInputSchema` description niet verplicht → verplicht maken
- **UI:** Categorie formulier — "(optioneel)" label verwijderen, required marker toevoegen

#### Punt 3: Mappad conditioneel
- **Check:** Of tenant SharePoint connectie heeft (via tenant settings/integratie status)
- **UI:** Mappad veld verbergen als geen SharePoint

#### Punt 6: Vertrouwensdrempel verbergen
- **UI:** Confidence threshold slider verbergen voor normale gebruikers
- **Default:** 70% als server-side default
- **Zichtbaar:** Alleen voor superadmin

#### Punt 7: Productive sidebar conditioneel
- **File:** `app-sidebar.tsx` regel 77-91
- **Check:** `productiveEnabled` of vergelijkbare flag uit tenant settings
- **Actie:** Sidebar item alleen tonen als Productive integratie geconfigureerd

#### Punt 8: Onboarding navigatie
- **File:** `onboarding-guide.tsx` regel 217-245
- **Bug:** `nextStep()` leest `steps[currentStep]` (huidige) maar moet `steps[currentStep + 1]` (volgende) navigeren
- **Fix:** Verwissel volgorde: eerst `setCurrentStep(+1)`, dan navigeer naar nieuwe stap

#### Punt 12: Categorie popup overflow
- **File:** `category-override-dialog.tsx`
- **Probleem:** 3 knoppen in 500px dialoog → overflow
- **Fix:** `flex-wrap` of stack verticaal op kleine schermen

#### Punt 15: Platform logging
- **File:** `admin.tsx` regels 254-278
- **Bug:** `fetch(url)` zonder `getFullUrl()` → stuurt naar verkeerde host op cross-domain
- **Fix:** `getFullUrl()` + `getAuthHeaders()` toevoegen aan alle fetch calls

#### Punt 16: Admin promotie dialoog
- **File:** `admin.tsx` regels 1298, 1335
- **Huidige:** `window.confirm("Weet je zeker...")`
- **Fix:** Vervang door shadcn AlertDialog component

#### Punt 17: Admin tenant filter
- **File:** `admin.tsx` — gebruikerstabel
- **Actie:** Dropdown filter voor tenant naam/ID bovenaan gebruikerslijst

#### Punt 18: Soft-delete
- **File:** `server/supabase-storage.ts` regel 431-443
- **Huidige:** Hard delete (`DELETE FROM users`)
- **Nieuwe:** `UPDATE users SET is_active = false, deactivated_at = now()`
- **Cleanup:** Cron job na 30 dagen

#### Punt 19: Server-side tier check
- **Huidige:** Client-side check of Pro tier
- **Fix:** Server endpoint checkt tenant tier voordat bot personalisatie opgeslagen wordt

#### Punt 20: Recording status debug
- **File:** `scheduled-meetings-view.tsx` regels 631-642
- **Code:** Correct mapping ("recording" → "Opnemen...", processing → "Verwerken...")
- **Vermoeden:** Webhook status update niet correct doorgestuurd
- **Actie:** Live debugging op dev omgeving

#### Punt 21: Kalender bereik
- **File:** `scheduled-meetings-view.tsx` regel 130
- **Huidige:** `?hours=168` (7 dagen)
- **Fix:** `?hours=336` (14 dagen)

---

## HUIDIGE SESSIE: Dev Testing Ronde 2 (17 Feb 2026)

### Feedback verwerkt (5 punten)
1. **AI instructies generator verbeterd**: Optioneel extra context veld, best-practice prompt engineering met system prompt (gestructureerde output, voorbeelden, specificiteit), `extraContext` parameter
2. **Logging dashboard tiles**: Gekleurde iconen toegevoegd (blauw Activity, groen CheckCircle, amber PenLine, paars FolderSync)
3. **Tijdlijn vernieuwen-knop**: Werkt nu via `invalidateQueries` + documents cache busting
4. **Werkruimte floating panel**: Visueel gealigneerd met app sidebar (sidebar-border, sidebar-accent, grotere iconen h-4, grotere touch targets py-2, "Standaard" badge tekst, breedte w-60)

### Commits
- `137b032` — Fix 9 user testing items (kalender, AI generator, tenant toggles, etc.)
- `e924746` — Improve AI prompt generator, logging tiles, timeline refresh, werkruimte panel

### Bestanden gewijzigd (ronde 1 + 2)
- `client/src/components/scheduled-meetings-view.tsx` — Kalender 3 weken + chronologische sortering
- `client/src/components/billing-section.tsx` — billingExempt check voor "Abonnement beheren"
- `client/src/components/onboarding/onboarding-guide.tsx` — Navigeer naar eerste stap bij start
- `client/src/components/werkruimte-timeline.tsx` — Vernieuwen-knop met invalidateQueries
- `client/src/components/werkruimte-panel.tsx` — Sidebar-aligned design (sidebar-accent, sidebar-border)
- `client/src/components/app-sidebar.tsx` — Werkruimte popover breedte w-60
- `client/src/pages/admin.tsx` — Bot Personalisatie toggle in tenant dialog
- `client/src/pages/logging.tsx` — Dashboard tiles met gekleurde iconen
- `client/src/pages/settings.tsx` — Zekerheidsdrempel verwijderd, cloud opslag melding verborgen, AI prompt generator met extra context, botPersonalizationEnabled gating
- `server/routes/config.ts` — POST `/api/category-prompts/:categoryId/generate` endpoint met best-practice prompt engineering
- `shared/schema.ts` — `botPersonalizationEnabled` in systemSettingsSchema + defaults

---

## VORIGE SESSIE: Pre-productie Audit Fixes (17 Feb 2026)

### Voltooide fixes
- **H-8**: DialogDescription toegevoegd aan alle 19 bestanden met Dialog
- **M-2/M-3**: Engels→Nederlands vertaling (Owner→Eigenaar, Quick View→Snel bekijken, etc.)
- **M-6**: Aria-labels op 7 wachtwoord toggles
- **ManualBotScheduler**: Toegevoegd aan Kalender pagina

### Gecommit als: 94b87c3 op `dev` branch

---

## VORIGE SESSIE: Document Editing, Preview & Timeline Fixes (16 Feb 2026)

### Context
Werkruimte documenten en tijdlijn waren functioneel, maar drie zaken misten:
1. **Event bewerken werkte niet**: Edit knoppen op tijdlijn events deden niets (hidden met opacity-0)
2. **Document bewerken**: Datum en titel van documenten waren niet te wijzigen
3. **Document preview**: Geen inline preview voor afbeeldingen, PDF en HTML

### Fase 0: Fix Event Bewerken op Tijdlijn
- **`werkruimte-timeline.tsx`**: Edit/delete knoppen altijd zichtbaar gemaakt (was `opacity-0 group-hover:opacity-100`), event items klikbaar gemaakt voor edit dialog

### Fase 1: Document Datum Veld
- **Migration** `migrations/0030_document_date.sql`: `document_date` kolom op `documents` tabel, backfill vanuit `created_at`, NOT NULL + DEFAULT now()
- **Schema** `shared/schema.ts`: `documentDate` veld + `updateDocumentSchema` uitgebreid
- **Storage** `server/supabase-storage.ts`: `transformDocumentFromDb/ToDb` + timeline query met `COALESCE(d.document_date, d.created_at)`
- **MemStorage** `server/storage.ts`: `documentDate` in create/update stubs

### Fase 2: Document Edit UI
- **`werkruimte-documents.tsx`**: Edit dialoog per document (titel, datum, type, status) + edit knop per rij. PATCH naar `/api/werkruimtes/:werkruimteId/documents/:id`. Invalideert zowel documents als timeline query cache.
- **`werkruimte-timeline.tsx`**: Document edit dialoog (titel, datum) + edit icoon per document entry op tijdlijn.

### Fase 3: Document Preview
- **`werkruimte-documents.tsx`**: Preview dialoog met `<img>` voor afbeeldingen, `<iframe>` voor PDF/HTML. Klikken op previewable document opent preview. Eye-icoon + download knop. Niet-previewable types (Word/Excel/PPT) worden direct gedownload.
- **`werkruimte-timeline.tsx`**: Zelfde preview functionaliteit op document entries in tijdlijn. Klikken op document opent preview of download.
- Hergebruikt bestaand download endpoint voor signed URLs — geen nieuw endpoint nodig.

### Bug Fixes meegenomen
- **`werkruimte-documents.tsx`**: `handleDownload` miste `getFullUrl()` + `getAuthHeaders()` (werkte niet op Railway cross-origin)
- **`werkruimte-documents.tsx`**: Datum toont nu `documentDate` i.p.v. `updatedAt`

### Bestanden Gewijzigd
| Bestand | Actie |
|---------|-------|
| `migrations/0030_document_date.sql` | **Nieuw** — document_date kolom |
| `shared/schema.ts` | Edit — documentDate veld + update schema |
| `server/supabase-storage.ts` | Edit — transforms + timeline query |
| `server/storage.ts` | Edit — MemStorage documentDate stubs |
| `client/src/components/werkruimte-documents.tsx` | Edit — edit dialoog + preview + getFullUrl fix |
| `client/src/components/werkruimte-timeline.tsx` | Edit — event fix + document edit/preview |

---

## VORIGE SESSIE: In-Person Meeting Recording met Deepgram (16 Feb 2026)

### Context
Gebruikers willen fysieke meetings opnemen rechtstreeks vanuit de web-app, met transcriptie en speaker herkenning. Recall.ai biedt geen server-side audio upload API — alleen bots die video calls joinen. Deepgram Nova-3 is gekozen vanwege: beste Nederlandse taalondersteuning, excellente speaker diarization, NL↔EN code-switching, en directe browser WebSocket streaming.

### Architectuur Overzicht

```
┌─────────────────────────────────────────────────────────────┐
│  PAD 1: Live Opname (browser)                               │
│                                                             │
│  getUserMedia() → MediaRecorder → WebSocket proxy endpoint  │
│       → Deepgram Nova-3 (NL, diarize=true)                 │
│       → Real-time transcript op scherm                      │
│       → Bij stop: audio blob opslaan + meeting aanmaken     │
├─────────────────────────────────────────────────────────────┤
│  PAD 2: Audio Upload (na afloop)                            │
│                                                             │
│  Upload MP3/WAV/M4A → Supabase storage                     │
│       → Deepgram batch API (NL, diarize=true)               │
│       → Transcript ontvangen → meeting aanmaken             │
├─────────────────────────────────────────────────────────────┤
│  GEMEENSCHAPPELIJK (beide paden)                            │
│                                                             │
│  Meeting aanmaken met transcript + audio                    │
│       → Pipeline: determine_owner → categorize →            │
│         generate_notes → generate_documents →               │
│         upload_sharepoint → create_draft → create_tasks     │
└─────────────────────────────────────────────────────────────┘
```

### Security Uitgangspunten (KRITISCH)

| Laag | Vereiste | Implementatie |
|------|----------|---------------|
| **Deepgram API key** | NOOIT naar client sturen | Server-side proxy — browser verbindt met onze server, server verbindt met Deepgram |
| **Tenant isolatie** | Audio per tenant gescheiden | Pad: `{tenantId}/{meetingId}.{ext}` in private `meeting-audio` bucket |
| **RLS** | Database queries tenant-scoped | Alle storage via `withTenant(tenantId, ...)` |
| **Route auth** | Alle endpoints beveiligd | `requireAuth` + `requireTenant` op alle nieuwe endpoints |
| **Audio uploads** | MIME whitelist + size limit | Multer: audio/mpeg, audio/wav, audio/webm, audio/ogg, audio/mp4, audio/x-m4a. Max 500MB |
| **WebSocket auth** | Session verificatie bij connect | Cookie-based session check bij WebSocket upgrade |
| **Deepgram key opslag** | Encrypted at rest | Via bestaand `DEEPGRAM_API_KEY` env var (zelfde patroon als andere API keys) |
| **Input validatie** | Zod schemas op alle inputs | Metadata (titel, datum, deelnemers) via Zod `.parse()` |
| **Tier limits** | Meeting limieten respecteren | `checkMeetingLimit()` check vóór meeting creatie |
| **Audit logging** | Opnames loggen | Security events bij start/stop opname en audio upload |

---

### Fase 1: Deepgram Service (`server/services/deepgram.ts`)

**Nieuw bestand** — Deepgram API wrapper met twee modi:

```typescript
interface TranscriptionResult {
  transcript: string;              // "Speaker 1: tekst\n\nSpeaker 2: tekst" formaat
  segments: TranscriptSegment[];   // Raw segments met timestamps
  participants: Participant[];      // Unieke sprekers
  durationSeconds: number;
  language: string;
  confidence: number;
}

interface TranscriptSegment {
  speakerName: string;    // "Spreker 1", "Spreker 2", etc.
  text: string;
  startTime: number;      // seconden
  endTime: number;
}
```

**Batch methode:** `transcribeAudio(audioBuffer, mimeType, options)` → Deepgram pre-recorded API
- Model: `nova-3`
- Language: `nl` (met `detect_language: true` als fallback)
- Diarization: `diarize: true`
- Smart formatting: `smart_format: true`, `punctuate: true`
- Output: `TranscriptionResult`

**Streaming methode:** `createLiveTranscriptionStream(options)` → Deepgram WebSocket
- Retourneert WebSocket URL + auth headers voor server-side proxy
- Zelfde parameters als batch

**Beveiligingsmaatregelen in service:**
- API key alleen server-side (uit `process.env.DEEPGRAM_API_KEY`)
- Retry logica met exponential backoff
- Timeout: 5 minuten voor batch, geen timeout voor streaming
- Error handling: duidelijke foutmeldingen zonder API key leakage

---

### Fase 2: Audio Upload Endpoint (`server/routes/recordings.ts`)

**Nieuw endpoint:** `POST /api/recordings/transcribe-audio`

```
Security stack:
  requireAuth → requireTenant → checkMeetingLimit → multer(audioUpload) → handler
```

**Flow:**
1. Ontvang audio file + metadata (titel, datum, deelnemers optioneel)
2. `checkMeetingLimit(tenantId)` — respecteer tier limieten
3. Upload audio naar Supabase: `supabaseFileStorage.uploadAudio(tenantId, meetingId, buffer, mimeType)`
4. Verstuur naar Deepgram batch API: `deepgramService.transcribeAudio(buffer, mimeType)`
5. Format transcript: zelfde `"Speaker: tekst"` formaat als Recall
6. `storage.createMeeting()` met transcript, audio path, participants
7. Initialiseer pipeline steps (markeer `ingest` + `fetch_transcript` als completed)
8. `pipelineQueue.enqueue(meetingId)`
9. Return meeting ID + transcript preview

**Multer configuratie:**
```typescript
const audioUpload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 500 * 1024 * 1024 },  // 500MB
  fileFilter: (req, file, cb) => {
    const allowed = ['audio/mpeg', 'audio/mp3', 'audio/wav', 'audio/webm', 'audio/ogg', 'audio/mp4', 'audio/x-m4a'];
    cb(null, allowed.includes(file.mimetype));
  }
});
```

**Zod schema:**
```typescript
const transcribeAudioSchema = z.object({
  title: z.string().min(1).max(200),
  date: z.string().datetime(),
  participants: z.array(z.object({
    name: z.string(),
    email: z.string().email().optional(),
    isExternal: z.boolean().optional()
  })).optional(),
  primaryWerkruimteId: z.string().optional(),
});
```

---

### Fase 3: Live Recording — Server-side WebSocket Proxy

**Nieuw endpoint:** `GET /api/recordings/live-transcription` (WebSocket upgrade)

**Waarom server-side proxy?**
- Deepgram API key mag NOOIT naar de browser
- Session-based auth verificatie bij WebSocket handshake
- Tenant context beschikbaar voor audit logging
- Rate limiting per tenant mogelijk

**Flow:**
```
Browser                    Server                     Deepgram
  |                          |                           |
  |-- WS upgrade ----------->|                           |
  |   (cookie auth)          |-- verify session -------->|
  |                          |-- WS connect ------------>|
  |                          |   (API key, nova-3, nl)   |
  |                          |                           |
  |-- audio chunks --------->|-- forward audio --------->|
  |                          |                           |
  |<-- transcript events ----|<-- transcript events -----|
  |                          |                           |
  |-- "stop" message ------->|                           |
  |                          |-- close WS -------------->|
  |                          |                           |
  |-- POST audio blob ------>|                           |
  |   (final save)           |-- store in Supabase       |
  |                          |-- create meeting          |
  |                          |-- enqueue pipeline        |
  |<-- meeting created ------|                           |
```

**WebSocket auth:**
```typescript
// Bij upgrade: parse session cookie, verify user + tenant
wss.on('connection', async (ws, req) => {
  const session = await parseSessionFromRequest(req);
  if (!session?.user?.activeTenantId) {
    ws.close(4001, 'Niet geauthenticeerd');
    return;
  }
  // ... start Deepgram proxy
});
```

**Audio opslag bij stop:**
- Browser stuurt finalized audio blob via regulier POST endpoint
- Server slaat op in Supabase `meeting-audio` bucket
- Meeting wordt aangemaakt met het volledige transcript

---

### Fase 4: Frontend — Opname Dialoog (`client/src/components/recording-dialog.tsx`)

**Nieuw component** — Dialoog met twee tabs:

**Tab 1: "Opnemen" (Live)**
- Grote rode opname-knop (pulserende animatie)
- Microfoon selectie dropdown (`navigator.mediaDevices.enumerateDevices()`)
- Timer display (opname duur)
- Real-time transcript met speaker labels
- Stop knop → bevestigingsdialoog met:
  - Titel invoer (verplicht)
  - Optioneel: werkruimte selectie
  - "Opslaan & Verwerken" knop

**Tab 2: "Upload" (Bestand)**
- Drag & drop zone voor audiobestanden
- Ondersteunde formaten: MP3, WAV, M4A, WebM, OGG
- Max 500MB indicator
- Na upload: titel invoer + werkruimte selectie
- "Transcriberen & Verwerken" knop
- Progress indicator (upload → transcriptie → verwerking)

**Waar te plaatsen:**
- Kalender pagina: "+ Opname" knop naast bestaande UI
- Vergaderingen pagina: "+ Opname" knop
- Werkruimte detail: in vergaderingen tab

---

### Fase 5: Pipeline Integratie

**Meeting creatie (zelfde patroon als Recall webhook):**
```typescript
const meeting = await storage.createMeeting({
  tenantId,
  title: metadata.title,
  date: new Date(metadata.date),
  duration: result.durationSeconds,
  participants: result.participants,
  transcript: result.transcript,
  status: "pending",
  audioStoragePath: audioPath,
  audioStatus: "stored",
  audioDurationSeconds: result.durationSeconds,
  primaryWerkruimteId: metadata.primaryWerkruimteId || null,
  owner: userId,  // Opnemer is eigenaar
}, tenantId);
```

**Pipeline steps (markeer eerste 2 als klaar):**
```typescript
const steps = PIPELINE_STAGES.map(stage => ({
  meetingId: meeting.id,
  stage,
  status: ['ingest', 'fetch_transcript'].includes(stage) ? 'completed' : 'pending',
  completedAt: ['ingest', 'fetch_transcript'].includes(stage) ? new Date() : null,
}));
await storage.createPipelineSteps(steps, tenantId);
await pipelineQueue.enqueue(meeting.id);
```

---

### Bestanden Overzicht

| # | Bestand | Actie | Beschrijving |
|---|---------|-------|-------------|
| 1 | `server/services/deepgram.ts` | **NIEUW** | Deepgram API wrapper (batch + streaming) |
| 2 | `server/routes/recordings.ts` | Edit | Audio upload + live transcription endpoints |
| 3 | `server/routes/index.ts` | Edit | WebSocket setup voor live transcription |
| 4 | `shared/schema.ts` | Edit | Zod schema voor audio transcription input |
| 5 | `client/src/components/recording-dialog.tsx` | **NIEUW** | Opname/upload dialoog component |
| 6 | `client/src/pages/kalender.tsx` | Edit | "+ Opname" knop toevoegen |
| 7 | `client/src/pages/meetings.tsx` | Edit | "+ Opname" knop toevoegen |

### Test Plan

**Functioneel:**
- [ ] Audio upload (MP3) → transcriptie → meeting aangemaakt met transcript
- [ ] Audio upload (WAV) → zelfde flow
- [ ] Audio upload (M4A) → zelfde flow
- [ ] Transcript bevat speaker labels ("Spreker 1:", "Spreker 2:", etc.)
- [ ] Meeting verschijnt in vergaderingen lijst na verwerking
- [ ] Pipeline verwerkt meeting (notes, categorisatie, etc.)
- [ ] Audio is afspeelbaar vanuit meeting detail pagina
- [ ] Live opname: microfoon audio wordt gestreamd
- [ ] Live opname: real-time transcript verschijnt tijdens opname
- [ ] Live opname: bij stop → meeting wordt aangemaakt
- [ ] Werkruimte toewijzing werkt bij opname
- [ ] Tier limieten worden gerespecteerd

**Security:**
- [ ] Deepgram API key is NOOIT zichtbaar in browser (network tab, JS bundles)
- [ ] WebSocket verbinding vereist geldige sessie
- [ ] Audio upload vereist auth + tenant
- [ ] Audio bestanden zijn tenant-gescheiden in Supabase (`{tenantId}/...`)
- [ ] Ongeldig MIME type wordt geweigerd (geen executables, geen video)
- [ ] Te groot bestand (>500MB) wordt geweigerd
- [ ] Meeting creation respecteert tenant RLS
- [ ] Andere tenant kan niet bij audio van deze tenant
- [ ] Zod validatie op alle request bodies
- [ ] Rate limiting op transcription endpoints
- [ ] Audit log bij audio upload en opname start/stop

**Edge cases:**
- [ ] Lege audio file → duidelijke foutmelding
- [ ] Deepgram API error → foutmelding, audio bewaard voor retry
- [ ] Browser mic permissie geweigerd → duidelijke instructie
- [ ] WebSocket verbinding verbroken → graceful recovery
- [ ] Zeer lang bestand (>2 uur) → werkt correct
- [ ] Geen DEEPGRAM_API_KEY geconfigureerd → feature verborgen of duidelijke fout

---

## HUIDIGE SESSIE: Werkruimte Hub Feedback & Verbeteringen (16 Feb 2026)

### Context
Na implementatie van de Werkruimte Intelligence Hub (Fasen A–E, commit `b40404a` op `dev`) zijn bij het testen 5 verbeterpunten gevonden. Daarnaast: werkruimte CRUD uit settings verplaatst naar werkruimte-pagina's, logo upload met crop/zoom, en destructieve delete bevestiging.

### Fase 1: Layout Fix
- **`werkruimte-detail.tsx`**: Container class `max-w-6xl mx-auto` → `flex-1 overflow-auto` + `p-6 space-y-6` (consistent met andere pagina's)

### Fase 2: Meeting Filters in Werkruimte Tab
- **`werkruimte-detail.tsx`**: Vergaderingen tab heeft nu search, status, category, en folder filters (gekopieerd uit `meetings.tsx` patroon)

### Fase 3: Document Uploads
- **Migration** `migrations/0027_document_uploads.sql`: Kolommen `file_url`, `file_name`, `file_size`, `file_type`, `is_file_upload` op `documents` tabel
- **Schema** `shared/schema.ts`: Document upload velden + `fileName`, `fileSize`, `fileType` in `activityContextSchema`
- **Storage** `server/supabase-storage.ts`: `createDocument` uitgebreid met file-velden
- **File Storage** `server/services/supabase-file-storage.ts`: `uploadDocument()` en `getDocumentSignedUrl()` methoden — **private bucket** (`werkruimte-documents`), signed URLs (1 uur), tenant-scoped paden
- **Routes** `server/routes/documents.ts`: `POST /api/werkruimtes/:werkruimteId/documents/upload` (Multer + MIME whitelist, geen SVG), `GET /api/werkruimtes/:werkruimteId/documents/:id/download` (signed URL)
- **Frontend** `client/src/components/werkruimte-documents.tsx`: Upload UI met drag & drop, type-iconen (Word/PDF/Excel/PPT/afbeelding), download via signed URL

### Fase 4: Tijdlijn Uitbreidingen
- **Migration** `migrations/0028_werkruimte_events.sql`: `werkruimte_events` tabel met RLS + FORCE + tenant_isolation policy. Event types: telefoontje, email, notitie, overig. Linked documents en meetings.
- **Schema** `shared/schema.ts`: `werkruimteEvents` tabel definitie + types
- **Storage** `server/supabase-storage.ts`: CRUD voor werkruimte_events (alle via `withTenant()`), `getWerkruimteTimeline` UNION ALL uitgebreid: meetings + documents + **events**
- **Routes** `server/routes/werkruimtes.ts`: CRUD endpoints `POST/PATCH/DELETE /api/werkruimtes/:id/events/:eventId` met `requireWerkruimteMember`
- **Frontend** `client/src/components/werkruimte-timeline.tsx`: "Event toevoegen" dialoog, edit/delete op handmatige events, type-iconen, klikbare meeting/document links

### Fase 5: Meeting Navigatie met Werkruimte Context
- **`meetings-table.tsx`**: `meetingUrlBuilder` prop voor custom meeting URLs
- **`werkruimte-detail.tsx`**: Vergaderingen tab passeert werkruimte context query params
- **`werkruimte-timeline.tsx`**: `werkruimteName` prop, meeting links met werkruimte context
- **`meeting-detail.tsx`**: Breadcrumb "Werkruimtes > {naam} > {meeting}" als `from=werkruimte`, terug-pijl naar werkruimte
- **`app-sidebar.tsx`**: Detecteert `from=werkruimte` query param → highlight Werkruimtes i.p.v. Vergaderingen

### Werkruimte Logo Upload
- **Migration** `migrations/0029_werkruimte_logo.sql`: `logo_url TEXT` kolom op `werkruimtes`
- **Schema** `shared/schema.ts`: `logoUrl` op werkruimtes tabel
- **File Storage** `server/services/supabase-file-storage.ts`: `uploadWerkruimteLogo()` / `deleteWerkruimteLogo()` — public `branding` bucket, pad `werkruimte-logos/{tenantId}/{werkruimteId}/`
- **Routes** `server/routes/werkruimtes.ts`: `POST/DELETE /api/werkruimtes/:id/logo` met `requireTenantAdmin` + Multer MIME whitelist (PNG/JPEG/GIF/WebP, geen SVG)
- **Frontend** `client/src/components/werkruimte-logo-upload.tsx` (NIEUW): `react-easy-crop` met zoom slider, canvas crop naar 256×256 PNG, client-side validatie (type + 5MB), hover overlay met camera-icoon, delete knop
- **`werkruimte-detail.tsx`**: Header gebruikt `WerkruimteLogoUpload` component
- **`werkruimtes.tsx`** + **`werkruimte-panel.tsx`**: Tonen logo's in kaarten en sidebar panel

### Werkruimte CRUD Beheer (uit Settings verplaatst)
- **`werkruimte-dialog.tsx`** (NIEUW): Herbruikbaar create/edit dialoog (react-hook-form + Zod). Velden: naam, type, beschrijving, kleurpicker. Gebruikt in 3 plekken.
- **`werkruimtes.tsx`**: Herschreven — "+ Nieuwe werkruimte" knop met inline dialoog, navigeert na creatie naar detail
- **`werkruimte-panel.tsx`**: Herschreven — inline create dialoog i.p.v. settings link
- **`werkruimte-detail.tsx`**: Settings tandwiel dropdown (DropdownMenu) met "Bewerken" en "Verwijderen"
  - **Bewerken**: Opent `WerkruimteDialog` in edit-modus
  - **Verwijderen**: Destructieve `AlertDialog` met rode accenten, waarschuwingsicoon, gedetailleerde opsomming (documenten, tijdlijn, chats permanent verwijderd; vergaderingen losgekoppeld)
- **`settings.tsx`**: Werkruimtes sectie verwijderd
- **`app-sidebar.tsx`**: "Werkruimtes" verwijderd uit settingsSubItems

### Nieuwe API Endpoints

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/werkruimtes/:werkruimteId/documents/upload` | POST | Member | Upload document (Multer + signed URL) |
| `/api/werkruimtes/:werkruimteId/documents/:id/download` | GET | Member | Download document (signed URL) |
| `/api/werkruimtes/:id/events` | POST | Member | Create tijdlijn event |
| `/api/werkruimtes/:id/events/:eventId` | PATCH | Member | Update tijdlijn event |
| `/api/werkruimtes/:id/events/:eventId` | DELETE | Member | Delete tijdlijn event |
| `/api/werkruimtes/:id/logo` | POST | Admin | Upload werkruimte logo |
| `/api/werkruimtes/:id/logo` | DELETE | Admin | Delete werkruimte logo |

### Nieuwe Bestanden

| Bestand | Beschrijving |
|---------|-------------|
| `migrations/0027_document_uploads.sql` | Document upload kolommen |
| `migrations/0028_werkruimte_events.sql` | Tijdlijn events tabel + RLS |
| `migrations/0029_werkruimte_logo.sql` | Logo URL kolom |
| `client/src/components/werkruimte-dialog.tsx` | Herbruikbaar create/edit dialoog |
| `client/src/components/werkruimte-logo-upload.tsx` | Logo upload met crop/zoom |

### Security
- `werkruimte_events`: RLS enabled + FORCE + tenant_isolation policy
- Document bucket: **private** (niet public), signed URLs (1 uur expiratie)
- Logo endpoints: MIME whitelist (geen SVG), max file size, `requireTenantAdmin`
- Alle nieuwe storage-methoden via `withTenant()`
- Event create: `tenant_id` en `created_by` server-side gezet
- Zod validatie op alle request bodies

### Bug Fixes
- `getWerkruimte(tenantId, werkruimteId)` → `getWerkruimte(werkruimteId, tenantId)` in logo endpoints (parameter volgorde was omgedraaid)
- `getActiveTenantId(res)` → `getActiveTenantId(req)` in logo endpoints
- `logoUrl: werkruimte.logoUrl ?? null` in MemStorage (undefined vs null)
- Timeline `entry.metadata?.description` type guard (`typeof === "string"`)
- Migration 0028: `linked_document_id` UUID → VARCHAR (documents.id is varchar)

---

## VORIGE SESSIE: Werkruimte als Intelligence Hub (15 Feb 2026)

### Context
Werkruimtes evolved from simple access-control containers into intelligence hubs with documents, timeline, scoped AI agent, and cross-werkruimte search capabilities. See `docs/DESIGN-werkruimte-hub.md` for the full design document.

### What was built (Phases A–E)

#### Fase A: Documents (database + storage + API + frontend)
- **SQL migration** `migrations/0025_documents.sql`: documents table with RLS, indexes, document_type_enum, document_status_enum
- **Schema** `shared/schema.ts`: documentTypeEnum (note/proposal/plan/prd/report/research), documentStatusEnum (draft/review/final/archived), documents pgTable, InsertDocument, UpdateDocumentInput, Document types. Added documentId/title/werkruimteId/changes to activityContextSchema.
- **Storage** `server/storage.ts` + `server/supabase-storage.ts`: Full CRUD — getDocuments, getDocument, createDocument, updateDocument, deleteDocument, getDocumentsForWerkruimte. Both SQL (withTenant RLS) and Supabase REST paths. Transform helpers: transformDocumentFromDb/ToDb.
- **Routes** `server/routes/documents.ts` (NEW): CRUD nested under `/api/werkruimtes/:werkruimteId/documents`. `requireWerkruimteMember` middleware (checks membership, admins bypass). Activity logging on create/update/delete.
- **Frontend** `client/src/components/werkruimte-documents.tsx` (NEW): Document list with type icons/status badges, create dialog, inline document editor with title/content/status, delete confirmation.

#### Fase B: Timeline
- **Storage** `server/supabase-storage.ts`: `getWerkruimteTimeline` using UNION ALL of meetings + documents, sorted by date DESC. Both SQL and REST API paths.
- **Route** `server/routes/werkruimtes.ts`: `GET /api/werkruimtes/:id/timeline` endpoint.
- **Frontend** `client/src/components/werkruimte-timeline.tsx` (NEW): Groups entries by month/year, shows meeting and document entries with icons/status/dates, meeting entries link to detail.

#### Fase C: Werkruimte-scoped Agent Chat
- **SQL migration** `migrations/0026_chat_werkruimte_scope.sql`: Added werkruimte_id to chat_threads (nullable FK, CASCADE delete). NULL = tenant-wide, NOT NULL = werkruimte-scoped.
- **Agent** `server/services/agent.ts`: AgentContext extended with werkruimteId/werkruimteName. `buildAgentSystemPrompt` injects `<werkruimte_scope>` section with strict isolation rules when scoped.
- **Chat routes** `server/routes/chat.ts`: Thread creation accepts werkruimteId. Thread listing filters by werkruimteId query param. Message sending resolves werkruimte name and passes to agent context.
- **Frontend** `client/src/components/werkruimte-chat.tsx` (NEW): Compact chat component scoped to werkruimte. Thread management (create/select/delete), SSE streaming.
- **Inzichten page** `client/src/pages/inzichten.tsx`: Filters for tenant-wide only threads (`werkruimteId=null`).

#### Fase D: Agent → Document Creation
- **Capability** `server/capabilities/createWerkruimteDocument.ts` (NEW): `create_werkruimte_document` agentic capability. Takes werkruimteId, title, content, type. Creates document as draft with aiGenerated=true. Requires approval (human-in-the-loop).
- **Registry** `server/capabilities/index.ts`: Registered in mutationCapabilities.

#### Fase E: Tenant-wide Agent Enhancement
- **Capability** `server/capabilities/searchDocuments.ts` (NEW): `search_documents` agentic capability. Searches documents across accessible werkruimtes with fuzzy matching. Respects werkruimte membership for non-admins.
- **Registry** `server/capabilities/index.ts`: Registered in queryCapabilities.
- **System prompt** `server/services/agent.ts`: Added document search guidance, cross-werkruimte comparison strategy (Step 6), workflow advice for combining search_meetings + search_documents.

#### Werkruimte Detail Page Rewrite
- **`client/src/pages/werkruimte-detail.tsx`** (REWRITTEN): Now uses Tabs with 5 tabs: Vergaderingen, Documenten, Inzichten, Tijdlijn, Leden. Each tab lazy-loads its content component.

### Files Overview

| New Files | Description |
|-----------|-------------|
| `migrations/0025_documents.sql` | Documents table + RLS |
| `migrations/0026_chat_werkruimte_scope.sql` | werkruimte_id on chat_threads |
| `server/routes/documents.ts` | Document CRUD API + requireWerkruimteMember |
| `server/capabilities/createWerkruimteDocument.ts` | Agent document creation capability |
| `server/capabilities/searchDocuments.ts` | Agent document search capability |
| `client/src/components/werkruimte-documents.tsx` | Documents list + editor |
| `client/src/components/werkruimte-timeline.tsx` | Timeline component |
| `client/src/components/werkruimte-chat.tsx` | Scoped agent chat |

| Modified Files | Changes |
|----------------|---------|
| `shared/schema.ts` | Document types/tables, activityContextSchema fields |
| `server/storage.ts` | IStorage: document + timeline methods, MemStorage stubs |
| `server/supabase-storage.ts` | Document CRUD, timeline query, chat thread werkruimte scope |
| `server/services/agent.ts` | AgentContext.werkruimteId/Name, scoped system prompt, enhanced workflow guidance |
| `server/capabilities/index.ts` | Registry: searchDocuments, createWerkruimteDocument |
| `server/routes/chat.ts` | werkruimte-scoped thread creation/listing/messaging |
| `server/routes/werkruimtes.ts` | Timeline endpoint |
| `server/routes/index.ts` | Register documents router |
| `client/src/pages/werkruimte-detail.tsx` | Tabs layout rewrite |
| `client/src/pages/inzichten.tsx` | Filter for tenant-wide only threads |

---

## VORIGE SESSIE: Primary Werkruimte — clientName → Werkruimte (15 Feb 2026)

### Context
Meetings have a free-text `clientName` field for the "Klant/relatie" concept. Now that werkruimtes exist, we're replacing this with a structured `primary_werkruimte_id` foreign key. This gives meetings a proper werkruimte reference instead of free text, enabling better filtering, consistency, and integration with the access control system.

### Design Decisions
- **`primary_werkruimte_id`** on meetings table: single FK replacing `clientName` (one primary werkruimte per meeting)
- Setting a primary werkruimte clears ALL existing `meeting_werkruimtes` entries, then adds only the new primary (1:1 relationship enforced)
- `clientName` field kept temporarily for backward compat until manual migration
- **Werkruimte Popover** (Radix): sidebar button opens a Popover to the right, not a separate panel. Radix handles click-outside, toggle, z-index via Portal
- **SidebarMenuButton forwardRef**: The shadcn/ui `SidebarMenuButton` component was missing `React.forwardRef`, breaking Radix `asChild` composition (Popover, Tooltip triggers silently failed). Fixed by adding forwardRef to the component.
- **Werkruimte detail page**: shows members with searchable combobox for adding (admin only), plus meetings list
- **People API ID mapping**: `/api/people` returns `id` (userTenant junction PK) and `userId` (actual user ID). When adding werkruimte members, always use `userId` — the backend leden endpoint expects user IDs, not userTenant IDs.
- **Quick-create werkruimte** from meeting detail combobox (admin only)
- Migration of existing clientName data → done manually with user later
- **Windows local dev**: server binds to `127.0.0.1` instead of `0.0.0.0`, skips `reusePort`

### Phase 1: Database + Backend (In Progress)

**Migration: `migrations/0024_primary_werkruimte.sql`**
- ALTER TABLE meetings ADD COLUMN primary_werkruimte_id VARCHAR REFERENCES werkruimtes(id) ON DELETE SET NULL
- Index on primary_werkruimte_id WHERE NOT NULL

**Schema: `shared/schema.ts`**
- Added `primaryWerkruimteId` to meetings table definition
- `clientName` marked as DEPRECATED (kept for migration)

**Storage: `server/supabase-storage.ts`** ✅
- LEFT JOIN werkruimtes in getMeetingsLight for enriched data (primaryWerkruimteName, primaryWerkruimteColor)
- DB↔JS mapping for primaryWerkruimteId in transformMeetingFromDb/LightFromDb/ToDb
- New helper: setPrimaryWerkruimte (updates meeting + upserts into meeting_werkruimtes)
- getMeetingsForWerkruimte also includes LEFT JOIN for primary werkruimte enrichment

**Routes: `server/routes/meetings.ts`** ✅
- PATCH `/api/meetings/:id/details` accepts primaryWerkruimteId
- Calls storage.setPrimaryWerkruimte for werkruimte changes
- Activity logging with action "assign_meeting_werkruimte"

### Phase 2: Frontend UI (Complete ✅)

**Werkruimte Panel: `client/src/components/werkruimte-panel.tsx`**
- Exports `WerkruimtePanelContent` component (pure content, no wrapper/backdrop)
- Used inside Radix Popover in sidebar — Radix handles all event management
- Search bar filters werkruimtes by name (shown when 7+ werkruimtes)
- Color dots for each werkruimte, "Alle werkruimtes" link, admin-only "+ Nieuwe werkruimte"
- `onSelect` callback closes the popover when a werkruimte is chosen

**Layout: `client/src/App.tsx`**
- Clean layout: SidebarProvider → AppSidebar + header + main content
- No werkruimte panel state management (moved to sidebar via Radix Popover)

**Sidebar: `client/src/components/app-sidebar.tsx`**
- "Werkruimtes" nav item wrapped in Radix `Popover` + `PopoverTrigger` + `PopoverContent`
- Uses standard `SidebarMenuButton` as trigger (requires forwardRef — see `sidebar.tsx` fix below)
- Popover opens to the right (`side="right"`, `sideOffset={8}`) with werkruimte list content
- Active state: highlighted when popover is open OR on /werkruimtes route

**SidebarMenuButton fix: `client/src/components/ui/sidebar.tsx`**
- Converted `SidebarMenuButton` from plain function to `React.forwardRef<HTMLButtonElement, ...>`
- Passes `ref` through to inner `<Comp>` element (native `<button>` or Radix `Slot`)
- Required for React 18 compatibility with Radix `asChild` patterns (PopoverTrigger, TooltipTrigger, etc.)
- Added `displayName = "SidebarMenuButton"` for DevTools

**Werkruimte Detail: `client/src/pages/werkruimte-detail.tsx`**
- Member management: list current members with avatars, remove button (admin only)
- Add members via searchable Command combobox in a Popover (search by name/email/username)
- Uses `/api/people` for available users (maps `userId` field, NOT `id`), `/api/werkruimtes/:id/leden` for add/remove
- Filters out already-added members and pending invitations from the combobox
- Stats cards: member count + meeting count
- Meetings list via `MeetingsTable` component

**Meeting Detail: `client/src/pages/meeting-detail.tsx`**
- Searchable Popover + Command combobox replaces free-text clientName edit
- Shows current werkruimte with color dot, or "Niet ingesteld" placeholder
- "Verwijder koppeling" option to clear werkruimte
- "+ Nieuwe werkruimte" option at bottom (admin only) → settings
- Uses Werkruimte type from schema, fetches all werkruimtes via /api/werkruimtes

**Meetings Table: `client/src/components/meetings-table.tsx`**
- "Klant" column → "Werkruimte" with color dot badge
- Sort field changed from "clientName" to "werkruimte"
- Sorts by primaryWerkruimteName (enriched from backend)

**Meetings Page: `client/src/pages/meetings.tsx`**
- clientFilter → werkruimteFilter state
- Filter dropdown lists werkruimtes with color dots (from /api/werkruimtes query)
- Filters by meeting.primaryWerkruimteId instead of clientName

### Phase 3: Agent System (Complete ✅)

**searchMeetings.ts**: clientName param → werkruimteName, fuzzy match against primaryWerkruimteName with clientName fallback
**queryCapabilities.ts**: client aggregation → werkruimte aggregation, detail output with werkruimteName
**meetingMutations.ts**: clientName param → primaryWerkruimteId, calls storage.setPrimaryWerkruimte
**agent.ts**: system prompt updated - "clientName" → "werkruimteName" in all instructions and examples
**insights.ts**: matchType "clientName" → "werkruimte", scoring uses werkruimte name with clientName fallback

---

## VORIGE SESSIE: Werkruimtes — Intra-Tenant Access Control (13 Feb 2026)

### Context
All meetings within a tenant are visible to all tenant users. Werkruimtes add the ability to restrict visibility based on per-person membership. This is critical for multi-department organizations (e.g., sales team can't see HR meetings). Two concepts:
- **Werkruimte** (NEW): Container for meetings + determines access via members. Types: client, project, internal, general.
- **Categorie** (EXISTING): Cross-cutting type label (orthogonal to werkruimtes, NOT a security boundary).

### Design Decisions
- Access = person-level membership in werkruimtes (departments are a future shortcut for bulk membership)
- Meetings without werkruimte remain visible to everyone (backwards compatible)
- Default "Algemeen" werkruimte for catch-all (cannot be deleted/renamed)
- Tier integration: Starter=0 werkruimtes (disabled), Pro=10, Enterprise=unlimited
- Two views: werkruimte page (deep, per-client) and vergaderingen page (broad, cross-cutting with filters)
- Pipeline has NO user context — meetings start unassigned, assigned later via rules or manually

### Phase 1: Backend Foundation (Complete ✅)

**Migration: `migrations/0023_werkruimtes.sql`**
- 3 new tables: `werkruimtes`, `werkruimte_leden`, `meeting_werkruimtes`
- Full RLS via `tenant_matches(tenant_id)` + FORCE ROW LEVEL SECURITY
- Indexes on all FK columns + composite indexes for common queries
- UNIQUE constraints: (werkruimte_id, user_id), (meeting_id, werkruimte_id)
- CASCADE deletes on all foreign keys

**Schema: `shared/schema.ts`**
- `werkruimteTypeEnum` = z.enum(["client", "project", "internal", "general"])
- `werkruimtes`, `werkruimteLeden`, `meetingWerkruimtes` pgTable definitions
- Insert schemas, input schemas, and TypeScript types
- 7 werkruimte activity actions added to activityActionEnum
- `werkruimtes` limits added to SUBSCRIPTION_TIERS (0/10/Infinity)

**Storage: `server/storage.ts` + `server/supabase-storage.ts`**
- 16 werkruimte methods added to IStorage interface
- Full MemStorage implementations for all methods
- Full SupabaseStorage implementations with `withTenant()` + raw SQL + RLS
- Transform helpers: `transformWerkruimteFromDb` / `transformWerkruimteToDb`
- Defense-in-depth: RLS session context + explicit WHERE tenant_id clauses

**Routes: `server/routes/werkruimtes.ts`**
- CRUD: GET/POST/PATCH/DELETE `/api/werkruimtes`
- Membership: GET/POST/DELETE `/api/werkruimtes/:id/leden`
- Meetings: GET `/api/werkruimtes/:id/meetings`
- Assignment: GET/POST/DELETE `/api/meetings/:meetingId/werkruimtes`
- Security: requireAuth + requireTenant on all routes, requireTenantAdmin on writes
- Tier limit check on creation, default werkruimte protection, user-tenant verification

**Tier Limits: `server/utils/tier-limits.ts`**
- `checkWerkruimteLimit()` — excludes default werkruimte from count, handles tier=0 (disabled)

### Phase 2: Settings UI (Complete ✅)

**Settings page: `client/src/pages/settings.tsx`**
- Werkruimtes section after Categorieën (tier-gated: Pro+ only)
- Table view: name, type badge, color dot, description, member count, actions
- Create/edit dialog: name, type selector, description, color picker
- Member management dialog: current members list, add/remove from tenant users
- "Algemeen" row locked (no delete, no name edit)

**Sidebar: `client/src/components/app-sidebar.tsx`**
- Added "Werkruimtes" to settingsSubItems (after Categorieën, icon: Layers)

### Phase 3: Meeting-Werkruimte Assignment UI (Complete ✅)

**Meeting detail: `client/src/pages/meeting-detail.tsx`**
- Werkruimte badges with color dots in metadata section
- "+" button → dropdown to assign werkruimte
- "×" on badges to remove assignment

**Meetings full response: `server/routes/meetings.ts`**
- GET `/api/meetings/:id/full` includes werkruimtes array

### Phase 4: Werkruimte Pages + Navigation (Complete ✅)

**Werkruimtes overview: `client/src/pages/werkruimtes.tsx`**
- Grid of werkruimte cards: name, type badge, color, description, member count
- Click → navigate to `/werkruimtes/:id`
- "Beheer werkruimtes" button links to settings for admins

**Werkruimte detail: `client/src/pages/werkruimte-detail.tsx`**
- Header: name, type badge, color, description, member avatars
- Stats cards: member count, meeting count
- Reuses MeetingsTable component for meetings list

**Routing: `client/src/App.tsx`**
- Added routes: `/werkruimtes` and `/werkruimtes/:id`

**Sidebar: `client/src/components/app-sidebar.tsx`**
- Added "Werkruimtes" to mainNavItems (between Vergaderingen and Kalender, icon: Layers)

### Phase 5: Access Filtering (Complete ✅)

**Storage: `server/supabase-storage.ts`**
- `getMeetingsLight` accepts `options?: { userId?: string; isAdmin?: boolean }`
- Non-admin users: see meetings without werkruimtes OR meetings in werkruimtes they're members of
- SQL: NOT EXISTS / EXISTS subqueries on meeting_werkruimtes + werkruimte_leden

**Routes: `server/routes/meetings.ts`**
- Passes userId + isAdmin from session to getMeetingsLight

### Phase 6: Agent Search Scoping (Complete ✅)

**Capability context: `server/capabilities/types.ts`**
- Added `isAdmin?: boolean` to CapabilityContext

**Agent service: `server/services/agent.ts`**
- Passes isAdmin from AgentContext to CapabilityContext

**Search meetings: `server/capabilities/searchMeetings.ts`**
- Werkruimte access filtering: non-admin users only see meetings they have werkruimte access to

**Chat route: `server/routes/chat.ts`**
- Computes isAdmin and passes it to agent context

### Post-Deploy Bug Fix (13 Feb 2026) ✅

**Bug: Werkruimtes not appearing after creation**
- Root cause: `getWerkruimteLeden` SQL query referenced `u.name` (non-existent column) instead of `u.display_name` in both SELECT and ORDER BY clauses
- Impact: GET /api/werkruimtes returned 500 because the member count enrichment loop failed
- Fix: Changed to `u.display_name` in both SELECT and ORDER BY (commits d4e5703 + 90e42b6)

### Status: Ready for Testing
All 6 phases implemented, deployed, and migration applied. Awaiting manual testing.

---

## VORIGE SESSIE: Split Meetings into Vergaderingen + Kalender (12 Feb 2026)

### Context
The Meetings page had 2 tabs (processed meetings + scheduled recordings) on one page. Split into two dedicated pages for clearer navigation.

### Changes

**New page: Kalender (`/kalender`)**
- `client/src/pages/kalender.tsx` — Page with title "Kalender", failed recordings banner, and scheduled meetings view
- `client/src/components/failed-recordings-banner.tsx` — Collapsible alert banner showing failed recordings with retry/delete actions; renders nothing when no failures

**Modified: Vergaderingen (`/meetings`)**
- `client/src/pages/meetings.tsx` — Removed tab toggle, URL param handling, and ScheduledMeetingsView import. Title changed to "Vergaderingen". Only shows processed meetings.

**Modified: ScheduledMeetingsView**
- `client/src/components/scheduled-meetings-view.tsx` — Added `hideFailed` prop (default `false`). When `true`, skips rendering the failed recordings section (handled by the banner instead).

**Modified: Routing & Navigation**
- `client/src/App.tsx` — Added `/kalender` route
- `client/src/components/app-sidebar.tsx` — "Meetings" → "Vergaderingen" (Radio icon), added "Kalender" (Calendar icon)
- `client/src/components/upcoming-recordings.tsx` — Updated links from `/meetings?tab=scheduled` → `/kalender`

---

## VORIGE SESSIE: Recall.ai Webhook Redesign & Bot Dedup (12 Feb 2026)

### Context
User testing revealed 3 critical issues: (1) duplicate meetings from webhook events + transcript not fetched, (2) bot joining meeting twice, (3) pipeline queue not ready for multi-tenant scale. Comprehensive 3-phase fix implemented.

### Phase 1: Bot Double-Join Prevention ✅

**1a. Expanded `getActiveRecordingByMeetingUrl` status filter**
- `server/supabase-storage.ts` + `server/storage.ts`
- Changed from `IN ('scheduled', 'joining', 'recording')` to `NOT IN ('cancelled', 'failed')`
- Now catches bots in `processing`, `completed`, and all intermediate states

**1b-d. URL dedup on all scheduling endpoints**
- `server/routes/recordings.ts`
- `POST /api/recordings` (manual): Returns 409 with "Er is al een notulist actief voor deze meeting"
- `schedule-event/:eventId`: Returns 409 (skips if same calendarEventId)
- `sync-calendar`: Silent skip, counted as `alreadyScheduled`

### Phase 2: Webhook Redesign — Single Trigger + Background Media Collection ✅

**2a. Single definitive trigger**
- `server/routes/recall-webhook.ts`
- Replaced 5-condition `isTranscriptReady` with: `bot.done` + `statusCode=done` (Recall's terminal event)
- Other events still update recording status but don't trigger media collection

**2b. Atomic recording claim**
- In-memory `activeMediaCollections` Set prevents concurrent duplicate processing
- DB check: `meetingId IS NULL` before creating meeting

**2c-d. Background media collection**
- Webhook responds immediately with 200
- New `collectMediaAndCreateMeeting()` runs in background:
  - 10 retries with progressive delays: 5s, 10s, 15s, 20s, 30s, 45s, 60s, 90s, 120s, 180s (~9.5 min total)
  - Audio fetch with 3 retries (10s, 30s, 60s delays)
  - Marks as "failed" (not stuck "processing") if transcript unavailable after all retries
- Replaced blocking `handleTranscriptReady` with non-blocking flow

**2e. Stuck recording recovery sweep**
- `server/services/recall-scheduler.ts`: Every scheduler run checks for stuck recordings
- `getStuckProcessingRecordings()` added to IStorage, MemStorage, SupabaseStorage
- Finds recordings stuck in `processing` with `meetingId IS NULL` for >15 min
- Re-triggers `collectMediaAndCreateMeeting` (limit: 10 per sweep)

### Phase 3: Pipeline Queue — Concurrent Per-Tenant Processing ✅

**3a. Per-tenant concurrency in `pipelineQueue.ts`**
- Configurable worker pool via `PIPELINE_CONCURRENCY` env var (default: 3)
- Per-tenant max: 1 (prevents Claude rate limits within a tenant)
- `activeTenants` Set tracks which tenants are being processed
- `claimNextItem()` uses `WHERE tenant_id NOT IN (...)` for fair round-robin scheduling
- Workers run in parallel — 3 different tenants processed simultaneously

**3b. Database indexes**
- `migrations/0021_webhook_and_queue_indexes.sql`
- `idx_pipeline_queue_tenant_pending`: Per-tenant fair scheduling
- `idx_scheduled_recordings_stuck`: Stuck recording recovery
- `idx_scheduled_recordings_url_dedup`: URL dedup with expanded statuses

### Files Modified

| File | Phase | Change |
|------|-------|--------|
| `server/supabase-storage.ts` | 1a, 2e | Expand status filter + `getStuckProcessingRecordings` |
| `server/storage.ts` | 1a, 2e | MemStorage + IStorage interface changes |
| `server/routes/recordings.ts` | 1b-d | URL dedup on 3 endpoints |
| `server/routes/recall-webhook.ts` | 2a-d | Full webhook rewrite |
| `server/services/recall-scheduler.ts` | 2e | Stuck recording recovery sweep |
| `server/services/pipelineQueue.ts` | 3a | Per-tenant concurrent workers |
| `migrations/0021_webhook_and_queue_indexes.sql` | 3b | 3 new indexes |

### Verification
All 3 phases verified with `npx tsc --noEmit` (0 errors) and `npm test` (69/69 tests pass).

---

## VORIGE SESSIE: Pre-Production Audit Fixes (12 Feb 2026)

### Context
5-agent pre-production audit completed (UI/UX, Senior Dev, Security, Marketing, End User).
Overall score: 7/10. Results in `AUDIT_RESULTS.md`. Now executing fixes in 5 phases.

### SaaS Transition
The product is now a true SaaS: platform provides the meeting bot (Recall.ai) and AI (Anthropic Claude).
Tenants no longer need to configure: Fireflies API key, Anthropic API key, LLM model selection, LLM usage costs.

### Completed Phases

**Phase 1: SaaS UI Cleanup** ✅
- Removed "Fireflies" nav item from sidebar (`app-sidebar.tsx`)
- Removed "Verbruik" (LLM usage) nav item from sidebar
- Removed from `settings.tsx`: recording provider toggle, Fireflies bulk import section, LLM model selector, Fireflies API key fields, Anthropic API key fields, LLM usage statistics section
- Updated integration status banner text (no more "environment variables" mention)
- Changed LLM model default from haiku to sonnet
- Cleaned up dead code: removed unused state variables, queries, handlers, types, imports

**Phase 2: Onboarding Rewrite** ✅
- Rewrote `onboarding-guide.tsx`: removed Fireflies API, Claude API, and Fireflies Import steps
- Renumbered 8 steps (was 10): Meeting Bot → Categories → AI Prompt → Branding → Automation → Microsoft 365 → Owner Assignment → Done
- Updated final step messaging for SaaS model

**Phase 3: Backend Lockdown** ✅
- Schema defaults updated: `recordingProvider` → "recall", `llmModel` → "claude-sonnet-4-5"
- `DEFAULT_SYSTEM_SETTINGS` and `DEFAULT_RECORDING_SETTINGS` updated
- Added `PLATFORM_MANAGED_KEYS` / `TENANT_MANAGED_KEYS` partitioning in `server/services/index.ts`
- Config GET endpoint filters platform-managed keys for non-superadmins
- Config PATCH endpoint rejects platform-managed key changes for non-superadmins
- LLM config PATCH strips `model` field for non-superadmins

**Phase 4: Critical Security Fixes** ✅
- Scheduler endpoints (recordings.ts) now require `requireSuperAdmin`
- LLM usage endpoints (admin.ts) now require `requireTenantAdmin`
- Task PATCH (meetings.ts) now validates with Zod `actionItemSchema.partial()`

**Phase 5: Quick Wins** ✅
- Webhook test endpoints (`/api/webhook/recall/test`, `/api/webhook/stripe/test`) now require superadmin auth
- Webhook simulate endpoint (`/api/webhook/recall/simulate`) now requires superadmin auth
- All recording debug endpoints (`/diagnose`, `/debug/*`) now require superadmin auth
- Total: 9 endpoints hardened with `requireSuperAdmin`

### Post-Deploy Bug Fixes (User Testing)

**Bug 1: Citations crash in Inzichten chat** ✅
- White screen crash: `ne.citations.map is not a function`
- Root cause: JSONB `citations` column has `.default([])` in schema, but existing DB rows have `null`
- Fix 1 (frontend): Added `Array.isArray(message.citations)` guard in `inzichten.tsx:528`
- Fix 2 (backend): Normalized citations to `[]` in both GET endpoints in `server/routes/chat.ts`

**Bug 2: Activity log "Activiteiten Overzicht" shows nothing** ✅
- Root cause: Backend `getActivityLogStats()` returns `{totalEdits, editsByAction}` but frontend expects `{totalActions, actionCounts}`
- The admin route (`admin.ts:1012`) already mapped the field names, but the logging route (`logging.ts:36`) returned raw stats
- Fix: Transformed response in `logging.ts` to match frontend `ActivityLogStats` interface (totalActions, actionCounts, period, insights)

### Verification
All 5 phases + bug fixes verified with `npx tsc --noEmit` (0 errors) and `npm test` (69/69 tests pass).

### Summary of All Changes

| File | Changes |
|------|---------|
| `client/src/components/app-sidebar.tsx` | Removed Fireflies + Verbruik nav items |
| `client/src/pages/settings.tsx` | Removed 6 obsolete sections, cleaned dead code, updated defaults |
| `client/src/components/onboarding/onboarding-guide.tsx` | Rewritten: 10→8 steps, SaaS-focused |
| `shared/schema.ts` | Defaults: recordingProvider→recall, llmModel→sonnet |
| `server/services/index.ts` | Added PLATFORM_MANAGED_KEYS / TENANT_MANAGED_KEYS |
| `server/routes/config.ts` | Platform key filtering (GET + PATCH), model strip for non-SA |
| `server/routes/recordings.ts` | requireSuperAdmin on scheduler + debug endpoints |
| `server/routes/admin.ts` | requireTenantAdmin on LLM usage endpoints |
| `server/routes/meetings.ts` | Zod validation on task PATCH |
| `server/routes/recall-webhook.ts` | requireSuperAdmin on test + simulate endpoints |
| `server/routes/stripe-webhook.ts` | requireSuperAdmin on test endpoint |
| `server/routes/common.ts` | Export PLATFORM_MANAGED_KEYS |

---

## VORIGE SESSIE: v2.6 → v2.7 → v2.8 Review Fixes (10 Feb 2026)

### B2B-SAAS-SKILLS.md v2.7 → v2.8: 16 Review Issues Opgelost

3-agent review panel (cold-start 8.8/10, compile chain 8.5/10, completeness 8.5/10) vond 16 issues. Alle opgelost:

**HIGH opgelost (5):**
- SQL CHECK: `'account_deletion'` + `'used'` toegevoegd aan tokens.type/status constraints
- GDPR: `createSecureToken()` vervangen door inline crypto patroon (matching §9)
- GDPR: email functienamen gefixt (`sendDeletionConfirmEmail`), `sendDeletionCompletedEmail` verwijderd
- Invitation route: `requireRole("admin")` → `requireTenantAdmin` (bestaande middleware)

**MEDIUM opgelost (6):**
- `upsertStripeSubscription`: 6 ontbrekende velden in UPDATE (price_id, billing_interval, trial_*, etc.)
- `GET /api/billing/status` route handler toegevoegd
- `logSecurityEvent`: userId type verbreed naar `string | null`
- `sendInvitationEmail`: ontbrekende `expiresAt` parameter toegevoegd
- `stripeWebhookRouter`: als aparte export gedefinieerd
- COALESCE NULL-reset: `updateUserDetails` herschreven met `this.sql(obj)` helper

**LOW opgelost (5):**
- `createItem`: metadata in INSERT
- `processedWebhooks.source`: `.default("stripe")` in Drizzle
- `updateStripeSubscription`: uitgebreid naar 6 updatebare velden
- `revokePendingTokens` MemStorage: lege string tenantId = alle tenants
- `getUserByEmail` MemStorage: case-insensitive matching

### B2B-SAAS-SKILLS.md v2.6 → v2.7: 20 Review Issues Opgelost

3-agent review panel (cold-start 8.5/10, compile chain 8/10, completeness 9.0/10) vond 20 issues. Alle opgelost:

**HIGH opgelost (8):**
- `updateTokenStatus`: usedAt? parameter toegevoegd aan SupabaseStorage
- `markWebhookProcessed`: source? parameter in IStorage + MemStorage
- registerSchema: §3/§7b geünificeerd (companyName + displayName)
- `getAllSettings`: SupabaseStorage implementatie toegevoegd
- `TenantProvisioningService`: singleton export
- `billingConfigHandler`: handler + billingRouter export gedefinieerd
- Invitation accept route: Express endpoint in public-auth
- Invitation create endpoint: admin POST /api/admin/invitations

**MEDIUM opgelost (10):**
- `revokePendingTokens`: tenant_id filter in SupabaseStorage
- `getUserTenants`: ut.created_at in SELECT
- `getPasswordHistory`: limit optional (default 5)
- `getBillingStatus`: ... pseudo-code → echte objecten
- Session config: standalone duplicaat verwijderd, verwijst naar setupAuth()
- JobQueue: alle sql → this.sql
- GDPR endpoints: volledige werkende implementatie (4 routes)
- SubscriptionGuard: export keyword
- `getUserByUsername`: toegevoegd aan IStorage + MemStorage
- `getPendingTokensByPrefix`: type versmald naar Token["type"]

**LOW opgelost (2):**
- token_prefix: varchar({length:8}) in Drizzle
- createdAt/updatedAt: .notNull() op alle defaultNow() kolommen

---

### B2B-SAAS-SKILLS.md v2.5 → v2.6: 23 Interne Inconsistenties Opgelost

3-agent review panel (cold-start developer 7.5/10, compile chain validator, completeness finder 7.5/10) vond 23 echte interne inconsistenties (na filtering van codebase-specifieke vergelijkingen):

**BLOCKERS opgelost (2):**
- SupabaseStorage ↔ IStorage: 11 method signature mismatches gefixt (createUser, addUserToTenant, removeUserFromTenant, getPendingTokensByType, getChatThreads, createChatMessage, deleteChatThread, createActivityLog, getActivityLogs, updateSystemSettings, updateStripeSubscription)
- createChatMessage RLS crash: verwijderd `getChatThread(threadId, "")`, nu via tenantId parameter

**HIGH opgelost (6):**
- InsertUser type verwijderd → createUser nu met individuele parameters (matcht IStorage)
- ChangePassword page component toegevoegd (was blank bij `mustChangePassword = true`)
- Password reset tokens: `tokenPrefix` toegevoegd voor O(1) lookup
- `getBillingStatus`: `export` keyword toegevoegd
- Session config placeholder vervangen door werkelijke configuratie in setupAuth()
- Logout endpoint: CSRF-exempt commentaar + `csrf_token` cookie cleared

**MEDIUM opgelost (10):**
- getTenantInvitations: WHERE tenant_id filter toegevoegd (was cross-tenant leak)
- switch-tenant endpoint: volledig geïmplementeerd in setupAuth()
- registerSchema: toegevoegd aan §3 shared/schema.ts
- rawBody type: `unknown` → `Buffer` in type augmentation
- Pagination: verduidelijkt als upgrade/vervanging van basic getItems
- `this.sql(obj)` bulk-insert: postgres library helper gedocumenteerd
- activity_log.tenant_id: TEXT vs UUID keuze uitgelegd in SQL comment
- updateUserDetails: locked_until nu consistent met COALESCE-patroon
- getUsersForTenant: return type aligned met IStorage ({id, role})
- healthCheck: toegevoegd aan IStorage + MemStorage + SupabaseStorage

**LOW opgelost (3):**
- generateCsrfToken: locatie-note (zelfde bestand als setupAuth)
- queryKey join: `queryKey[0]` ipv `queryKey.join("/")` (voorkomt extra /)
- Infinity serialisatie: waarschuwing toegevoegd bij SUBSCRIPTION_TIERS

**Extra fixes:**
- createUserWithDetails: toegevoegd aan SupabaseStorage (was alleen in IStorage + MemStorage)
- Layout component: verduidelijkt dat beschermde routes IN layout staan, publieke BUITEN
- Tenant switching §5: cross-reference naar setupAuth() implementatie

---

### B2B-SAAS-SKILLS.md v2.4 → v2.5: 30 Review Issues Opgelost (eerder)

3-agent review panel (cold-start developer, copy-paste validator, build-order checker) scoorde 8.2/10 en identificeerde 30 issues. Alle opgelost:

**BLOCKERS opgelost (3):**
- SupabaseStorage uitgebreid van 6 naar ~35 methoden (volledige IStorage implementatie)
- `setupAuth()` wrapper + imports + Express Router bundeling patroon toegevoegd
- `errorHandler` ZodError type guard gefixt (property check + type narrowing)

**HIGH opgelost (13):**
- ResetPassword frontend URL → `/api/password-reset/:token/complete` (matcht backend)
- `change-password` endpoint + frontend route + App.tsx route toegevoegd
- `processedEvents` → `isWebhookDuplicate()`/`markWebhookProcessed()` (§22 harmonisatie)
- Pagination null tenantId fix (withTenant/withoutTenant branch)
- `stripeService.getBillingStatus()` → direct `getBillingStatus()` import
- MFA `mfaPending` race condition: login retourneert nu resultaat, check `result.mfaRequired`
- `buildSessionUser` inline in brownfield dual-auth (+ try/catch voor async)
- Import block voor §7 auth.ts (bcrypt, crypto, storage, schemas, audit, email)
- `loginLimiter` stub note in Fase 1 checklist
- Stripe SDK `new Stripe()` instantiatie in §15
- `billingInterval` mapping uit Stripe subscription interval
- `recallWebhookRouter` verwijderd (domain-specifiek overblijfsel)
- `dualAuthMiddleware` async: try/catch + next(err) voor error handler

**MEDIUM opgelost (14):**
- CSRF mechanisme uitleg toegevoegd (3 kanalen = 1 token via meerdere transport)
- Session cookie default → `sameSite: "lax"` (Model A, aanbevolen)
- `req.user.activeTenantId` → `user.activeTenantId` (geen passport.js vermenging)
- `InsertStripeSubscription` currentPeriod types → `Date | null`
- `tenant_matches()` TEXT vs UUID uitleg in SQL comment
- Session table RLS `allow_all` policy note
- Billing checkout route handler toegevoegd (was alleen service functie)
- `lockedUntil` type fix: `Date | null` ipv `Date`
- `sql` import/constructor in JobQueue + health check
- Spinner imports in SubscriptionGuard en LoadingScreen
- SubscriptionGuard stub note in Fase 1 checklist
- shadcn packages (clsx, tailwind-merge, cva, tailwindcss-animate) in package.json
- `ADMIN_EMAIL` + `VITE_API_URL` in §26 env var lijst
- SQL `--` comment → `//` in TypeScript block (§22)

---

## v2.4 Domain-Agnostisch Refactor (10 Feb 2026)

### B2B-SAAS-SKILLS.md v2.3 → v2.4: Universele SaaS Blueprint

Document omgebouwd van Schakel-specifiek naar domain-agnostisch. Het document is nu bruikbaar als blauwdruk voor élke B2B SaaS applicatie.

**Domain-specifiek → Generiek (~35 edits):**
- `meetings` tabel → `items` tabel (met `metadata JSONB` ipv `participants JSONB`)
- `categories` tabel verwijderd (domain-specifiek)
- `pipeline_queue` → `job_queue` (met `entity_id` + `job_type` ipv `meeting_id`)
- `PipelineQueue` → `JobQueue` class met generieke `enqueue(entityId, tenantId, jobType)`
- `Meeting`/`Category` types → `Item`/`InsertItem` types
- `isMeetingOwner`/`ownerPriority` verwijderd uit `user_tenants`
- `checkMeetingLimit` → `checkEntityLimit`, `getMeetingCountForMonth` → `getEntityCountForPeriod`
- Alle IStorage/MemStorage/SupabaseStorage methoden omgezet
- Alle `/api/meetings` → `/api/items`
- Alle tests en RLS voorbeelden omgezet
- Bron-attributie veralgemeniseerd

**Non-domain fixes (code consistency review):**
- Password reset URL: frontend `/api/auth/forgot-password` → `/api/password-reset/request` (matcht backend)
- `AUTH_1009` (AUTH_PASSWORD_CHANGE_REQUIRED) toegevoegd aan ErrorCodes + status mappings + messages
- `processed_webhooks` dual schema: §22 verwijst nu naar §2 definitie (single source of truth)
- Drizzle booleans: `.notNull()` toegevoegd aan `billingExempt`, `isActive`, `mustChangePassword`, `failedLoginAttempts`

**Verificatie (0 resterende issues):**
- `meeting/Meeting`: alleen in changelog
- `Math.random`: 0 resultaten
- `as any`: alleen in uitleg-comment
- `admin123`: 0 resultaten
- `categories/Category`: 0 resultaten

---

### B2B-SAAS-SKILLS.md v2.2 → v2.3: Agent Review Gaps Opgelost

3-agent review panel (empty project, small project, large project) identificeerde 22 unieke gaps verdeeld over 4 categorieën. Alle opgelost:

**Categorie A: Frontend Bootstrapping (5 items) ✅**
- `"type": "module"` in package.json
- devDependencies (@types/express, @types/express-session, @types/connect-pg-simple, @types/bcrypt, @types/node)
- `client/index.html` Vite entry template
- `client/src/main.tsx` met QueryClientProvider → AuthProvider → App chain
- `client/src/index.css` met shadcn CSS variables (light + dark mode)

**Categorie B: Incomplete Implementaties (5 items) ✅**
- `SecurityEvents` constants object (19 event types)
- `categories` CREATE TABLE + `processed_webhooks` CREATE TABLE
- Alle 14 resterende Drizzle pgTable definities (was "... volgen hetzelfde patroon")
- Volledige MemStorage (~35 methoden, was placeholder)
- Stop-impersonate + acceptInvitation placeholder code vervangen door werkende implementatie
- Server imports (fs, path, storage, validateEnvironment)

**Categorie D: Frontend Pages (5 items) ✅**
- 6 volledige page templates: Register, AcceptInvitation, ForgotPassword, ResetPassword, Landing, Layout
- Register route toegevoegd aan App.tsx
- Fase 0/1 checklists bijgewerkt

**Categorie C: Brownfield Migration Guide (7 items) ✅**
- JWT → Session-based auth transitie (big bang + dual-auth)
- INTEGER → UUID PK migratie (2 opties)
- Role system migratie (flat → two-level)
- Stripe charges → subscriptions
- ORM adapter patronen (Prisma, Knex)
- Frontend router compatibiliteit (wouter → react-router)
- Multi-tenancy script, RLS generatie, file storage migratie, rollback strategieën

**Resultaat:** Document v2.2 → v2.3. Reviews in `/reviews/b2b-saas-skills-evaluation.md` en `/reviews/b2b-saas-skills-evaluatie.md`.

### B2B-SAAS-SKILLS.md v2.1 → v2.2: 28 Gaps Opgelost

Kritische analyse onthulde 28 gaps verdeeld over 4 categorieën. Alle opgelost:

**Categorie A: Ontbrekende Types/Interfaces (10 items) ✅**
- `SessionData.createdAt` toegevoegd aan express-session augmentation
- `SessionUser.impersonationStartedAt` toegevoegd
- `User.failedLoginAttempts` + `lockedUntil` in CREATE TABLE, Drizzle schema, en types
- `IStorage.close()` methode + implementaties in MemStorage en SupabaseStorage
- `IStorage.isWebhookProcessed()` + `markWebhookProcessed()` + implementaties
- `IStorage.updateSystemSettings()` + implementatie
- `IStorage.updateUserTenantMeetingOwner()` + `getPendingTokensByPrefix()`
- `tokens.token_prefix` kolom in CREATE TABLE + index + createInvitationToken flow
- Alle missende types gedefinieerd: Token, InsertToken, StripeSubscription, InsertStripeSubscription, Meeting, InsertMeeting, Category, InsertCategory, ActivityLog, InsertActivityLog, ChatThread, InsertChatThread, ChatMessage, InsertChatMessage, InvitationStatus, TierLimitResult
- `errorMessages` + `errorStatusCodes` mappings + `ApiError.statusCode` property

**Categorie B: Incomplete Implementaties (5 items) ✅**
- `targetSessionData` in §12 impersonation volledig opgebouwd vanuit target user
- `DEFAULT_CATEGORIES` + `DEFAULT_SYSTEM_SETTINGS` constants gedefinieerd
- `isRateLimitError()` methode aan PipelineQueue toegevoegd
- Alle 5 email templates volledig geïmplementeerd (was alleen uitnodiging)
- `processed_webhooks` SQL uit comments gehaald naar uitvoerbare SQL

**Categorie C: Architectuurbeslissingen gedocumenteerd (5 items) ✅**
- SubscriptionGuard fail-open bij errors: bewuste keuze gedocumenteerd
- Multi-tenant MFA gedrag: sectie toegevoegd met uitleg over tenant-switching
- Token O(n) → O(1) prefix-lookup: volledig geïmplementeerd in validateToken()
- db:push → generate+migrate transitie: stap-voor-stap handleiding
- Same-origin vs cross-origin: waarschuwing dat document cross-origin toont als default

**Categorie D: Ontbrekende Componenten (8 items) ✅**
- Frontend: LoadingScreen, Spinner, NotFound, SubscriptionBlocked, WarningBanner, ErrorBoundary
- AppSidebar navigatie component
- Pricing pagina met tier cards + interval toggle
- Paginatie patroon (PaginationParams, PaginatedResult, storage + route + hook)
- Timezone utilities (formatDate, timeAgo, getUserTimezone)
- CI/CD: GitHub Actions workflow (check + test + build)
- Structured logging (logger.ts met JSON output in productie)

**Resultaat:** Document van 4291 → 5110 regels. Versie 2.1 → 2.2.


### Extra: `as any` Cast Cleanup (9 casts → 0 in productie code)

| Bestand | Was | Nu |
|---------|-----|-----|
| `supabase-storage.ts:814` | `(data as any)?.length` | `(data ?? []).length` |
| `admin.ts:1034` | `action as any` | `action as ActivityAction \| undefined` |
| `errors.ts:258` | `err as any` | `err as import("zod").ZodError` |
| `agent.ts:483` | `] as any` content blocks | Widened messages type |
| `agent.ts:635` | `result.data as any` | `Record<string, unknown>` + typed narrowing |
| `pipeline.ts:359,466` | `(llmConfig as any).customInstructions` | Removed (dead code) |
| `stripe.ts:341-342` | `(subscription as any).current_period_*` | `subscription.items.data[0]?.current_period_*` |

Resultaat: **0 `as any` casts in productie server code.** Alleen 2 in test files (voor mocking).

---

### Track A Review (eerder in deze sessie)

### Context

Expert review panel (Senior Developer + Senior Architect + Senior Security Specialist) heeft het B2B-SAAS-SKILLS.md document en de codebase beoordeeld. Unaniem score: **7.5/10**. Rapporten in `/reviews/`:
- `senior-developer-review.md` — Code quality, bugs, DX
- `senior-architect-review.md` — Scalability, resilience, operations
- `senior-security-review.md` — OWASP, crypto, session management
- `expert-consensus-review.md` — Geconsolideerde bevindingen + aanbevelingen

### Aanpak: Eerst applicatie (Track A), dan documentatie (Track B)

Lessons learned uit Track A nemen we mee naar Track B (B2B-SAAS-SKILLS.md updates).

---

### TRACK A IMPLEMENTATIEPLAN

#### Fase 1: Security Fixes (CRITICAL) ✅ DONE

**Fix 1: Session Regeneration na Login** ✅
- **Probleem**: Geen `req.session.regenerate()` na authenticatie → session fixation attack
- **Bestanden**:
  - `server/auth.ts:328` — login (`req.session.user = sessionUser`)
  - `server/auth.ts:469-475` — MFA verify (`req.session.user = sessionUser`)
  - `server/routes/public-auth.ts:453` — register reactivation
  - `server/routes/public-auth.ts:579` — new registration
- **Fix**: Wrap elke `req.session.user =` in `req.session.regenerate()` callback
- **Let op**: Na regenerate() is het een nieuw session object — MFA state is automatisch weg

**Fix 2: MFA Code Generatie** ✅
- **Probleem**: `Math.random()` is niet cryptografisch veilig
- **Bestand**: `server/auth.ts:13-14`
- **Fix**: `Math.floor(100000 + Math.random() * 900000)` → `crypto.randomInt(100000, 1000000)`
- **Let op**: `crypto` is al geimporteerd op regel 3

**Fix 3: Encryption Salt** ✅
- **Probleem**: `crypto.scryptSync(key, "salt", 32)` — hardcoded salt
- **Bestand**: `server/utils/encryption.ts:51-52`
- **Fix**: Gebruik `ENCRYPTION_SALT` env var, fallback naar afgeleide salt
- **MIGRATIE**: Bestaande data is encrypted met salt="salt". Zet `ENCRYPTION_SALT=salt` in productie env vars voor backwards compatibility. Nieuwe installaties gebruiken random salt.
- **Extra fix**: `decrypt()` regel 93-96 — catch block retourneert ciphertext als plaintext. Fix: alleen plaintext retourneren als het niet encrypted lijkt (`!isEncrypted()`), anders throw error.

**Fix 4: Rate Limiting op MFA Verify** ✅
- **Probleem**: `/api/auth/mfa/verify` geen rate limiter → brute-force op 6-digit codes
- **Bestand**: `server/auth.ts:383`
- **Fix**: Nieuwe `mfaVerifyRateLimiter` (max 5 pogingen per 15 min) toevoegen als middleware

**Fix 5: CSRF localStorage Beperken** ✅
- **Probleem**: CSRF tokens in localStorage uitleesbaar bij XSS
- **Bestand**: `client/src/lib/queryClient.ts:7-29`
- **Fix**: localStorage fallback alleen in `import.meta.env.DEV` mode. Productie gebruikt alleen cookie.

#### Fase 2: Reliability & Scalability (HIGH) ✅ DONE (10 Feb 2026)

**Fix 6: Graceful Shutdown** ✅
- `setupGracefulShutdown(server)` in `server/app.ts` — handles SIGTERM/SIGINT
- `close()` methode op IStorage, SupabaseStorage (`sql.end()`), MemStorage (no-op)
- 30-seconde force-exit timeout, stops HTTP server, Recall scheduler, DB connections

**Fix 7: Connection Pool Vergroten** ✅
- `server/supabase-storage.ts:91` — `max: parseInt(process.env.DB_POOL_MAX || "10", 10)`
- Configureerbaar via `DB_POOL_MAX` env var, default verhoogd van 5 naar 10

**Fix 8: Token Validatie O(n) → O(1)** ✅
- `token_prefix` kolom (eerste 8 chars raw token) — migratie `0017_token_prefix.sql`
- `getPendingTokensByPrefix()` in IStorage, MemStorage, SupabaseStorage
- `validateToken()` zoekt eerst op prefix, fallback naar O(n) voor oude tokens
- GDPR route ook bijgewerkt met prefix-first lookup

**Fix 9: Webhook Dedup Persistent** ✅
- `processed_webhooks` tabel — migratie `0018_webhook_dedup.sql`
- L1 (in-memory) + L2 (database) dedup in `recall-webhook.ts`
- L2 write is non-blocking, L2 failure is non-fatal (L1 beschermt nog steeds)
- `isWebhookProcessed()`, `markWebhookProcessed()`, `cleanupOldWebhooks()` in IStorage

**Fix 10: Seed Script Hardening** ✅
- `server/seed.ts` — production: vereist `ADMIN_PASSWORD` env var (crash als niet gezet)
- Development: genereert `crypto.randomBytes(16).toString("base64url")` als geen password gezet
- Geen hardcoded "admin123" meer

#### Fase 3: Hardening (MEDIUM) ✅ DONE

| # | Fix | Bestand | Status |
|---|-----|---------|--------|
| 11 | Account lockout na herhaalde fails | `server/auth.ts` | ✅ 5 pogingen → 15 min lockout, 423 status |
| 12 | Session absolute timeout (30 dagen) | `server/auth.ts` | ✅ `createdAt` in session, middleware check |
| 13 | Impersonation timeout (1 uur) | `server/auth.ts` | ✅ `impersonationStartedAt` auto-expire |
| 14 | Tenant switch isActive check | `server/auth.ts` | ✅ Tenant lookup + `isActive` check |
| 15 | Zod .max() op alle string schemas | `shared/schema.ts` | ✅ Alle user-input strings gelimiteerd |
| 16 | Database indexes | `migrations/0020_performance_indexes.sql` | ✅ 15 indexes op veelgebruikte kolommen |
| 17 | Express type augmentation | `errors.ts`, `gdpr.ts`, `subscription.ts` | ✅ `any` casts verwijderd, proper typing |

#### Track A Verificatie:
```bash
npx tsc --noEmit  # ✅ Clean compile
npm test           # ✅ 69/69 tests pass (4 test files)
```

#### Self-Review Resultaat (10 Feb 2026)

**Automatische checks:**
- ✅ TypeScript compileert zonder fouten
- ✅ 69/69 tests slagen (storage, tier-limits, auth-security, encryption)
- ✅ Geen `(req as any)` casts meer — 3 webhook rawBody casts opgeschoond naar `req.rawBody as Buffer`

**Handmatige code review bevindingen:**

| Item | Status | Details |
|------|--------|--------|
| `failedLoginAttempts`/`lockedUntil` consistentie | ✅ | Schema, IStorage, MemStorage, SupabaseStorage allemaal in sync |
| `close()` idempotent | ✅ | Null-guard voorkomt dubbel sluiten |
| `createdAt` op alle auth flows | ✅ | Alle 4 full-auth flows (login, MFA, register, reactivate) zetten `createdAt` |
| Token prefix collision | ✅ | 48-bit prefix + bcrypt verify = veilig, fallback voor oude tokens |
| L1+L2 webhook dedup | ✅ | L2 failure is non-fatal, L1 cache heeft max size + eviction |
| Session destroy + next() | ✅ | express-session cleared req.session synchroon, downstream ziet unauthenticated |
| Remaining `as any` casts | ✅ | Alle 9 pre-existing casts opgelost (admin, errors, agent, pipeline, stripe) |

**Nog niet getest (handmatig te doen na deploy):**
- [ ] Login → session ID wijzigt (DevTools → Cookies)
- [ ] MFA verify → rate limited na 5 pogingen
- [ ] Encrypted settings laden correct na encryption fix
- [ ] CSRF werkt in productie build (geen localStorage)
- [ ] Server graceful shutdown bij SIGTERM
- [ ] Seed script faalt zonder ADMIN_PASSWORD
- [ ] Account lockout na 5 failed logins → 423 response
- [ ] Impersonation auto-expire na 1 uur
- [ ] Tenant switch naar inactive tenant → 403

#### Nieuwe database migraties (draaien voor deploy):
```
migrations/0017_token_prefix.sql         — token_prefix kolom + partial index
migrations/0018_webhook_dedup.sql        — processed_webhooks tabel
migrations/0019_account_lockout.sql      — failed_login_attempts + locked_until
migrations/0020_performance_indexes.sql  — 15 performance indexes
```

#### Nieuwe env vars (optioneel):
```
DB_POOL_MAX=10          # Database connection pool (default: 10, was hardcoded 5)
ENCRYPTION_SALT=salt    # Set expliciet in bestaande productie (backwards compat)
ADMIN_PASSWORD=...      # Vereist in productie voor seed script
```

---

## VORIGE SESSIE: Agent Capabilities Uitbreiden (9 Feb 2026 - Evening)

Alle platform-acties zijn nu beschikbaar als agent capability. Kernprincipe: **"Alles is een capability — voorbereiden op een agentic future"**.

### 5 Nieuwe Agent Mutations - COMPLETED ✅

| Capability | Naam | Wraps | requiredIntegration |
|------------|------|-------|---------------------|
| `schedule_recording` | Notulist Inplannen | Recall.ai service (NIEUW) | `none` |
| `create_outlook_draft` | E-mail Concept Maken | `createOutlookDraftCapability` | `microsoft` |
| `generate_document` | Document Genereren | `generateDocumentsCapability` | `none` |
| `upload_sharepoint` | Uploaden naar SharePoint | `uploadSharePointCapability` | `microsoft` |
| `create_tasks` | Taken Aanmaken | `createProductiveTasksCapability` | `productive` |

Alle 5 capabilities:
- `requiresApproval: true` (human-in-the-loop)
- Gebruiken `createAgenticCapability()` met displayName, icon, whenToUse/whenNotToUse
- Gefilterd op basis van tenant integraties (Microsoft, Productive)

### Architectuur

```
Agent Capabilities (12 totaal)
├── Query (5) - read-only, geen goedkeuring nodig
│   ├── search_meetings
│   ├── get_meeting_details
│   ├── list_categories
│   ├── list_clients
│   └── get_meeting_stats
└── Mutation (7) - schrijven, goedkeuring vereist
    ├── update_meeting_summary (bestaand)
    ├── update_meeting_details (bestaand)
    ├── schedule_recording (NIEUW)
    ├── create_outlook_draft (NIEUW - wraps pipeline)
    ├── generate_document (NIEUW - wraps pipeline)
    ├── upload_sharepoint (NIEUW - wraps pipeline)
    └── create_tasks (NIEUW - wraps pipeline)
```

### Nieuwe Bestanden

| Bestand | Beschrijving |
|---------|-------------|
| `server/capabilities/scheduleRecording.ts` | Notulist inplannen capability |
| `server/capabilities/agentMutations.ts` | 4 thin wrappers voor pipeline capabilities |

### Gewijzigde Bestanden

| Bestand | Wijziging |
|---------|-----------|
| `server/capabilities/index.ts` | 5 capabilities geregistreerd + 2 bugfixes |
| `client/src/pages/inzichten.tsx` | Popover dynamisch op basis van `/api/chat/tools` |
| `server/services/agent.ts` | System prompt met workflow advies voor alle mutations |

### Bugfixes meegenomen

1. **`getCapabilityByName()` zocht op object key (camelCase) i.p.v. capability `.name` (snake_case)** — nu zoekt het op beide
2. **`getAgentCapabilities()` filterde uit `allCapabilities` (incl. pipeline)** — nu filtert uit `agentCapabilities` (query + mutation only)
3. **Frontend popover was hardcoded** — nu dynamisch op basis van API response

### Inzichten Agent Improvements (eerder in deze sessie)

- **Search fix:** `searchMeetings` filter verbreed — was `completed && notesMarkdown`, nu `notesMarkdown || transcript || completed`
- **UI redesign:** Badge row vervangen door popover dropdown met icons en beschrijvingen

### Commits
```
33c012f Add 5 new agent capabilities for full platform action coverage
8b0c0ba Improve Inzichten agent: fix search + redesign available actions
```

---

## VORIGE SESSIE: Security Hardening - 4 Fases (9 Feb 2026)

TypeScript compile errors gefixt (10+) en vervolgens 4-fase security hardening plan volledig uitgevoerd op basis van quality review bevindingen.

### TypeScript Fixes - COMPLETED ✅

10+ compile errors gefixt:
- `categorize.ts` — ontbrekende `tenantId` in `extractMetadata()`
- `generateNotes.ts` — `promptInfo` type union mismatch
- `storage.ts` — `clientName: null`, `AudioStatus` casts, `getUsersForTenant` types
- `supabase-storage.ts` — `getUsersForTenant` signature, `getMeetingsLight` velden
- `shared/schema.ts` — `MeetingLight` type gewijzigd naar `Omit<Meeting, 'transcript'>` (was ook notesMarkdown/notesHtml)
- `auth.ts` + `meeting-detail.tsx` — `Array.from()` voor Map/NodeList iterators
- `dashboard-filter-context.tsx` — `Meeting` → `MeetingLight`
- `recent-errors.tsx` + `review-queue.tsx` — `Meeting` → `MeetingLight`

### Fase 1: Inzichten Agent Fix - COMPLETED ✅

**Probleem:** `services: null as any` in `server/services/agent.ts` — agent mutations crashten.

**Fix:** Echte service instanties geïmporteerd en doorgegeven:
```typescript
services: {
  fireflies: firefliesService,
  claude: claudeService,
  microsoftGraph: microsoftGraphService,
  productive: productiveService,
  htmlTemplate: htmlTemplateService,
},
```

**Bestand:** `server/services/agent.ts`

### Fase 2: tenantId Required in Storage - COMPLETED ✅

**Probleem:** 19 IStorage methods hadden optionele `tenantId?`, waardoor RLS per ongeluk omzeild kon worden.

**Aanpak:** Parameter gewijzigd van `tenantId?: string` naar `tenantId: string | null` (required). `null` = expliciete superadmin/system bypass.

**19 methods gewijzigd in IStorage interface:**
- `getMeeting`, `updateMeeting`, `deleteMeeting`
- `getCategoryByPath`, `getCategory`, `updateCategory`, `deleteCategory`
- `getTeamMemberByEmail`, `getTeamMemberByUserId`
- `getSetting`, `setSetting`, `getAllSettings`, `deleteSetting`
- `getChatThread`, `updateChatThread`, `deleteChatThread`
- `getChatMessages`, `createChatMessage`
- `getScheduledRecordingById`

**40+ callers bijgewerkt in:**
- `server/services/pipeline.ts` — `null` voor system-level, `meeting.tenantId` voor scoped
- `server/routes/meetings.ts` — superadmin pattern: `user.globalRole === "superadmin" ? null : tenantId`
- `server/routes/chat.ts` — tenantId uit session
- `server/routes/config.ts` — tenantId voor category mutations
- `server/routes/recall-webhook.ts` — `recording.tenantId` of `null`
- `server/routes/pipeline.ts` — superadmin pattern
- `server/routes/recordings.ts` — tenantId uit session
- `server/services/index.ts` — `?? null` conversie
- `server/capabilities/meetingMutations.ts` — `context.tenantId`
- `server/services/recall-test-utils.ts` — `null` voor test contexts

### Fase 3: File Storage Tenant Isolation - COMPLETED ✅

**Probleem:** Logo's en bot-avatars werden opgeslagen zonder tenant prefix.

**Fix:** `tenantId` parameter toegevoegd aan `uploadLogo()` en `uploadBotAvatar()`:
```
Oud: logos/{filename}
Nieuw: logos/{tenantId}/{filename}

Oud: bot-avatars/{filename}
Nieuw: bot-avatars/{tenantId}/{filename}
```

Bestaande URLs blijven werken (lazy migration).

**Bestanden:**
- `server/services/supabase-file-storage.ts`
- `server/routes/config.ts`

### Fase 4: Tier Limit Enforcement - COMPLETED ✅

**Probleem:** `SUBSCRIPTION_TIERS` definieerde limieten (Starter: 50 meetings/maand, 1 user; Pro: 250/10) maar deze werden nergens afgedwongen.

**Nieuwe utility:** `server/utils/tier-limits.ts`
- `checkMeetingLimit(tenantId, options?)` — controleert meetings/maand limiet
- `checkUserLimit(tenantId, options?)` — controleert gebruikerslimiet
- Bypass voor superadmins en `billingExempt` tenants

**Nieuwe storage method:** `getMeetingCountForMonth(tenantId)`
- Efficiënte SQL COUNT met `date_trunc('month', CURRENT_TIMESTAMP)`

**Enforcement punten:**
| Route | Check | Response bij limiet |
|-------|-------|---------------------|
| `POST /api/meetings/ingest` | Meeting limiet | 429 + reden |
| Recall webhook (meeting creation) | Meeting limiet | Recording status → "failed" + melding |
| `POST /api/tenant/users/invite` | User limiet | 429 + reden |

### Fase 5: Test Suite - COMPLETED ✅

Automated test suite toegevoegd om security hardening te valideren.

**Config:** `vitest.config.ts` met `npm test` en `npm run test:watch`

**69 tests in 4 bestanden:**
- `server/storage.test.ts` (23 tests) — Tenant isolation voor meetings, settings, categories, chat threads, scheduled recordings. Meeting creation defaults.
- `server/utils/tier-limits.test.ts` (16 tests) — Tier limit enforcement per tier (Starter/Pro/Enterprise), superadmin bypass, billingExempt bypass, SUBSCRIPTION_TIERS constants.
- `server/auth-security.test.ts` (10 tests) — Session regeneration, MFA crypto, CSRF, rate limiting, account lockout.
- `server/utils/encryption.test.ts` (20 tests) — AES-256-GCM encrypt/decrypt, salt configuratie, backwards compatibility.

### Commits
```
90cf519 Fix all TypeScript compilation errors (10+ errors resolved)
2d2b079 Security hardening: require tenantId in storage + fix agent services
6f259a9 Add tenant isolation to file storage paths
591c608 Implement tier limit enforcement for meetings and users
c187477 Add test suite for security hardening validation
```

### Bestanden Gewijzigd (13 bestanden + 1 nieuw)
```
server/services/agent.ts                # Fase 1: real services ipv null
server/storage.ts                       # Fase 2: IStorage interface + MemStorage
server/supabase-storage.ts              # Fase 2+4: signatures + getMeetingCountForMonth
server/services/pipeline.ts             # Fase 2: tenantId in alle storage calls
server/routes/meetings.ts               # Fase 2+4: tenantId + meeting limit check
server/routes/chat.ts                   # Fase 2: tenantId in chat storage calls
server/routes/config.ts                 # Fase 2+3: tenantId voor categories + uploads
server/routes/recall-webhook.ts         # Fase 2+4: tenantId + meeting limit check
server/routes/pipeline.ts               # Fase 2: superadmin pattern
server/routes/recordings.ts             # Fase 2: tenantId
server/services/index.ts                # Fase 2: undefined → null
server/capabilities/meetingMutations.ts # Fase 2: context.tenantId
server/services/recall-test-utils.ts    # Fase 2: null for tests
server/services/supabase-file-storage.ts# Fase 3: tenant prefix in paths
server/routes/tenant-users.ts           # Fase 4: user limit check
server/utils/tier-limits.ts             # Fase 4: NIEUW - limit helpers
```

---

## VORIGE SESSIE: Sidebar & Settings UX Improvements (6 Feb 2026)

### Sidebar Restructure - COMPLETED ✅

Geherstructureerd van geneste collapsibles naar flat structure voor betere UX.

**Voor:**
```
CONFIGURATIE (gray label)
└── Instellingen (collapsible)
    ├── Opnemen (nested collapsible)
    ├── Verwerken (nested collapsible)
    └── ...
```

**Na:**
```
INSTELLINGEN (gray label)
├── Opnemen (collapsible met Video icon)
├── Verwerken (collapsible met Sparkles icon)
├── Organisatie (collapsible met Building2 icon)
└── Account (collapsible met CreditCard icon)
```

**Voordelen:**
- Geen dubbele klik meer nodig (was: Instellingen → Sectie → Item)
- Direct zichtbaar welke categorieën er zijn
- Icons maken het makkelijker te scannen
- Collapsed sidebar toont nog steeds single "Instellingen" icon

### Settings Page Layout Consistency - COMPLETED ✅

Settings pagina layout consistent gemaakt met andere pagina's (dashboard, admin, meetings).

**Wijzigingen:**
- Padding gewijzigd van `max-w-4xl mx-auto p-6 space-y-8` naar `p-6 space-y-6`
- Verwijderd: ~300 regels code met onnodige nested Cards binnen accordion sections
- Resultaat: Minder wit-op-wit nesting, schonere visuele hiërarchie

### Deep Linking to Settings Sections - COMPLETED ✅

Sidebar items navigeren nu direct naar de juiste sectie op de settings pagina.

**Implementatie:**
- `scrollToSection()` functie dispatch custom event
- Settings pagina luistert naar `settingsNavigate` event
- Accordion opent automatisch + scroll naar sectie
- URL hash wordt bijgewerkt (bijv. `/settings#meeting-bot`)
- Active state in sidebar volgt scroll positie via IntersectionObserver

### Visual Contrast Improvements - COMPLETED ✅

CSS variabelen aangepast voor betere contrast:
- `--border`: `0 0% 91%` → `0 0% 89%`
- `--card-border`: `0 0% 95%` → `0 0% 89%`

### Commits
```
dfe3829 Restructure sidebar: flat settings groups under INSTELLINGEN label
96ec593 Add collapsible settings groups in sidebar
bfc3071 Make settings page layout consistent with other pages
0eac0d6 Remove nested Cards from settings accordion sections
79afba9 Improve card contrast with darker background and borders
84a0abf Fix sidebar deep linking to settings sections
```

### Speaker Diarization Fix - COMPLETED ✅

Recall.ai transcripts bevatten nu speaker namen. Het probleem was dat onze code zocht naar `seg.speaker` terwijl Recall.ai `seg.participant.name` gebruikt.

**Fix in `transformTranscriptFromApi()`:**
```typescript
// Oud (werkte niet)
speakerName: seg.speaker || "Unknown"

// Nieuw (werkt met Recall.ai format)
speakerName: seg.participant?.name || seg.speaker || seg.speaker_name || "Unknown"
```

**Recall.ai transcript formaat:**
```json
{
  "participant": { "id": 100, "name": "Jan de Vries", "is_host": true },
  "words": [{ "text": "Welkom", "start_timestamp": { "relative": 0.5 } }]
}
```

### Files Changed
```
client/src/components/app-sidebar.tsx   # Sidebar restructure, deep linking
client/src/pages/settings.tsx           # Layout consistency, Card removal
client/src/index.css                    # Border contrast improvements
server/services/recall.ts               # Speaker diarization parsing fix
```

---

## VORIGE SESSIE: Quality Review & Critical Fixes (5 Feb 2026)

### Quality Review met Agent Teams

Parallelle quality review uitgevoerd met drie gespecialiseerde agents:
- **Tenant Guardian** - Multi-tenant security review
- **Pipeline Inspector** - Processing pipeline robustness review
- **Commerce Auditor** - Stripe integration review

**Resultaat:** 3 CRITICAL, 12 HIGH, 13 MEDIUM, 10 LOW issues gevonden.
Rapporten in `/reviews/` directory.

### CRITICAL Fixes Geïmplementeerd ✅

#### 1. Database-backed Pipeline Queue
**Probleem:** In-memory queue verloor meetings bij server restart.

**Oplossing:**
- Nieuwe `pipeline_queue` tabel (`migrations/0014_pipeline_queue.sql`)
- Queue persistent in database met status tracking
- Recovery van 'processing' items bij startup
- `FOR UPDATE SKIP LOCKED` voor safe concurrent access

**Bestanden:**
- `migrations/0014_pipeline_queue.sql` (nieuw)
- `server/services/pipelineQueue.ts` (herschreven)

#### 2. Webhook Cache Size Limit
**Probleem:** Unbounded Map kon memory exhaustion veroorzaken.

**Oplossing:**
- `WEBHOOK_CACHE_MAX_SIZE = 10000` toegevoegd
- LRU-style eviction (verwijdert 10% bij max)
- Voorkomt DoS via webhook replay attacks

**Bestand:** `server/routes/recall-webhook.ts`

#### 3. Stripe Webhook Idempotency
**Probleem:** Geen deduplicatie - duplicate welcome emails mogelijk.

**Oplossing:**
- Event ID deduplication toegevoegd (zoals Recall webhook)
- `constructAndValidateEvent()` voor ID extractie vóór processing
- Zelfde size limit pattern als Recall webhook

**Bestanden:**
- `server/routes/stripe-webhook.ts`
- `server/services/stripe.ts`

### HIGH Priority Fixes Geïmplementeerd ✅

#### H5: Exponential Backoff voor Transcript Retry
**Probleem:** Single 5s retry bij transcript fetch was onvoldoende.

**Oplossing:**
- 3 retries met exponential backoff (5s, 10s, 20s)
- `TRANSCRIPT_RETRY_DELAYS = [5000, 10000, 20000]`
- Betere logging bij elke retry poging

**Bestand:** `server/routes/recall-webhook.ts`

#### H7: API Timeouts voor Recall.ai
**Probleem:** Geen timeouts op externe API calls - kon hangen.

**Oplossing:**
- `fetchWithTimeout()` helper met AbortController
- Default timeout: 30s (API calls)
- Transcript download: 60s
- Image upload: 10s

**Bestand:** `server/services/recall.ts`

#### H10: Welcome Email Idempotency
**Probleem:** Welcome email kon meerdere keren verzonden worden bij webhook retry.

**Oplossing:**
- `welcome_email_sent_at` kolom in `stripe_subscriptions`
- Check voordat email wordt verzonden
- Update na succesvolle verzending
- Nieuwe migration: `0015_welcome_email_tracking.sql`

**Bestanden:**
- `migrations/0015_welcome_email_tracking.sql` (nieuw)
- `shared/schema.ts`
- `server/services/stripe.ts`
- `server/supabase-storage.ts`
- `server/storage.ts`

---

## VORIGE SESSIE: Recall.ai Pipeline Volledig Werkend (5 Feb 2026)

### 🎉 END-TO-END PIPELINE WERKEND!

De Recall.ai meeting bot integratie is nu volledig werkend. Meetings worden:
1. Opgenomen door Recall.ai bot
2. Transcript ontvangen via webhook
3. Verwerkt door AI pipeline (categorisatie, notities, documenten)
4. Getoond in dashboard

### Fixes in Deze Sessie

#### 1. Webhook Transcript Processing - FIXED ✅

**Probleem:** Webhook retourneerde `newStatus: "failed"` voor `transcript.done` events.

**Fixes:**
- Metadata extractie gecorrigeerd: `payload.data?.bot?.metadata?.tenantId` (niet `payload.data?.metadata`)
- `owner: "unassigned"` toegevoegd aan meeting creation
- Pipeline steps worden nu aangemaakt in webhook handler
- Status mapping uitgebreid voor transcript events

#### 2. JSONB Storage - FIXED ✅

**Probleem:** `participants.map is not a function` error in pipeline.

**Oorzaak:** PostgreSQL JSONB velden werden opgeslagen met `JSON.stringify()` wat resulteerde in dubbel-escaped strings.

**Fix:** Gebruik `sql.json()` voor JSONB kolommen:
```typescript
// FOUT
${JSON.stringify(dbMeeting.participants || [])}

// CORRECT
${sql.json(dbMeeting.participants || [])}
```

**Bestanden:** `server/supabase-storage.ts`

#### 3. Array.isArray Defensive Checks - FIXED ✅

**Probleem:** Frontend crashed met "Kt.filter is not a function" wanneer API arrays als objects retourneerde.

**Fix:** `Array.isArray()` checks toegevoegd in alle componenten:
```typescript
const safeMeetings = Array.isArray(meetings) ? meetings : [];
```

**Bestanden:**
- `client/src/pages/dashboard.tsx`
- `client/src/pages/meetings.tsx`
- `client/src/pages/review-queue.tsx`
- `client/src/components/recent-errors.tsx`
- `client/src/components/meetings-table.tsx`
- `client/src/components/dashboard-filter-context.tsx`
- `client/src/components/task-review-card.tsx`
- `server/services/pipeline.ts`

#### 4. Transcript Loading for Skipped Stages - FIXED ✅

**Probleem:** Pipeline faalde bij categorisatie met "No transcript available" omdat `fetch_transcript` was overgeslagen maar transcript niet in context geladen.

**Fix:** Laad transcript in context wanneer stage wordt overgeslagen:
```typescript
if (step.status === "completed") {
  if (stage === "fetch_transcript" && !context.transcript && context.meeting.transcript) {
    context.transcript = context.meeting.transcript;
  }
  return;
}
```

**Bestand:** `server/services/pipeline.ts`

#### 5. Scheduler Logging Cleanup - FIXED ✅

**Probleem:** Railway logs toonden rode errors voor tenants zonder MS365 OAuth - dit is een verwachte conditie, geen error.

**Fix:** Verwachte condities worden nu als info gelogd:
```typescript
if (message.includes("OAuth") || message.includes("niet gevonden") || message.includes("not configured")) {
  console.log(`[RecallScheduler] Skipping tenant ${tenantId}: ${message}`);
} else {
  console.error(`[RecallScheduler] Error processing tenant ${tenantId}:`, error);
}
```

**Bestand:** `server/services/recall-scheduler.ts`

#### 6. Webhook Deduplication Cache Clear - ADDED ✅

Voor development/testing: endpoint om webhook cache te clearen:
```
POST /api/webhook/recall/clear-cache
```
Alleen beschikbaar in non-production omgevingen.

**Bestand:** `server/routes/recall-webhook.ts`

### Commits

```
833cfb5 Improve scheduler logging - don't log expected conditions as errors
0ac482d Add endpoint to clear webhook deduplication cache (dev only)
141eac1 Add debug logging for Recall.ai transcript parsing
d6fc7ce Load transcript into context when fetch_transcript stage is skipped
2046e6c Skip fetch_transcript if meeting already has transcript
```

---

## VORIGE SESSIE: Security Audit & Bot Fixes (3 Feb 2026 - Late Night)

### Security Audit - COMPLETED

Uitgebreide security audit van de Recall.ai bot implementatie uitgevoerd.

#### CRITICAL: Webhook Signature Verification - FIXED ✅

**Probleem:** Webhook signature verificatie was niet correct geïmplementeerd:
- Code checkte alleen of header begon met "v1," zonder echte HMAC verificatie
- `req.body` werd gebruikt ipv `req.rawBody` (parsed JSON vs originele bytes)

**Fix:**
1. Echte HMAC-SHA256 verificatie geïmplementeerd (Svix format)
2. Gebruikt nu `req.rawBody` voor correcte signature berekening
3. Timestamp validatie (max 5 minuten oud)
4. Security event logging voor failed verifications

```typescript
// server/routes/recall-webhook.ts
const rawBody = (req as any).rawBody;
const rawBodyString = rawBody.toString("utf8");
const verification = verifyWebhookSignature(rawBodyString, req.headers, webhookSecret);
```

#### HIGH: Production Mock Mode Check - FIXED ✅

**Probleem:** Als `RECALL_API_KEY` niet was ingesteld, draaide de service stil in mock mode - zelfs in productie.

**Fix:** In productie wordt nu duidelijk gelogd en `isConfigured()` retourneert `false`:
```typescript
if (this.mockMode && process.env.NODE_ENV === "production") {
  console.error("RecallService: CRITICAL - Running in MOCK mode in PRODUCTION!");
  return false;
}
```

#### MEDIUM: Tenant Verification in Webhook - ADDED ✅

Webhook verifieert nu dat de tenant in de metadata overeenkomt met de recording:
```typescript
const metadataTenantId = payload.bot?.metadata?.tenantId;
if (metadataTenantId && metadataTenantId !== recording.tenantId) {
  return res.status(400).json({ error: "Tenant mismatch" });
}
```

#### Audit Event Added

Nieuwe security event: `WEBHOOK_SIGNATURE_FAILED` in `server/utils/audit.ts`

### Bot Creation Fix - Recall API Timeout Minimums ✅

**Probleem:** Recall API retourneerde 400 error:
```json
{"automatic_leave":{"bot_detection":{
  "using_participant_events":{"timeout":["Ensure this value is greater than or equal to 10."]},
  "using_participant_names":{"timeout":["Ensure this value is greater than or equal to 10."]}
}}}
```

**Oorzaak:** Bot detection timeouts waren te laag ingesteld (2-5 seconden).

**Fix:** Alle timeouts verhoogd naar minimum 10 seconden:

| Setting | Was | Nu |
|---------|-----|-----|
| `everyone_left_timeout` | 2s | 10s |
| `using_participant_names.timeout` | 2s | 10s |
| `using_participant_events.timeout` | 5s | 10s |

### Bot Join Time UI Setting - ADDED ✅

UI control toegevoegd in Settings → Bot Instellingen voor `botJoinLeadTimeMinutes`:
- Input veld met range 0-15 minuten
- Default: 5 minuten
- Label: "Bot join tijd"
- Beschrijving: "minuten voor de meeting"

**Let op:** Vercel moet correct deployen voor UI wijzigingen zichtbaar zijn.

### Files Gewijzigd

```
server/services/recall.ts            # Timeout fixes (min 10s)
server/routes/recall-webhook.ts      # Signature verification, tenant check
server/utils/audit.ts                # WEBHOOK_SIGNATURE_FAILED event
client/src/pages/settings.tsx        # Bot join time UI setting
```

---

## Git Branch Strategie

### Branches
| Branch | Omgeving | Auto-deploy |
|--------|----------|-------------|
| `main` | **Productie** | Vercel (frontend), Railway (backend) |
| `dev` | **Staging** | Vercel (frontend), Railway (backend) |

### Hotfix Workflow (Productie Bug Fixen)

Gebruik deze workflow wanneer productie kapot is maar `dev` nog niet klaar is voor deploy.

```
VOOR:
main (prod)  ──●─────────────────── (broken!)
               \
dev             ●───●───●───●───── (veel nieuwe features, niet klaar)

NA:
main (prod)  ──●────────●───────── (fixed!)
               \       /
                ●─────  ← hotfix branch (alleen de fix)
                 \
dev               ●───●───●───●─── (krijgt fix ook)
```

**Stappen:**

```bash
# 1. Start vanaf main (productie code)
git checkout main
git pull origin main

# 2. Maak hotfix branch
git checkout -b hotfix/beschrijving-van-fix

# 3. Maak je fix (minimaal, alleen wat nodig is)
# ... edit files ...

# 4. Commit
git add <files>
git commit -m "Fix: beschrijving van de fix"

# 5. Merge naar main en push (deploy naar productie)
git checkout main
git merge hotfix/beschrijving-van-fix --no-ff -m "Merge hotfix: beschrijving"
git push origin main

# 6. Merge hotfix ook naar dev (zodat dev de fix ook heeft)
git checkout dev
git merge hotfix/beschrijving-van-fix --no-ff -m "Merge hotfix from main"
git push origin dev

# 7. Cleanup (optioneel)
git branch -d hotfix/beschrijving-van-fix
```

**Belangrijke regels:**
- Hotfix branch ALTIJD vanaf `main` maken, NOOIT vanaf `dev`
- Alleen minimale changes - fix het probleem, niets meer
- Na merge naar `main`, ALTIJD ook mergen naar `dev`
- Test op productie na deploy (hard refresh: Ctrl+Shift+R)

### Recente Hotfixes

| Datum | Branch | Probleem | Fix |
|-------|--------|----------|-----|
| 2026-02-03 | `hotfix/blank-screen-array-error` | Blank screen + 500 error `/api/stats` | `Array.isArray()` checks (zie details hieronder) |

**Details hotfix 2026-02-03:**
- **Root cause:** Sommige meetings in database hadden `action_items` als object `{}` ipv array `[]`
- **Symptomen:** 500 error op `/api/stats`, blank screen na login
- **Files gefixed:**
  - `client/src/pages/dashboard.tsx` - refetchInterval callback
  - `client/src/pages/meetings.tsx` - refetchInterval callback
  - `client/src/pages/settings.tsx` - people filter
  - `client/src/components/quick-actions/review-queue.tsx` - actionItems filters
  - `server/storage.ts` - getDashboardStats()
  - `server/supabase-storage.ts` - getDashboardStats()

---

## VOLGENDE STAPPEN

### Fase 1: Critical Fixes ✅ DONE (5 Feb 2026)

- [x] **Database-backed pipeline queue** - Meetings overleven server restart
- [x] **Webhook cache size limits** - Voorkomt memory exhaustion
- [x] **Stripe webhook idempotency** - Voorkomt duplicate emails

### Fase 2: HIGH Priority Fixes ✅ MOSTLY DONE (9 Feb 2026)

Zie `/reviews/FINAL-QUALITY-REPORT.md` voor complete lijst.

**Afgerond:**
- [x] **H1: Make tenantId required** - 19 methods, 40+ callers (9 Feb 2026)
- [x] **H2: File storage tenant isolation** - Tenant prefix in file paths (9 Feb 2026)
- [x] **H5: Exponential backoff voor transcript retry** - 3 retries (5s, 10s, 20s)
- [x] **H7: API timeouts** - Toegevoegd aan alle Recall.ai API calls
- [x] **H8: Tier limit enforcement** - meetings/maand + users/tenant (9 Feb 2026)
- [x] **H10: Welcome email idempotency** - `welcome_email_sent_at` tracking
- [x] **Inzichten Agent fix** - `services: null as any` → real services (9 Feb 2026)

**Nog te doen:**
- [ ] **H3: Retry bij transient failures** - Pipeline stage retry logic

### Fase 3: Agent Capabilities ✅ DONE (9 Feb 2026)

- [x] **5 nieuwe agent mutations** - schedule_recording, create_outlook_draft, generate_document, upload_sharepoint, create_tasks
- [x] **Dynamische popover** - Frontend toont beschikbare acties op basis van API
- [x] **Bugfixes** - getCapabilityByName lookup, getAgentCapabilities filter, search filter

### Track A: Security & Reliability Hardening ✅ DONE (10 Feb 2026)

17 fixes in 3 fasen:
- **Fase 1 (CRITICAL)**: Session fixation, MFA crypto, encryption salt, MFA rate limit, CSRF hardening
- **Fase 2 (HIGH)**: Graceful shutdown, connection pool, token O(1) lookup, persistent webhook dedup, seed hardening
- **Fase 3 (MEDIUM)**: Account lockout, session timeout, impersonation timeout, tenant switch check, Zod max constraints, DB indexes, Express typing

4 nieuwe migraties + 69 tests. Self-review passed.

### Track B: B2B-SAAS-SKILLS.md Update ✅ DONE (10 Feb 2026)

23 edits doorgevoerd in B2B-SAAS-SKILLS.md (v2.1):
- **12 code fixes**: session.regenerate (3x), crypto.randomInt (2x), encryption salt, CSRF localStorage, seed script, Stripe SDK v20, rawBody typing, Zod .max(), tenant switch isActive
- **7 nieuwe secties**: MFA rate limiter, account lockout, absolute session timeout, impersonation auto-expire, L2 webhook dedup, graceful shutdown, performance indexes
- **4 doc verbeteringen**: getBillingStatus berekening, security checklist, env vars, versienummer
- **Bonus**: `as any` casts verwijderd uit middleware voorbeelden (req.session.user pattern)

Verificatie: `Math.random` → 0, `admin123` → 0, hardcoded `"salt"` → 0, `as any` → 1 (comment only)

### Fase 4: Testing & Stabilisatie

- [ ] **Run migrations** - `0017_token_prefix.sql` + `0018_webhook_dedup.sql` + `0019_account_lockout.sql` + `0020_performance_indexes.sql`
- [ ] **Battle test Recall.ai bot** - Uitgebreid testen in echte meetings
- [ ] **Staging test checklist** - Doorlopen van `TESTING-CHECKLIST.md`

### Fase 5: UI/UX Verbeteringen ✅ PARTIALLY DONE (6 Feb 2026)

- [ ] **UI "Geplande Meetings"** - Verbeter user experience
- [ ] **Dashboard wizard voor bot activity** - Status indicators, quick actions
- [x] **Settings navigatie herstructureren** - Flat collapsible groups met icons
- [x] **Sidebar deep linking** - Klik op sectie → scroll naar juiste plek
- [x] **Card nesting opgeschoond** - Verwijderd onnodige nested Cards

### Fase 6: Production Release

- [ ] **Productie deploy** - Merge `dev` naar `main` na bovenstaande
- [ ] **Monitoring** - Railway logs voor edge cases

---

## LAATSTE SESSIE: Duplicate Bot & Auto-Leave Fixes (3 Feb 2026 - Evening)

### Issue 1: Twee Bots in Dezelfde Meeting

**Probleem:** Wanneer twee verschillende calendar events dezelfde meeting URL delen (bijv. terugkerende Teams meetings), werden er twee bots gepland die beide de meeting joinden.

**Root cause:** Deduplicatie check keek alleen naar `calendar_event_id`, niet naar `meeting_url`.

**Fix:**
1. Nieuwe storage method `getActiveRecordingByMeetingUrl()` toegevoegd
2. `/api/recordings/toggle-bot/:eventId` - checkt URL voordat bot wordt aangemaakt
3. `/api/recordings/refresh` - skipt events die URL delen met bestaande bot
4. `scheduleRecordingForEvent()` in scheduler - zelfde check

**Resultaat:** Bij duplicate URL krijgt gebruiker melding: "Er is al een notulist gepland voor deze meeting"

### Issue 2: Bot Verliet Meeting Niet

**Probleem:** Wanneer alle deelnemers vertrokken, bleef de bot in de meeting. Meeting timer ging door.

**Root cause:** Met twee bots zag elke bot de andere als "deelnemer", waardoor `everyone_left_timeout` nooit triggerde. Ook waren de timeouts te lang.

**Fix - Auto-leave configuratie getuned:**

| Setting | Oud | Nieuw |
|---------|-----|-------|
| `everyone_left_timeout` | 30s | 10s |
| `using_participant_names.activate_after` | 60s | 30s |
| `using_participant_names.timeout` | 30s | 10s |
| `using_participant_events.activate_after` | 120s | 60s |
| `using_participant_events.timeout` | 60s | 30s |

Extra bot namen toegevoegd: "schakel", "mia", "transcri", "meeting bot", "ai bot"

### Files Gewijzigd
```
server/storage.ts                    # getActiveRecordingByMeetingUrl() interface
server/supabase-storage.ts           # getActiveRecordingByMeetingUrl() implementation
server/routes/recordings.ts          # URL deduplication in toggle-bot en refresh
server/services/recall-scheduler.ts  # URL deduplication in scheduleRecordingForEvent
server/services/recall.ts            # Auto-leave timing tuning
```

---

## Bug Fixes & UX Verbeteringen (3 Feb 2026 - Afternoon)

### Bugs Gefixed

**1. Badge counts updaten niet na toggle**
- Oorzaak: Counts kwamen van API response, niet lokaal berekend
- Fix: `totalMeetings` en `meetingsWithNotulist` worden nu lokaal berekend uit events
- Werkt nu direct na toggle zonder page refresh

**2. Delete werkt niet voor opnames zonder recallBotId**
- Oorzaak: Cache invalidatie matchte niet de juiste query keys
- Fix: Predicate-based invalidation voor alle `/api/recordings` queries

**3. Recall API error 405 bij verwijderen**
- Oorzaak: Recall kan alleen "scheduled" bots verwijderen, niet bots die al joined hebben
- Fix: Backend checkt nu status voordat Recall delete wordt geprobeerd
- Voor failed/completed: alleen lokale delete, met melding dat origineel bewaard blijft

### Terminologie Aangepast

| Was | Wordt |
|-----|-------|
| "Verwijderen" | "Notulist verwijderen" |
| "+ Notulist" | "Notulist toevoegen" |
| "bot" | "AI notulist" |
| "Recall.ai" | (verwijderd uit UI) |
| "bots gepland" | "notulist(en) gepland" |

### Delete Dialog Verbeterd

**Voor "scheduled" opnames (kunnen geannuleerd worden):**
```
┌─────────────────────────────────────────┐
│ Kies wat je wilt verwijderen:           │
│                                         │
│ ● Alleen uit dit overzicht              │
│   De geplande opname blijft staan       │
│                                         │
│ ○ Volledig annuleren                    │
│   Annuleert ook de geplande opname      │
└─────────────────────────────────────────┘
```

**Voor failed/completed opnames:**
- Simpele bevestiging
- Melding: "De originele opname blijft bewaard op de server"

### Warning Banner Verwijderd
- De verwarrende "X meeting(s) zijn gepland maar hebben GEEN Recall bot" melding is weg

---

## VOLTOOID: Unified Meetings Page + Settings Cleanup ✅

### Implementatie Compleet (2 Feb 2026)

De Meetings pagina heeft nu twee views:
1. **"Verwerkte meetings"** - Bestaande functionaliteit (100% intact)
2. **"Geplande meetings"** - Nieuwe view voor scheduled recordings

### Files
```
client/src/pages/meetings.tsx                    # View toggle toegevoegd
client/src/components/scheduled-meetings-view.tsx # NIEUW - geplande meetings component
server/routes/recordings.ts                       # Delete logic verbeterd
```

### Scheduled Meetings View Features

| Feature | Status | Beschrijving |
|---------|--------|--------------|
| Kalender events | ✅ | Toon komende meetings van Microsoft Calendar |
| Notulist toggle | ✅ | Toevoegen/verwijderen van AI notulist per meeting |
| Per-event loading | ✅ | Loading state per meeting (niet globaal) |
| Real-time UI update | ✅ | Directe cache invalidatie via predicates |
| "Geen notulist" badge | ✅ | Amber badge + achtergrond voor meetings zonder notulist |
| Live status | ✅ | Gepland → Opnemen → Verwerken (met animatie) |
| Auto-refresh | ✅ | 5s polling tijdens actieve opnames, 60s anders |
| Mislukte opnames | ✅ | Collapsible sectie (default ingeklapt) |
| Delete dialog | ✅ | Radio-button stijl keuze, status-afhankelijke opties |

### Settings Pagina Opgeschoond ✅
Verwijderd uit Settings → Meeting Planning:
- "Komende Meetings" sectie (nu in Meetings → Geplande meetings)
- "Opname Statistieken" sectie
- "Opname Geschiedenis" sectie
- Delete recording dialog + alle gerelateerde code

Toegevoegd: Hint kaart die verwijst naar Meetings pagina voor scheduling.

**-872 regels code verwijderd** uit `settings.tsx`

---

## KRITIEK: Recall.ai EU Region

**We gebruiken `RECALL_REGION=eu-central-1` (EU regio)!**

Sommige API endpoints werken anders in de EU regio:

| Feature | EU Status | Oplossing |
|---------|-----------|-----------|
| `transcription_options` in bot create | **NIET ONDERSTEUND** | Niet meesturen |
| `/bot/{id}/transcript` endpoint | **DEPRECATED** | Gebruik bot recordings method |
| Bot avatar (`bot_image`) | Werkt | - |
| Bot detection (auto-leave) | Werkt | - |

**Transcript ophalen in EU:**
1. GET `/bot/{id}` → haal bot details op
2. Pak `recordings[].media_shortcuts.transcript.data.download_url`
3. Download transcript van die URL

**Implementatie:** Zie `server/services/recall.ts` - `getTranscriptViaBot()` methode

---

## Vandaag Gedaan (2 Feb 2026 - Late Night Session)

### Simplified Meeting Recordings UI - COMPLETED
Removed confusing "Microsoft 365 Agenda" and "Automatische Scheduler" sections in favor of a single unified list.

**New UI:**
- Single "Komende Meetings" list showing all upcoming calendar meetings
- Toggle button per meeting to add/remove bot
- Auto-join setting determines default behavior

**New API Endpoints:**
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/recordings/toggle-bot/:eventId` | POST | Toggle bot on/off for a meeting |
| `/api/recordings/refresh` | POST | Combined calendar fetch + bot scheduling |

**New Response Fields in `/api/recordings/calendar-events`:**
- `hasBotId`: Whether meeting has bot scheduled
- `withBotCount`: Total meetings with bots
- `scheduledButNoBotCount`: Scheduled recordings without bot IDs
- `warning`: Optional sync warning message

**Auto-Join Behavior:**
- **Auto-join ON**: All meetings get bot automatically; user can remove specific ones
- **Auto-join OFF**: No meetings get bot by default; user can add specific ones
- Users can toggle bots on/off unlimited times for any meeting

**Files Changed:**
```
client/src/components/upcoming-recordings.tsx  # Simplified UI with toggle buttons
server/routes/recordings.ts                    # New endpoints
```

---

## Vandaag Gedaan (2 Feb 2026 - Night Session)

### CRITICAL: Pipeline Trigger for Recall Transcripts - COMPLETED
The meeting was being created from Recall transcripts but **the pipeline was never triggered**.

**Fix:** Added `pipelineQueue.enqueue(meeting.id)` to `recall-webhook.ts` after meeting creation.

```typescript
// server/routes/recall-webhook.ts
// After creating meeting from transcript:
console.log(`[Recall Webhook] Enqueueing meeting ${meeting.id} to pipeline for processing`);
pipelineQueue.enqueue(meeting.id);
```

**This enables the full flow:**
1. Recall bot records meeting
2. `transcript.done` webhook received
3. Meeting created from transcript
4. **Pipeline triggered** → AI summary, documents, SharePoint, Outlook, tasks

### Recall Recording Import Endpoints - NEW
Added endpoints similar to Fireflies import for manual processing.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/recordings/importable` | GET | List recordings that can be imported |
| `/api/recordings/import` | POST | Batch import recordings to meetings |
| `/api/recordings/:id/reprocess` | POST | Reprocess a single recording |

### Microsoft Graph Timezone Fix - COMPLETED
Calendar times were off by timezone offset (e.g., 09:30 showing as 10:30).

**Root cause:** Microsoft Graph returns `dateTime` without timezone info, timezone in separate field.

**Fix:** Added `parseMsGraphDateTime()` function that properly interprets the datetime in the calendar's timezone.

```typescript
// Before (wrong)
start: new Date(event.start.dateTime)  // Interpreted as UTC

// After (correct)
start: parseMsGraphDateTime(event.start.dateTime, event.start.timeZone)
```

### UI Improvements - COMPLETED
- Added **delete buttons** (trash icon) for scheduled/failed recordings
- Removed incorrect "Bekijk alle x opnames" link that went to /meetings
- Delete mutation with proper cache invalidation

### Bot ID Storage Fix - COMPLETED
Bot ID wasn't being saved due to JSON.parse error on metadata object.

**Fix:** Handle both string and object metadata in `transformBotFromApi()`.

---

## Vandaag Gedaan (1 Feb 2026 - Late Night Session)

### Recall.ai EU Region Support - COMPLETED
Recall.ai API key was for EU region (eu-central-1) but code was using US endpoint.

**Fix:** Added `RECALL_REGION` environment variable support.

```typescript
// server/services/recall.ts
const region = process.env.RECALL_REGION;
if (region && ["us-east-1", "us-west-2", "eu-central-1", "ap-northeast-1"].includes(region)) {
  this.baseUrl = `https://${region}.recall.ai/api/v1`;
}
```

**Railway config required:**
```
RECALL_REGION=eu-central-1
```

### Recall.ai API Field Fixes - IN PROGRESS
Multiple API fields were causing 400 errors. Systematically removed undocumented/unsupported fields.

| Field | Issue | Fix |
|-------|-------|-----|
| `metadata` | Sent as string instead of object | Fixed: removed JSON.stringify |
| `recording_mode` | Not a valid API field | Removed |
| `bot_image` | Not in API docs | Removed |
| `transcription_options` | Wrong structure | Removed |

**Current simplified payload:**
```json
{
  "meeting_url": "https://teams.microsoft.com/...",
  "bot_name": "Schakel AI Notulist",
  "join_at": "2026-02-02T09:25:00.000Z",
  "automatic_leave": {
    "waiting_room_timeout": 600,
    "noone_joined_timeout": 600,
    "everyone_left_timeout": 120
  }
}
```

### Timezone Display Fix - COMPLETED
Times were showing +1 hour off (e.g., 09:30 meeting showing as 10:30).

**Root cause:** Double timezone conversion in `date-fns-tz`.

```typescript
// WRONG - double conversion
const zonedDate = toZonedTime(dateObj, timezone);
return formatTz(zonedDate, formatStr, { timeZone: timezone });

// CORRECT - single conversion
return formatInTimeZone(dateObj, timezone, formatStr, { locale: nl });
```

**Dynamic timezone detection added:**
```typescript
function getBrowserTimezone(): string {
  try {
    return Intl.DateTimeFormat().resolvedOptions().timeZone || "Europe/Amsterdam";
  } catch {
    return "Europe/Amsterdam";
  }
}
```

### Cache Invalidation Fix - COMPLETED
"Geplande opnames" wasn't refreshing after sync because query key mismatch.

**Problem:** Query key was `["/api/recordings/upcoming?hours=48"]` but invalidation used `["/api/recordings/upcoming"]`.

**Fix:** Use predicate-based invalidation:
```typescript
queryClient.invalidateQueries({ predicate: (query) =>
  typeof query.queryKey[0] === 'string' &&
  query.queryKey[0].startsWith("/api/recordings/upcoming")
});
```

### Bot Early Join Feature - COMPLETED
Added setting for bot to join X minutes before meeting starts.

**New setting:** `botJoinLeadTimeMinutes` (default: 5, range: 0-15)

```typescript
const leadTimeMinutes = settings.botJoinLeadTimeMinutes ?? 5;
const botJoinTime = new Date(meetingStartDate.getTime() - leadTimeMinutes * 60 * 1000);
```

### Database Aggregation Fix - COMPLETED
LLM usage stats failing with `function sum(text) does not exist`.

**Fix:** Cast text column to numeric before SUM:
```sql
COALESCE(SUM(total_cost_usd::numeric), 0)::numeric as total_cost_usd
```

### Route Ordering Fix - COMPLETED
`/api/recordings/status` returning 404 because Express matched `/:id` first.

**Fix:** Move specific routes before parameterized routes:
```typescript
// CORRECT ORDER
router.get("/api/recordings/status", ...);    // Specific first
router.get("/api/recordings/upcoming", ...);
router.get("/api/recordings/history", ...);
router.get("/api/recordings/:id", ...);       // Parameterized last
```

### Welcome Email Update - COMPLETED
Removed "Eerste stappen" section since users don't configure Recall.ai themselves.

---

## Recall.ai Meeting Bot - STATUS

### Implementatie: ✅ VOLLEDIG COMPLEET EN WERKEND

| Component | Status | Bestand |
|-----------|--------|---------|
| Database schema | ✅ | `migrations/0012_recall_integration.sql` |
| Type definitions | ✅ | `shared/schema.ts` |
| Core service | ✅ | `server/services/recall.ts` |
| Scheduler (polling) | ✅ | `server/services/recall-scheduler.ts` |
| Webhook handler | ✅ | `server/routes/recall-webhook.ts` |
| API routes | ✅ | `server/routes/recordings.ts` |
| Storage layer | ✅ | `server/supabase-storage.ts` |
| Frontend widget | ✅ | `client/src/components/upcoming-recordings.tsx` |
| Manual scheduler | ✅ | `client/src/components/manual-bot-scheduler.tsx` |
| Pipeline integration | ✅ | `server/services/pipeline.ts` |

### End-to-End Testing: ✅ VOLLEDIG WERKEND (5 Feb 2026)
- [x] RECALL_API_KEY in Railway
- [x] RECALL_REGION=eu-central-1 in Railway
- [x] RECALL_WEBHOOK_SECRET in Railway
- [x] Webhook URL in Recall.ai dashboard
- [x] `scheduled_recordings` tabel aangemaakt
- [x] Bot creation API call succeeds
- [x] Bot joins meeting correctly
- [x] Transcript webhook received and processed
- [x] Meeting created from transcript
- [x] AI pipeline processes meeting (categorization, notes, documents)
- [x] Meeting appears in dashboard

### Recall.ai Regional Endpoints
```
US (default): https://api.recall.ai/api/v1
US East:      https://us-east-1.recall.ai/api/v1
US West:      https://us-west-2.recall.ai/api/v1
EU Frankfurt: https://eu-central-1.recall.ai/api/v1
Asia Tokyo:   https://ap-northeast-1.recall.ai/api/v1
```

### Webhook URL
```
Staging: https://map-api-dev.schakel.ai/api/webhook/recall
Production: https://map-api.schakel.ai/api/webhook/recall
```

---

## Known Issues / Debugging

### Timezone Display
If times still appear incorrect after deployment:
1. Hard refresh (Ctrl+Shift+R) to get latest frontend
2. Check browser timezone: `Intl.DateTimeFormat().resolvedOptions().timeZone`
3. Server stores times in UTC, client converts to local

### Old Meetings Still Showing
The `/api/recordings/upcoming` endpoint only returns future meetings. If old ones show:
1. Hard refresh the browser
2. Click "Synchroniseer" to invalidate cache
3. Meetings deleted from Outlook remain in database until cleaned up

### Bot Not Joining
Check Railway logs for:
```
RecallService: Creating bot with payload: {...}
Recall API error creating bot: 400 {...}
```

Common issues:
- Wrong region → Add `RECALL_REGION` env var
- Invalid field → Check API docs, remove unsupported fields
- Meeting URL format → Must be full URL with protocol
- **Timeout values too low** → All `bot_detection` timeouts must be ≥10 seconds
- Mock mode in production → Check if `RECALL_API_KEY` is set

### Webhook Signature Failures
Check for `[Recall Webhook] Signature verification failed` in logs.
- Verify `RECALL_WEBHOOK_SECRET` matches Recall.ai dashboard
- Secret format should be `whsec_...` (base64 encoded)
- Timestamps must be within 5 minutes

---

## Stripe Configuratie - STATUS

### Staging: ✅ VOLLEDIG WERKEND
- [x] STRIPE_SECRET_KEY (live mode)
- [x] STRIPE_WEBHOOK_SECRET
- [x] Webhook URL: `https://map-api-dev.schakel.ai/api/webhook/stripe`
- [x] Checkout flow getest en werkend
- [x] Welcome modal toont na checkout

---

## Environment Variables Checklist (Railway)

### Recall.ai
- [x] `RECALL_API_KEY`
- [x] `RECALL_REGION` (eu-central-1 for Frankfurt)
- [x] `RECALL_WEBHOOK_SECRET`

### Billing (Stripe)
- [x] `STRIPE_SECRET_KEY`
- [x] `STRIPE_WEBHOOK_SECRET`
- [x] `STRIPE_PRICE_STARTER_MONTHLY`
- [x] `STRIPE_PRICE_STARTER_ANNUAL`
- [x] `STRIPE_PRICE_PRO_MONTHLY`
- [x] `STRIPE_PRICE_PRO_ANNUAL`

### Email (MailerSend)
- [ ] `MAILERSEND_API_KEY` - **VERIFY IN RAILWAY**
- [ ] `MAILERSEND_FROM_EMAIL` (default: noreply@schakel.ai)

### Core
- [x] `DATABASE_URL`
- [x] `SESSION_SECRET`
- [x] `ANTHROPIC_API_KEY`
- [x] `ENCRYPTION_KEY`
- [x] `VITE_FRONTEND_URL`

---

## Quick Commands

```bash
# Type check
npm run check

# Check scheduled recordings
psql "$DATABASE_URL" -c "SELECT id, meeting_title, status, meeting_start FROM scheduled_recordings ORDER BY created_at DESC LIMIT 10;"

# Delete old scheduled recordings (past meetings)
psql "$DATABASE_URL" -c "DELETE FROM scheduled_recordings WHERE meeting_start < NOW() - INTERVAL '1 day';"

# Check Recall webhook test endpoint
curl -s "https://map-api-dev.schakel.ai/api/webhook/recall/test"

# Make tenant billing-exempt
psql "$DATABASE_URL" -c "UPDATE tenants SET billing_exempt = true WHERE slug = 'your-tenant';"
```

---

## Files Changed (1 Feb 2026 - Late Night)

```
# Modified files
server/services/recall.ts              # Regional endpoints, simplified bot payload
client/src/lib/timezone.ts             # Fixed double conversion, dynamic detection
client/src/pages/settings.tsx          # Cache invalidation fix, provider save fix
server/routes/recordings.ts            # Route ordering, early join feature
server/supabase-storage.ts             # LLM usage aggregation fix
server/services/email.ts               # Removed "Eerste stappen" from welcome email
shared/schema.ts                       # botJoinLeadTimeMinutes setting
```

---

## Architectuur Notes

### Recall.ai Flow
```
1. Calendar Sync / Manual Add
   ↓
2. POST /api/recordings → Creates scheduled_recording + Recall bot
   ↓
3. Bot joins meeting at scheduled time (minus lead time)
   ↓
4. Recall sends webhook events (status changes)
   ↓
5. transcript.done webhook → Fetch transcript → Create meeting
   ↓
6. Meeting appears in dashboard for AI processing
```

### Timezone Handling
```
1. Server stores all times in UTC (ISO 8601)
2. Database: TIMESTAMPTZ columns
3. API: Returns ISO strings (UTC)
4. Client: Uses formatInTimeZone() with browser's timezone
5. Display: Shows local time to user
```

### Subscription Access Flow
```
1. User logs in
   ↓
2. SubscriptionGuard checks /api/billing/status
   ↓
3. Check order:
   - Superadmin? → full access
   - Tenant.billingExempt? → full access
   - Tenant.isActive = false? → blocked
   - Subscription status → full/warning/blocked
   ↓
4. Render app or blocked page
```

---

## Pricing

| Tier | Monthly | Annual | Trial | Features |
|------|---------|--------|-------|----------|
| Starter | €49 | €470 | 14 dagen | 50 meetings, 1 user, basis AI |
| Pro | €149 | €1,430 | 14 dagen | 250 meetings, 10 users, SharePoint, Productive |
| Enterprise | Custom | Custom | - | Onbeperkt, dedicated support |

---

## Future Ideas

### Mobile Recording App (Voxtral Transcribe)
**Status:** Idee - nog niet uitgewerkt
**Datum:** 9 februari 2026

Een native mobile app waarmee gebruikers fysieke meetings en telefoongesprekken kunnen opnemen en direct laten transcriberen. Aanvulling op Recall.ai (dat online meetings afhandelt).

**Technologie:**
- **Transcriptie**: Voxtral Transcribe 2 (Mistral AI, feb 2026) - $0.003/minuut, Nederlands ondersteund, speaker diarization ingebouwd, context biasing voor vakjargon
- **App**: React Native / Expo (sluit aan bij bestaande React/TypeScript stack)
- **Aparte GitHub repo** met eigen build pipeline (Expo/EAS Build), communiceert met dezelfde Express backend API op Railway

**Flow:**
1. Gebruiker opent app, drukt op "Opnemen"
2. Audio wordt lokaal opgenomen op de telefoon
3. Na afloop uploaden naar backend
4. Backend stuurt naar Voxtral API voor transcriptie met diarization
5. Transcript gaat door bestaande Claude-pipeline voor notities/categorisatie/acties
6. Resultaat verschijnt in het dashboard, net als bij Recall.ai meetings

**Backend-werk in deze repo:**
- Nieuw endpoint voor audio upload + Voxtral integratie
- Dit endpoint is ook vanuit de web-app bruikbaar

**Kosten vergelijking per uur meeting:**
- Recall.ai: ~$2-4 (bot + transcriptie)
- Voxtral: ~$0.18 (alleen transcriptie, opname gratis op device)
