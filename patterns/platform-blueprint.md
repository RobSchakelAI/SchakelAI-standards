# Schakel Platform Blueprint

> **Versie:** 0.2 — Draft (Scout integration)
> **Datum:** 17 februari 2026
> **Auteurs:** Rob & Simon (Schakel AI B.V.) + Claude
> **Status:** Levend document — wordt aangescherpt met elke build

---

## Leeswijzer

Dit document is het technische antwoord op het visiedocument "Van AI Agency naar Managed Operations Partner." Waar dat document het *waarom* en *wat* beschrijft, beschrijft dit document het *hoe*.

Het combineert:
- De strategische visie (Managed Operations Centers voor het MKB)
- De technische lessen uit MAP (Meeting Automation Platform)
- De gecodificeerde patronen uit B2B SaaS Infrastructure Skills
- De Founders Context Brief (wie we zijn en hoe we denken)
- **Schakel Scout** (het kennisbeheersysteem dat het compounding effect aandrijft)
- Ontwerprichtlijnen en deployment-ervaring

Het doel: één blauwdruk die beschrijft hoe het Schakel Platform werkt — van module tot deployment, van eerste klantgesprek tot draaiend systeem, en hoe de kennis die dat alles voedt systematisch wordt beheerd.

---

## 1. Wat we bouwen

### De elevator pitch

Schakel bouwt en beheert **op maat samengestelde Operations Centers** voor MKB-bedrijven. Elk systeem ziet eruit alsof het speciaal voor die klant is gebouwd — en dat klopt ook — maar onder de motorkap deelt het een bewezen, modulair platform met alle andere klanten.

### Het pak-metafoor (technisch)

Een kleermaker met standaardpatronen voor jasjes, broeken en vesten. Elke klant krijgt een andere combinatie: de stof, de kleur, de knopen, de pasvorm worden aangepast. De basispatronen zijn bewezen en worden hergebruikt.

**Technisch vertaald:**

| Metafoor | Platform |
|----------|----------|
| Basispatronen | Schakel Core (auth, billing, multi-tenancy, RLS, AI) |
| Kledingstukken | Modules (taken, uren, facturatie, meetings, HR, roosters, etc.) |
| Kleur & stof | Client configuratie (branding, terminologie, navigatie) |
| Pasvorm | Client-specifieke integraties en bedrijfslogica |
| Kleermaker | Rob + Claude |
| Verkoper | Simon |

### Vier componenten, één ecosysteem

```
┌─────────────────────────────────────────────────────────────────────┐
│                       SCHAKEL ECOSYSTEEM                            │
├────────────┬───────────────────┬──────────────────┬─────────────────┤
│ Schakel    │  SaaS             │ Operations       │ Service         │
│ Scout      │  Products         │ Centers (kern)   │ Projects        │
├────────────┼───────────────────┼──────────────────┼─────────────────┤
│ Kennis-    │ MAP               │ Easydash         │ Junea Scout     │
│ beheer +   │ LinkedIn Tool     │ DRG              │ Nirint          │
│ auto-      │                   │ Klant 3..N       │ Toekomstige     │
│ harvest    │                   │                  │ pilots          │
├────────────┼───────────────────┼──────────────────┼─────────────────┤
│ Beheert    │ Multi-tenant      │ Per-client       │ Standalone of   │
│ schakel-   │ single deploy     │ eigen database   │ module-         │
│ core repo  │                   │ eigen domein     │ bijdrage        │
└────────────┴───────────────────┴──────────────────┴─────────────────┘
      │                                    ▲
      │ schakel-core (skills/rules/        │ Learnings vloeien
      │ patterns) voedt alle builds        │ terug via Scout
      └────────────────────────────────────┘
```

**SaaS Products** (MAP, LinkedIn): Multi-tenant, één deployment, gedeelde database met RLS. Secundaire inkomstenstroom en visitekaartje. MAP is ook het R&D-lab waar nieuwe patronen worden getest.

**Operations Centers** (de kern): Per-client deployment, eigen database, eigen domein (`app.klantnaam.nl`). Recurring revenue. Het product dat compoundt.

**Service Projects**: Klantwerk dat het platform verrijkt. Elke build voegt modules of integraties toe aan het platform. Klanten betalen voor hun oplossing; Schakel bouwt tegelijkertijd IP op.

---

## 2. Platform Architectuur

### Overzicht

```
schakel-platform/
├── core/                          # Schakel Core — het fundament
│   ├── auth/                      # Session-based auth, MFA, CSRF
│   ├── billing/                   # Stripe subscriptions, checkout, portal
│   ├── multi-tenancy/             # RLS, tenant isolation, tenant context
│   ├── storage/                   # Database layer, withTenant(), migrations
│   ├── ai/                        # Claude API wrapper, capability system
│   ├── email/                     # MailerSend transactional email
│   ├── file-storage/              # Supabase Storage, signed URLs, tenant-prefixed
│   ├── error-handling/            # Centralized error codes, ApiError
│   ├── audit/                     # Security event logging
│   ├── middleware/                 # Rate limiting, CORS, helmet
│   └── ui/                        # shadcn/ui base components + Schakel extensions
│
├── modules/                       # Herbruikbare functionele modules
│   ├── werkruimtes/               # Workspaces met access control
│   ├── documents/                 # Documentbeheer + file uploads
│   ├── timeline/                  # Activiteitenfeed (meetings + docs + events)
│   ├── ai-agent/                  # Per-workspace AI chat agent
│   ├── tasks/                     # Taakbeheer / kanban
│   ├── time-tracking/             # Urenregistratie (declarabel/niet-declarabel)
│   ├── invoicing/                 # Facturatie (Moneybird, Exact, standalone)
│   ├── meeting-intelligence/      # MAP core (recording, transcription, AI notes)
│   ├── hr/                        # Personeelsbeheer
│   ├── scheduling/                # Roosterplanning met conflictdetectie
│   ├── dashboards/                # BI dashboards, KPIs, rapportages
│   ├── content-management/        # LinkedIn tool core (voice profiles, publishing)
│   ├── calendar/                  # Kalenderintegratie (Microsoft 365)
│   ├── onboarding/                # Gebruikers-onboarding wizard
│   └── notifications/             # In-app en email notificaties
│
├── integrations/                  # Externe koppelingen
│   ├── microsoft-365/             # SharePoint, Outlook, Calendar (Graph API)
│   ├── moneybird/                 # Boekhoudkoppeling
│   ├── recall-ai/                 # Meeting bot (Teams, Zoom, Meet, Webex)
│   ├── productive/                # Projectmanagement (Productive.io)
│   ├── pipedrive/                 # CRM
│   ├── foursquare/                # Locatiedata
│   └── prospeo/                   # Email verrijking
│
├── clients/                       # Per-client configuratie
│   ├── easydash/
│   │   ├── config.ts              # Modules, terminologie, branding, integraties
│   │   ├── theme.ts               # Kleuren, fonts, logo
│   │   └── overrides/             # Client-specifieke componenten (indien nodig)
│   ├── drg/
│   │   ├── config.ts
│   │   ├── theme.ts
│   │   └── overrides/
│   └── _template/                 # Startpunt voor nieuwe klanten
│       ├── config.ts
│       └── theme.ts
│
├── apps/                          # Deployment targets
│   ├── saas/
│   │   ├── map/                   # MAP SaaS deployment config
│   │   └── linkedin/              # LinkedIn tool SaaS deployment config
│   └── ops-center/                # Operations Center build target
│       └── build.ts               # Leest client config, produceert tailored build
│
├── standards/                     # schakel-core repo (als submodule of symlink)
│   ├── skills/                    # B2B SaaS Infrastructure Skills (10 bestanden)
│   ├── patterns/                  # Architectuur referenties (incl. dit document)
│   ├── rules/                     # Codeerstandaarden, security, design system
│   └── context/                   # Visie, founders brief
│
└── shared/                        # Gedeelde types, schemas, utilities
    ├── schema.ts                  # Zod schemas, Drizzle tables, TypeScript types
    ├── types.ts                   # Platform-brede type definities
    └── constants.ts               # Tier limits, feature flags, defaults
```

### Schakel Core

Het fundament onder elk product. Geëxtraheerd uit MAP's productie-gevalideerde code en gecodificeerd in B2B SaaS Infrastructure Skills.

**Niet-onderhandelbare principes:**

| Principe | Implementatie |
|----------|---------------|
| Session-based auth | Geen JWT in localStorage. Cookies (httpOnly, secure, sameSite) + CSRF |
| Multi-tenant isolatie | App-layer (tenantId verplicht) + Database-layer (PostgreSQL RLS) |
| Defense in depth | Elke laag valideert onafhankelijk. Geen vertrouwen op "de vorige laag checkt het wel" |
| Tenant context altijd expliciet | `tenantId: string \| null` — nooit optioneel, `null` = bewuste superadmin bypass |
| Encrypted at rest | API keys, OAuth tokens, gevoelige data encrypted met AES-256-GCM |
| Audit logging | Alle security-gevoelige events worden gelogd |

### Module Anatomie

Elke module is een zelfstandige eenheid met een vast patroon:

```
modules/tasks/
├── schema.ts              # Drizzle tabel definities + Zod validatie
├── migrations/            # SQL migraties voor deze module
│   └── 001_tasks.sql
├── storage.ts             # IStorage interface methods voor deze module
├── routes.ts              # Express API endpoints
├── capabilities.ts        # AI agent capabilities (query + mutation)
├── components/            # React UI componenten
│   ├── task-board.tsx     # Kanban board
│   ├── task-list.tsx      # Lijst view
│   ├── task-detail.tsx    # Detail dialog
│   └── task-form.tsx      # Create/edit form
├── hooks.ts               # React hooks (useQuery, useMutation wrappers)
├── config.ts              # Module configuratie schema
│   # Wat is configureerbaar:
│   # - Terminologie ("Taken" vs "Work Items" vs "Tickets")
│   # - Beschikbare statussen
│   # - Kanban kolommen
│   # - Welke velden zichtbaar zijn
│   # - Integraties (Productive.io sync ja/nee)
└── index.ts               # Module entry point (registratie)
```

**Module registratie:**

```typescript
// modules/tasks/index.ts
export const tasksModule: PlatformModule = {
  id: "tasks",
  name: "Taakbeheer",
  description: "Kanban-board en taakbeheer",
  version: "1.0.0",

  // Wat deze module nodig heeft
  dependencies: ["werkruimtes"],  // Taken leven binnen werkruimtes

  // Database
  schema: tasksSchema,
  migrations: tasksMigrations,

  // Backend
  routes: tasksRoutes,          // Registreert /api/tasks/* endpoints
  storage: tasksStorage,        // IStorage method implementations
  capabilities: tasksCapabilities,  // AI agent tools

  // Frontend
  components: tasksComponents,  // Exporteert alle UI componenten
  pages: tasksPages,            // Registreert route pages
  navigation: {                 // Sidebar/nav items
    icon: "CheckSquare",
    label: "tasks",             // Wordt vertaald via terminologie config
    position: 3,
  },

  // Configuratie
  configSchema: tasksConfigSchema,  // Zod schema voor module config
  defaults: tasksDefaults,          // Standaard configuratie
};
```

---

## 3. Client Configuratie

### Het configuratiebestand

Elk Operations Center wordt gedefinieerd door één configuratiebestand dat bepaalt hoe het platform zich gedraagt voor die klant.

```typescript
// clients/easydash/config.ts
import type { ClientConfig } from "@schakel/platform";

export const config: ClientConfig = {
  // === Identiteit ===
  id: "easydash",
  name: "Easydash",
  domain: "app.easydash.nl",

  // === Branding ===
  branding: {
    primaryColor: "#2563eb",
    accentColor: "#f59e0b",
    logo: "./assets/easydash-logo.svg",
    favicon: "./assets/favicon.ico",
    appTitle: "Easydash",
  },

  // === Navigatie ===
  navigation: {
    layout: "sidebar",            // "sidebar" | "topnav"
    orientation: "project",       // Hoe werkruimtes worden gepresenteerd
  },

  // === Terminologie ===
  // Vertaalt generieke platformtermen naar klantspecifieke taal
  terminology: {
    workspace: "Project",
    workspaces: "Projecten",
    workspaceType: "project",     // Default type bij aanmaken
    member: "Teamlid",
    members: "Team",
    task: "Taak",
    tasks: "Taken",
    timeEntry: "Uurregistratie",
    invoice: "Factuur",
    meeting: "Vergadering",
    dashboard: "Dashboard",
  },

  // === Actieve modules ===
  modules: {
    werkruimtes: {
      enabled: true,
      types: ["project", "internal"],  // Alleen project en intern
      defaultType: "project",
    },
    tasks: {
      enabled: true,
      kanban: true,
      statuses: ["backlog", "todo", "in_progress", "review", "done"],
    },
    timeTracking: {
      enabled: true,
      billable: true,
      nonBillable: true,
      roundingInterval: 15,       // Minuten
    },
    invoicing: {
      enabled: true,
      provider: "moneybird",
      autoGenerate: false,        // Handmatig genereren
    },
    meetingIntelligence: {
      enabled: true,
      provider: "recall",
      autoJoin: true,
    },
    documents: { enabled: true },
    timeline: { enabled: true },
    aiAgent: { enabled: true },
    dashboards: {
      enabled: true,
      widgets: ["project-health", "hours-budget", "invoice-status"],
    },
    hr: { enabled: false },
    scheduling: { enabled: false },
  },

  // === Integraties ===
  integrations: {
    moneybird: { enabled: true },
    microsoft365: {
      enabled: true,
      sharepoint: true,
      outlook: true,
      calendar: true,
    },
    productive: { enabled: false },
  },

  // === Billing ===
  billing: {
    model: "managed",            // "managed" (Operations Center) | "saas" (self-service)
    monthlyFee: 2000,            // EUR
    setupFee: 5000,              // EUR
    billingExempt: false,        // true voor enterprise/managed klanten zonder Stripe
  },
};
```

```typescript
// clients/drg/config.ts — compleet andere applicatie, zelfde platform
export const config: ClientConfig = {
  id: "drg",
  name: "DRG Operations",
  domain: "app.drg-operations.nl",

  branding: {
    primaryColor: "#059669",
    logo: "./assets/drg-logo.svg",
    appTitle: "DRG Operations",
  },

  navigation: {
    layout: "topnav",              // Top navigatie i.p.v. sidebar
    orientation: "department",     // Afdelingsgeoriënteerd
  },

  terminology: {
    workspace: "Afdeling",         // Niet "Project" maar "Afdeling"
    workspaces: "Afdelingen",
    member: "Medewerker",
    members: "Medewerkers",
    task: "Actie",
    dashboard: "Overzicht",
  },

  modules: {
    werkruimtes: {
      enabled: true,
      types: ["department"],
      defaultType: "department",
    },
    hr: {
      enabled: true,               // HR module AAN (bij Easydash UIT)
      contractTypes: ["vast", "flex", "oproep"],
    },
    scheduling: {
      enabled: true,               // Roosterplanning AAN
      conflictDetection: true,
      laborLawCompliance: "nl",    // Nederlandse arbeidstijdenwet
    },
    dashboards: {
      enabled: true,
      widgets: ["staff-overview", "weather-impact", "event-calendar"],
    },
    documents: { enabled: true },
    timeline: { enabled: true },
    aiAgent: { enabled: true },
    // Modules die UIT staan voor DRG:
    tasks: { enabled: false },
    timeTracking: { enabled: false },
    invoicing: { enabled: false },
    meetingIntelligence: { enabled: false },
  },

  integrations: {
    moneybird: { enabled: false },
    microsoft365: { enabled: false },
  },
};
```

**Easydash en DRG zouden niet eens weten dat ze op hetzelfde platform draaien.**

---

## 4. Deployment Strategie

### Operations Centers: Per-client deployment

Elke Operations Center klant krijgt een volledig gescheiden omgeving:

```
┌──────────────────────────────────────────────────────────┐
│                    Schakel Platform Repo                  │
│                    (één codebase)                         │
└────────────┬────────────────────────┬────────────────────┘
             │                        │
        build easydash           build drg
             │                        │
             ▼                        ▼
┌────────────────────┐  ┌────────────────────────┐
│ app.easydash.nl    │  │ app.drg-operations.nl  │
├────────────────────┤  ├────────────────────────┤
│ Railway Project A  │  │ Railway Project B      │
│ Supabase DB A      │  │ Supabase DB B          │
│ Eigen env vars     │  │ Eigen env vars         │
│ Eigen backups      │  │ Eigen backups          │
└────────────────────┘  └────────────────────────┘
```

**Waarom per-client deployment (niet multi-tenant)?**

| Factor | Multi-tenant | Per-client | Keuze |
|--------|-------------|------------|-------|
| Data-isolatie | RLS (goed, niet perfect) | Fysiek gescheiden (bulletproof) | **Per-client** |
| Compliance/privacy | Complex bij AVG-vragen | Simpel: "uw data staat in uw database" | **Per-client** |
| Downtime-impact | Eén deployment = alle klanten | Alleen die ene klant | **Per-client** |
| Custom integraties | Lastig per tenant | Eigen env vars per client | **Per-client** |
| Kosten per klant | €5-10/mnd shared | €50-150/mnd dedicated | Acceptabel bij €1.500-3.000/mnd fee |
| Schaalbaarheid | Onbeperkt | Max ~50-100 met handmatig beheer | Voldoende voor MKB-focus |
| Updates uitrollen | Eén deploy = klaar | Per client deployen | Trade-off (zie CI/CD) |

Bij €2.000/mnd recurring per klant is €100-150/mnd infrastructuur (7.5%) ruimschoots acceptabel.

### SaaS Products: Multi-tenant deployment

MAP en LinkedIn Tool draaien als klassieke multi-tenant SaaS:

```
┌─────────────────────┐
│ map.schakel.ai      │  Eén deployment, alle tenants
├─────────────────────┤
│ Vercel (frontend)   │
│ Railway (backend)   │
│ Supabase (shared DB)│
│ RLS tenant isolatie │
└─────────────────────┘
```

### CI/CD Pipeline

```
Push naar platform repo
         │
         ▼
    GitHub Actions
         │
    ┌────┴────┐
    │  Tests  │  npm test, tsc --noEmit, lint
    └────┬────┘
         │ (pass)
         ▼
    Welke clients zijn gewijzigd?
    (of: core gewijzigd → alles herbouwen)
         │
    ┌────┴──────────┬────────────────┐
    │               │                │
    ▼               ▼                ▼
Build Easydash  Build DRG      Build MAP SaaS
    │               │                │
    ▼               ▼                ▼
Deploy to       Deploy to        Deploy to
Railway A       Railway B        Railway + Vercel
```

### Nieuwe klant opzetten (het "Klant-0-naar-Live" playbook)

```
Dag 1:  Discovery gesprek (Simon)
        → Welke modules? Welke integraties? Welke terminologie?
        → Output: ingevuld config.ts bestand

Dag 2:  Infrastructuur provisioning
        → Supabase project aanmaken
        → Railway project aanmaken
        → DNS instellen (app.klantnaam.nl)
        → Environment variables configureren

Dag 3-5: Configuratie en branding
        → Config.ts verfijnen
        → Logo, kleuren, terminologie instellen
        → Eventuele client-specifieke overrides bouwen

Dag 5-10: Client-specifieke modules (indien nodig)
        → Nieuwe module bouwen als er een gap is
        → Module toevoegen aan platform (herbruikbaar!)

Dag 10-14: Testing en onboarding
        → Simon doet de onboarding met de klant
        → Rob monitort de eerste week

Dag 14: Live
        → Operations Center draait
        → Maandelijks beheer start
```

---

## 5. Technology Stack

### Bevestigde stack (gevalideerd door MAP productie)

| Laag | Technologie | Bewezen in |
|------|-------------|------------|
| **Frontend** | React 18 + TypeScript + Vite | MAP, LinkedIn |
| **Styling** | TailwindCSS + shadcn/ui | MAP, LinkedIn |
| **Backend** | Node.js + Express + TypeScript | MAP |
| **Database** | PostgreSQL (Supabase) + Drizzle ORM | MAP |
| **Auth** | Session-based (express-session) + CSRF + MFA | MAP |
| **Betalingen** | Stripe (subscriptions, checkout, portal) | MAP |
| **AI** | Anthropic Claude API | MAP, alle projecten |
| **File Storage** | Supabase Storage (signed URLs) | MAP |
| **Email** | MailerSend | MAP |
| **Hosting backend** | Railway | MAP |
| **Hosting frontend** | Vercel | MAP |
| **DNS** | Cloudflare (of klant's DNS) | MAP |

### Ontwikkeltools

| Tool | Rol |
|------|-----|
| **Claude Code** | Primaire development partner. Schrijft, reviewt, en onderhoud code |
| **Claude (chat)** | Strategie, documentatie, architectuurbeslissingen |
| **GitHub** | Source control, CI/CD, issue tracking |
| **Cursor** | Aanvullende IDE voor snelle iteraties |

### Waarom deze stack

1. **Bewezen**: MAP draait in productie met deze stack. Geen experimenten.
2. **AI-vriendelijk**: Claude Code kent deze stack uitstekend. Maximale productiviteit.
3. **Compounding**: Elke build in deze stack verrijkt B2B SaaS Infrastructure Skills en het platform.
4. **MKB-passend**: Geen overkill. Geen Kubernetes. Geen microservices. Gewoon goede, begrijpelijke architectuur die twee mensen (+ AI) kunnen onderhouden.

---

## 6. Het Vliegwiel — Technisch

```
                    ┌──────────────────┐
                    │  Nieuwe klant    │
                    │  (Simon vindt)   │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  Discovery       │
                    │  Config opstellen│
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    │ Bestaat module?  │
                    └──┬───────────┬───┘
                  Ja   │           │  Nee
                       ▼           ▼
              ┌──────────┐  ┌──────────────────┐
              │ Config +  │  │ Bouw module      │
              │ deploy    │  │ (herbruikbaar!)  │
              └──────┬────┘  └────────┬─────────┘
                     │               │
                     │    ┌──────────┘
                     │    │ Module vloeit terug
                     │    │ naar platform
                     │    ▼
                     │  ┌──────────────────┐
                     │  │ Platform wordt   │
                     │  │ rijker           │
                     │  └──────────────────┘
                     │    │
                     ▼    ▼
              ┌──────────────────┐
              │  Klant live      │
              │  Recurring €€€   │
              └──────────────────┘
                     │
                     ▼
              Volgende klant is SNELLER
              (meer modules bestaan al)
```

**Concrete compounding per klant:**

| Klant | Nieuw gebouwd | Hergebruikt | Geschatte effort |
|-------|---------------|-------------|------------------|
| Easydash (klant 1) | Core + 7 modules + 2 integraties | Niets (eerste build) | Maanden |
| DRG (klant 2) | HR module + scheduling module | Core + documents + timeline + AI agent + dashboards | Weken |
| Klant 3 | Mogelijk 1 nieuwe module | Core + 8-10 modules | 1-2 weken |
| Klant 5-10 | Waarschijnlijk alleen configuratie | Alles | Dagen |

---

## 7. Schakel Scout — De Kennismotor

Het vliegwiel uit sectie 6 beschrijft hoe elke klant het platform verrijkt. Maar dat vliegwiel draait alleen als kennis **systematisch** wordt vastgelegd, bijgewerkt en verdeeld. Zonder systeem is kennisdeling ad-hoc: je onthoudt iets, zoekt het op, past het misschien toe. Met de Scout wordt het machinaal.

### Wat de Scout doet

Schakel Scout is een interne web-applicatie die twee kennisstromen beheert:

1. **Extern**: Wekelijks automatisch het web afscannen naar nieuwe best practices, patronen en tooling-updates voor onze stack — en die als concrete voorstellen aanbieden
2. **Intern**: Centraal beheer van alle skills, rules en patterns die onze projecten voeden — zodat een verbetering in MAP automatisch beschikbaar is voor Easydash, DRG, en elk toekomstig project

```
┌────────────────────────────────────────────────────────────┐
│                     SCHAKEL SCOUT                           │
│                                                            │
│  ┌─────────────────┐           ┌─────────────────────┐    │
│  │ Knowledge Hub   │           │ External Scout       │    │
│  │                 │           │                      │    │
│  │ Browse, search, │           │ Perplexity harvests  │    │
│  │ create, edit    │           │ Opus 4.6 analyseert  │    │
│  │ skills/rules/   │           │ Voorstellen ter      │    │
│  │ patterns        │           │ review               │    │
│  └────────┬────────┘           └──────────┬───────────┘    │
│           │                               │                │
│           └───────────┬───────────────────┘                │
│                       ▼                                    │
│              ┌─────────────────┐                           │
│              │  schakel-core   │  (Private GitHub repo)    │
│              │  Git = source   │                           │
│              │  of truth       │                           │
│              └────────┬────────┘                           │
└───────────────────────┼────────────────────────────────────┘
                        │ git pull (per project)
           ┌────────────┼────────────────┐
           ▼            ▼                ▼
     ┌──────────┐ ┌──────────┐    ┌──────────┐
     │   MAP    │ │ Easydash │    │   DRG    │
     │   SaaS   │ │ Ops Ctr  │    │ Ops Ctr  │
     └──────────┘ └──────────┘    └──────────┘
```

### Drie kennistypes

| Type | Doel | Voorbeeld | Claude Code locatie |
|------|------|-----------|---------------------|
| **Skills** | On-demand kennis die Claude laadt wanneer relevant | `drizzle-patterns.md`, `react-conventions.md` | `.claude/skills/<naam>/SKILL.md` |
| **Rules** | Altijd-actieve instructies per sessie | `error-handling.md`, `code-style.md` | `.claude/rules/<naam>.md` |
| **Patterns** | Architectuurreferenties voor nieuwe features | `webhook-handling.md`, `multi-tenant.md` | Referentie (niet direct geladen) |

### Het compounding mechanisme

```
EXTERN (wereld leert)                    INTERN (wij leren)
        │                                        │
        ▼                                        ▼
Perplexity scant wekelijks          Rob bouwt in MAP/Easydash/DRG
        │                                        │
        ▼                                        ▼
Opus 4.6 analyseert                 Ontdekt betere aanpak
vs. huidige kennis                          │
        │                                   ▼
        ▼                           Updatet via Scout UI
Voorstellen in Scout UI                     │
        │                                   │
        ▼                                   ▼
Rob reviewt (< 15 min/week)    ─────► schakel-core (git)
                                            │
                                   git pull per project
                                            │
                               ┌────────────┼────────────┐
                               ▼            ▼            ▼
                             MAP      Easydash        DRG
                          (profiteert van alles wat eerder geleerd is)
```

**Zonder Scout:** Kennis zit in Rob's hoofd en verspreid over projecten. Ad hoc, inconsistent.
**Met Scout:** Kennis is gecentraliseerd, versioned, doorzoekbaar, en automatisch verrijkt. Elke maandagochtend is de hele development setup een stukje slimmer.

### Harvest pipeline (wekelijks)

| Fase | Service | Wat het doet |
|------|---------|-------------|
| **1. Harvest** | Perplexity Sonar API | 15-20 queries over 3 domeinen (claude-code, fullstack, ai-workflows) |
| **2. Analyse** | Claude Opus 4.6 API | Vergelijkt bevindingen met huidige kennis. Genereert voorstellen: create / update / deprecate |
| **3. Review** | Scout UI | Rob beoordeelt voorstellen. Approve → auto-commit naar schakel-core |

**Kosten:** ~€8/maand (Perplexity + Opus API calls). ROI: één vermeden fout of één betere pattern per maand betaalt dit 100x terug.

### Source of Truth principe

- **Git (schakel-core repo)** = source of truth voor **content**. De daadwerkelijke skill/rule/pattern bestanden.
- **Database (Neon)** = source of truth voor **metadata**. Tags, bronverwijzingen, versiegeschiedenis, rapportdata, voorstelstatussen.
- **Sync:** Bij startup en elke 5 minuten scant de app de repo op wijzigingen. Bestanden in git maar niet in DB → metadata record aanmaken. Bestanden in DB maar verwijderd uit git → markeren als deleted.

### Scout tech stack

| Component | Technologie | Waarom |
|-----------|-------------|--------|
| Frontend | React 18 + TypeScript + Vite + shadcn/ui | Standaard Schakel stack |
| Backend | Express.js + TypeScript | Standaard Schakel stack |
| Database | Neon PostgreSQL + Drizzle ORM | Standaard Schakel stack |
| Git | simple-git (Node.js) | Clone, pull, commit, push naar schakel-core |
| Search | Perplexity Sonar API | Externe kennisharvest |
| Analysis | Anthropic Opus 4.6 API | Intelligente vergelijking en voorstellen |
| Scheduling | node-cron | Wekelijkse harvest (zondag 03:00) |

### Relatie tot het Platform Blueprint

De Scout **beheert** de kennis. Het Platform Blueprint **gebruikt** die kennis.

```
Scout beheert schakel-core:
  skills/drizzle-patterns.md
  rules/error-handling.md
  patterns/multi-tenant.md
  patterns/webhook-handling.md
  ...

Platform projecten consumeren schakel-core:
  MAP:      git pull → .claude/skills/, .claude/rules/
  Easydash: git pull → .claude/skills/, .claude/rules/
  DRG:      git pull → .claude/skills/, .claude/rules/
```

De B2B SaaS Infrastructure Skills zijn **opgesplitst** in 10 bestanden in `skills/` (van `00-introduction.md` tot `09-build-deploy.md`). Op termijn worden deze verder verfijnd in individuele skills en patterns, beheerd via de Scout.

### Bouwvolgorde Scout (geschat: 15-20 uur)

| Sprint | Scope | Uren |
|--------|-------|------|
| 1. Fundament | Repo setup, DB schema, git integration, sync | 3-4 |
| 2. Knowledge Hub | CRUD, filters, markdown editor, versiegeschiedenis | 3-4 |
| 3. External Scout | Perplexity + Opus pipeline, rapporten | 4-5 |
| 4. Review Flow | Approve/reject/edit, auto-commit | 2-3 |
| 5. Settings & Polish | Query management, schedule, seed data | 2-3 |

---

## 8. De Rob + Claude + Simon Workflow

### Hoe een Operations Center wordt gebouwd

```
Simon                              Rob + Claude
──────                             ────────────
Klantgesprek
  → Behoeften inventariseren
  → Config draft opstellen ──────► Config.ts verfijnen
                                   Module gap-analyse
                                   Nieuwe modules bouwen
                                   Deployment opzetten
                                   Testing
Simon
  → Klant onboarding
  → Training
  → Eerste support ──────────────► Monitoring
                                   Bug fixes
                                   Doorontwikkeling
```

### Hoe Claude Code wordt ingezet

Claude Code is niet "een tool die Rob gebruikt." Claude is de facto de derde developer.

**Per project: een CLAUDE.md** (zoals MAP's CLAUDE.md) die de AI volledige context geeft:
- Projectoverzicht en architectuur
- Actieve modules en configuratie
- Deployment details
- Recente wijzigingen
- Security checklist
- Beschikbare commando's

**Per platform: B2B SaaS Infrastructure Skills** als de technische bijbel die Claude raadpleegt voor:
- Auth patronen (session-based, MFA, CSRF)
- Multi-tenancy (RLS, tenant isolation)
- Stripe billing
- Error handling
- Security best practices
- Migration strategieën

**De compound interest van documentatie:**
Elke keer dat Rob een probleem oplost, wordt de oplossing gedocumenteerd in B2B SaaS Infrastructure Skills of in de relevante module docs. De volgende keer dat Claude (of een toekomstige developer) hetzelfde probleem tegenkomt, is de oplossing er al. Dit is hoe twee mensen het werk doen van tien.

---

## 9. Per-Client Deployment — Gedetailleerd

### Infrastructuur per klant

| Component | Service | Kosten (indicatief) |
|-----------|---------|---------------------|
| Backend | Railway (Hobby plan per project) | ~€5-20/mnd |
| Database | Supabase (Free/Pro per project) | €0-25/mnd |
| Frontend | Vercel (per project of subfolder) | €0-20/mnd |
| File storage | Supabase Storage (per project) | Included |
| Domein | Klant's eigen domein of subdomain | €0-15/jaar |
| SSL | Automatisch via Railway/Vercel/Cloudflare | Gratis |
| **Totaal** | | **€5-80/mnd** |

Bij een maandelijkse fee van €1.500-3.000 is dit 2-5% van de omzet.

### Domein strategie

**Optie A: Eigen domein (aanbevolen voor vertrouwen)**
- `app.easydash.nl` — voelt als hun eigen systeem
- Klant beheert eigen domein, wijst CNAME naar Schakel

**Optie B: Schakel subdomain (sneller op te zetten)**
- `easydash.schakel.app` — duidelijk onderdeel van Schakel platform
- Schakel beheert alles

**Aanbeveling:** Start met Optie B voor snelheid. Migreer naar Optie A zodra klant productie gaat.

### Updates en onderhoud

```
Platform update beschikbaar
         │
         ▼
    Staging deploy per client
    (automatisch via CI/CD)
         │
         ▼
    Smoke tests (automatisch)
         │
    ┌────┴────┐
    │  Pass?  │
    └──┬───┬──┘
    Ja │   │ Nee
       ▼   ▼
  Productie  Alert naar Rob
  deploy     → Fix → Retry
```

**Rollback:** Elke client deployment is onafhankelijk. Als een update bij Easydash problemen geeft, kan die worden teruggedraaid zonder DRG te raken.

---

## 10. Module Catalogus (huidige en geplande)

### Status per module

| Module | Status | Bron | Beschikbaar voor |
|--------|--------|------|------------------|
| **Core: Auth** | ✅ Productie | MAP | Alle projecten |
| **Core: Multi-tenancy + RLS** | ✅ Productie | MAP | Alle projecten |
| **Core: Billing (Stripe)** | ✅ Productie | MAP | SaaS producten |
| **Core: AI (Claude)** | ✅ Productie | MAP | Alle projecten |
| **Core: File Storage** | ✅ Productie | MAP | Alle projecten |
| **Core: Email** | ✅ Productie | MAP | Alle projecten |
| **Werkruimtes** | ✅ Productie | MAP | Alle projecten |
| **Documents** | ✅ Productie | MAP | Alle projecten |
| **Timeline** | ✅ Productie | MAP | Alle projecten |
| **AI Agent** | ✅ Productie | MAP | Alle projecten |
| **Meeting Intelligence** | ✅ Productie | MAP | MAP + Ops Centers die het willen |
| **Onboarding** | ✅ Productie | MAP | Alle projecten |
| **Dashboards** | 🟡 Basis | MAP | Uitbreiden per klant |
| **Content Management** | 🟡 Basis | LinkedIn Tool | LinkedIn + content-klanten |
| **Tasks / Kanban** | 🔴 Nieuw te bouwen | Easydash | Easydash + toekomstige klanten |
| **Time Tracking** | 🔴 Nieuw te bouwen | Easydash | Easydash + toekomstige klanten |
| **Invoicing** | 🔴 Nieuw te bouwen | Easydash | Easydash + toekomstige klanten |
| **HR / Personeelsbeheer** | 🔴 Nieuw te bouwen | DRG | DRG + toekomstige klanten |
| **Scheduling / Roosters** | 🔴 Nieuw te bouwen | DRG | DRG + toekomstige klanten |

**De eerste twee klanten (Easydash + DRG) vullen de modulecatalogus zo ver aan dat klant 3-10 grotendeels met bestaande modules kunnen worden bediend.**

---

## 11. Kwaliteitsstandaarden

### Overgenomen uit B2B SaaS Infrastructure Skills (productie-gevalideerd)

**Security:**
- Session-based auth met CSRF — geen JWT in localStorage
- PostgreSQL RLS op alle tenant-scoped tabellen
- `tenantId: string | null` — nooit optioneel, nooit undefined
- Webhook signature verificatie (HMAC-SHA256)
- Webhook deduplicatie (svix-id caching)
- Input validatie met Zod op alle endpoints
- Rate limiting op gevoelige endpoints
- Audit logging voor security events
- File uploads: private buckets, signed URLs, MIME whitelist
- Encrypted secrets at rest (AES-256-GCM)

**Code kwaliteit:**
- TypeScript strict mode
- `tsc --noEmit` voor elke commit
- Centralized error codes (`ApiError` + `ErrorCodes`)
- Vitest voor unit tests
- Handmatige test checklists voor integratie-tests (zie TESTING-CHECKLIST.md)

**Database:**
- Drizzle ORM voor schema management
- SQL migraties (niet push) voor productie
- `withTenant()` wrapper voor alle tenant-scoped queries
- Indexes op alle `tenant_id` foreign keys

**Frontend:**
- React 18 + TypeScript + Vite
- shadcn/ui componenten (consistent, accessible)
- TanStack Query voor data fetching
- Geen globale state management (server state via queries)
- Nederlandse UI tekst (doelgroep NL)

**Design:**
- Material Design 3 principes
- Inter font (headings, UI) + JetBrains Mono (code, timestamps)
- Minimal animations (purposeful only)
- WCAG AA contrast

---

## 12. Risico's en Mitigatie

| Risico | Impact | Mitigatie |
|--------|--------|----------|
| Rob uitvalt | Alles stopt | Claude Code kan codebase onderhouden. Simon kan beperkt bijsturen. Modulaire architectuur maakt inwerken junior dev makkelijker. B2B SaaS Infrastructure Skills documenteert alles. |
| AI agents vervangen ons | Business model vervalt | Per-client deployment met eigen data, integratie-complexiteit, en menselijk aanspreekpunt zijn niet automatiseerbaar. Focus op operationeel beheer, niet alleen bouwen. |
| Platform wordt te complex | Nieuwe klanten worden moeilijker | Strikte module-grenzen. Elke module is onafhankelijk testbaar. Config-driven, niet code-driven. |
| Klant wil iets wat niet in modules past | Scope creep | Duidelijk onderscheid: configuratie (goedkoop) vs. nieuwe module (investering die terugvloeit naar platform). Simon managet verwachtingen. |
| Updates breken bestaande klanten | Vertrouwen schaadt | Per-client deployment = per-client rollback. Staging per client. Geen big-bang updates. |

---

## 13. Eerste Stappen (Q1-Q2 2026)

### Nu → Easydash live

1. **Platform repo opzetten** met de architectuur uit dit document
2. **Schakel Core extraheren** uit MAP codebase
3. **Easydash configuratie** opstellen (config.ts)
4. **Tasks, Time Tracking, Invoicing modules** bouwen
5. **Moneybird integratie** bouwen
6. **Easydash deployen** op eigen infra
7. **Feedback verwerken**, modules verfijnen

### Parallel: Schakel Scout bouwen

8. **Scout v1 bouwen** (15-20 uur — zie sectie 7)
9. **schakel-core repo** aanmaken met initiële content (bestaande skills/rules migreren)
10. **Wekelijkse harvest** activeren — elke maandag review van externe inzichten
11. **B2B SaaS Infrastructure Skills opsplitsen** in individuele skills/rules/patterns in schakel-core

### Parallel: MAP en LinkedIn als SaaS

12. **MAP lanceren** op map.schakel.ai (dev → productie merge)
13. **LinkedIn tool lanceren** (testgebruikers zoeken)
14. **Eerste SaaS-inkomsten** genereren (passief)

### Na Easydash: DRG valideren

15. **DRG configuratie** opstellen
16. **HR en Scheduling modules** bouwen
17. **DRG deployen** — als dit significant sneller gaat dan Easydash, werkt het vliegwiel
18. **Module catalogus documenteren** voor Simon's sales gesprekken

### Doorlopend: Kennis systematisch laten compounderen

19. **Scout wekelijkse reviews** (< 15 min/week)
20. **Learnings uit elke build** terugvoeren naar schakel-core via Scout
21. **Dit blueprint** bijwerken na elke klant
22. **CLAUDE.md per project** up-to-date houden
23. **Kwartaalreview**: werkt het model? Wat moet bijgestuurd?

---

## Appendix A: Verklarende woordenlijst

| Term | Betekenis |
|------|-----------|
| **Schakel Core** | Het gedeelde technische fundament (auth, billing, multi-tenancy, RLS, AI) |
| **Module** | Een zelfstandige functionele eenheid (taken, uren, facturatie, etc.) |
| **Operations Center** | Een op maat geconfigureerd systeem voor een MKB-klant |
| **Client Config** | Het configuratiebestand dat bepaalt hoe het platform zich gedraagt voor een klant |
| **B2B SaaS Infrastructure Skills** | Het technische naslagwerk met productie-gevalideerde patronen (10 bestanden in `skills/`) |
| **Schakel Scout** | Interne web-app die schakel-core beheert: kennishub + geautomatiseerde externe harvest |
| **schakel-core** | Private GitHub repo met skills, rules en patterns — de levende kennisbasis |
| **Pak-metafoor** | Standaard patronen, unieke combinaties per klant |
| **Compounding** | Elke build verrijkt het platform, elke volgende build gaat sneller |
| **Managed Operations Partner** | Schakels positionering: niet projecten opleveren, maar systemen beheren |

## Appendix B: Repository structuur

```
schakel-core/
├── CLAUDE.md                         ← AI-assistentinstructies
├── README.md                         ← Wat deze repo is
├── skills/                           ← B2B SaaS Infrastructure Skills (10 bestanden)
│   ├── 00-introduction.md            ← Intro, brownfield guide, architectuuroverzicht
│   ├── 01-project-setup.md           ← §1 Dependencies & config
│   ├── 02-database-schema.md         ← §2 SQL tabellen & indexes
│   ├── ...                           ← §3-27 (zie skills/README.md)
│   └── 09-build-deploy.md            ← §25-27 Build & deploy
├── patterns/
│   └── platform-blueprint.md         ← DIT DOCUMENT
├── rules/
│   ├── code-standards.md             ← Code conventies
│   ├── security-checklist.md         ← Security vereisten
│   └── design-system.md              ← Design richtlijnen
├── context/
│   ├── founders-brief.md             ← Wie we zijn
│   └── vision.md                     ← Strategische visie
└── harvest/
    └── radar.md                      ← Externe scan-configuratie
```
