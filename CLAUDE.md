# CLAUDE.md — schakel-core

> Instructies voor AI-assistenten die werken met deze repository en met alle Schakel AI-projecten.

## Wat is schakel-core?

schakel-core is de centrale kennisbank van Schakel AI. Geen applicatiecode — alleen kennis: productie-bewezen skills, architectuurpatronen, codeerregels en strategische context.

Elk Schakel-project (MAP, Easydash, DRG, LinkedIn-tool, etc.) haalt kennis uit deze repo. Elke build pusht geleerde lessen terug. Dit is het vliegwiel: bouwen, leren, vastleggen, hergebruiken.

## Repositorystructuur

```
schakel-core/
├── CLAUDE.md          # Dit bestand — AI-assistentinstructies
├── CONTEXT.md         # Status, planning, to-do's van deze repo
├── README.md          # Wat deze repo is, waarom, hoe te gebruiken
├── skills/            # Actionable how-to kennis (on-demand, pull wanneer nodig)
├── patterns/          # Architectuur-referentiedocumenten (blauwdrukken, ontwerpen)
├── rules/             # Altijd-actieve standaarden (verplicht in elk project)
├── context/           # Wie we zijn, visie, strategie, businessmodel
└── harvest/           # Output van radar-scans (concepten voor promotie)
    └── radar.md       # Brede scanning-config voor externe kennisoogst
```

## Drie kennistypes

### 1. Skills (on-demand)

Technische how-to-documenten. Worden opgehaald wanneer je een specifieke feature bouwt.

Voorbeelden:
- "Hoe implementeer je session auth met CSRF"
- "Hoe zet je Stripe billing op"
- "Hoe configureer je multi-tenant RLS in PostgreSQL"

De skills zijn opgesplitst in 10 bestanden in `skills/` — van project setup tot deployment. Zie `skills/README.md` voor het overzicht.

**Wanneer raadplegen:** Voor je een feature implementeert, controleer of er een skill voor bestaat.

### 2. Patterns (referentie)

Architectuurblauwdrukken. Worden gebruikt bij planning en ontwerpbeslissingen.

Voorbeelden:
- "Hoe het Schakel Platform is opgebouwd"
- "Hoe modules zijn ontworpen"
- "Hoe de multi-tenant architectuur werkt"

**Wanneer raadplegen:** Bij architectuurbeslissingen, nieuwe modules, of systeemontwerp.

### 3. Rules (altijd actief)

Standaarden die in ELK project gevolgd moeten worden. Niet-onderhandelbaar.

Voorbeelden:
- Codeerconventies en naamgeving
- Beveiligingseisen
- Design system richtlijnen

**Wanneer raadplegen:** Altijd. Deze regels zijn verplicht bij elke code-wijziging.

## Hoe we samenwerken

Rob is de bouwer. Claude is de mede-developer. We werken als een team:

- **Rob bepaalt WAT** er gebouwd wordt — richting, prioriteiten, scope
- **Claude implementeert, denkt mee, en waarschuwt** — code schrijven, problemen signaleren, alternatieven voorstellen
- **Kennis vloeit terug** — wat we leren tijdens een build wordt vastgelegd in schakel-core

Dit is geen eenrichtingsverkeer. Claude mag (en moet) terugduwen als iets de principes schendt, als er een betere aanpak is, of als er risico's zijn die Rob misschien niet ziet. Maar Rob beslist.

**Bij twijfel: vraag.** Als iets niet 100% duidelijk is — requirements, scope, prioriteit, een technische keuze — vraag het aan Rob. Liever één vraag te veel dan een verkeerde aanname die later terugkomt. Ga niet gokken.

## Multi-repo architectuur

schakel-core wordt als `--add-dir` gekoppeld aan project-repo's. Elk project heeft zijn eigen `CLAUDE.md` en `CONTEXT.md`. Claude Code leest **beide** automatisch.

```
Project repo (bijv. Easydash)     schakel-core (--add-dir)
├── CLAUDE.md    ← WAT            ├── CLAUDE.md    ← HOE
├── CONTEXT.md   ← projectstatus  ├── CONTEXT.md   ← kennisbankstatus
└── src/                          ├── skills/
                                  ├── rules/
                                  ├── patterns/
                                  └── context/
```

**Naamconventie:** Beide heten `CLAUDE.md` en `CONTEXT.md`. Het verschil zit in de header op regel 1:
- schakel-core: `# CLAUDE.md — schakel-core` (HOE — methodiek, stack, regels)
- Project: `# CLAUDE.md — Easydash` (WAT — dit project, deze features)

Zie `context/how-we-build.md` voor het volledige verhaal achter deze architectuur.

## Nieuw project opstarten

Wanneer schakel-core is toegevoegd aan een project (via `--add-dir` of anderszins):

**Stap 1: Oriënteer je** (doe dit automatisch)
- Dit bestand (CLAUDE.md) geeft je: tech stack, principes, taalconventies, regels
- Lees `context/founders-brief.md` — wie Rob en Simon zijn, hoe Schakel denkt
- Lees `CONTEXT.md` — huidige status van de kennisbank

**Stap 2: Vraag wat we bouwen**
- Implementeer NIET automatisch alle skills. Skills zijn een referentiebibliotheek, geen checklist.
- Vraag: "Wat bouwen we? Welke features zijn er nodig?"
- Trek pas skill-bestanden in wanneer ze relevant zijn voor wat er gebouwd wordt

**Stap 3: Werk volgens de regels**
- `rules/` is ALTIJD actief — codeerstandaarden, security, design system
- `skills/` is ON-DEMAND — raadpleeg wanneer je een specifieke feature implementeert
- `patterns/` is voor ARCHITECTUUR — raadpleeg bij ontwerpbeslissingen

**Voorbeeld:**
Rob zegt: "We bouwen een dashboard met auth en Stripe billing."
→ Lees `skills/06-auth-security.md` voor auth patterns
→ Lees `skills/07-billing-onboarding.md` voor Stripe
→ Volg altijd `rules/security-checklist.md` en `rules/code-standards.md`
→ NIET automatisch §24 (queue processing) of §12 (impersonation) implementeren — die zijn er als je ze nodig hebt

## Hoe bij te dragen (terugpushen naar schakel-core)

Na het bouwen van iets in een project, push geleerde lessen terug:

| Wat je hebt geleerd | Waar het naartoe gaat | Voorbeeld |
|---|---|---|
| Nieuwe technische aanpak | `skills/` | "Hoe recall.ai webhooks te verwerken" |
| Nieuw architectuurpatroon | `patterns/` | "Event-driven module communicatie" |
| Nieuwe of bijgewerkte standaard | `rules/` | "API error response format" |
| Strategische inzichten | `context/` | "Marktpositionering update" |

**Trigger-commando:** Gebruik in een Schakel-project de instructie:

```
"Push dit naar schakel-core als [skill/pattern/rule]"
```

Dit signaleert dat kennis uit het huidige project vastgelegd moet worden in schakel-core.

## Harvest workflow

De `harvest/radar.md` definieert brede gebieden om externe kennis te scannen. De harvest-workflow werkt als volgt:

1. **Scan**: Claude Code + WebSearch onderzoekt gebieden uit `harvest/radar.md`
2. **Vergelijk**: Resultaten worden vergeleken met bestaande kennis in schakel-core
3. **Concept**: Nieuwe of bijgewerkte content wordt naar `harvest/` geschreven
4. **Review**: Concepten worden beoordeeld op relevantie en kwaliteit
5. **Promotie**: Goedgekeurde content wordt verplaatst naar `skills/`, `patterns/`, of `rules/`
6. **PR**: Wijzigingen worden voorgesteld via pull requests

## Tech stack (productie-gevalideerd)

Deze stack is bewezen in productie en is de standaard voor alle Schakel-projecten:

### Frontend
- **React 18** met **TypeScript**
- **Vite** als build tool
- **TailwindCSS** voor styling
- **shadcn/ui** als componentenbibliotheek

### Backend
- **Node.js** met **Express** en **TypeScript**
- RESTful API-architectuur

### Database
- **PostgreSQL** (via Supabase of Neon)
- **Drizzle ORM** voor type-safe queries
- **Row Level Security (RLS)** voor tenant-isolatie op databaseniveau

### Authenticatie
- **Session-based** (cookies + CSRF tokens)
- **NOOIT** JWT in localStorage — dit is een harde regel

### AI
- **Anthropic Claude API** voor alle AI-functionaliteit

### Betalingen
- **Stripe** voor abonnementen en facturatie

### Integraties
- **Microsoft 365** (Graph API) — email, agenda, documenten
- **Moneybird** — boekhouding
- **Recall.ai** — meeting transcripties
- **Productive.io** — projectmanagement

### Hosting
- **Railway** — backend services
- **Vercel** — frontend deployments
- **Supabase** — database en auth-services

### Development
- **Claude Code** — AI-assisted development
- **GitHub** — versiebeheer en CI/CD

## Taalconventies

| Context | Taal | Voorbeeld |
|---|---|---|
| Code (variabelen, functies, types) | Engels | `getUserById`, `TenantConfig` |
| UI-tekst en foutmeldingen | Nederlands | `"Sessie verlopen, log opnieuw in"` |
| Commentaar in code | Engels | `// Validate tenant access before query` |
| Database-kolommen | Engels, snake_case | `tenant_id`, `created_at` |
| Documenten in deze repo | Nederlands | (behalve codevoorbeelden) |
| Commit messages | Engels | `fix: prevent tenant data leak in RLS policy` |

## Niet-onderhandelbare principes

Deze regels gelden altijd. Geen uitzonderingen. Geen "we fixen het later".

### Authenticatie
- **Session-based auth met cookies en CSRF tokens**
- NOOIT JWT in localStorage
- NOOIT tokens in URL-parameters
- Secure, HttpOnly, SameSite=Strict cookies

### Multi-tenant isolatie (defense in depth)

Twee lagen, altijd beide actief:

1. **Applicatielaag**: `tenantId` is verplicht bij elke database-query
2. **Databaselaag**: PostgreSQL RLS-policies als vangnet

```typescript
// tenantId typing — NOOIT optional
tenantId: string | null
// string = normale tenant-operatie
// null   = expliciete superadmin-bypass (gelogd en geaudit)
// NOOIT undefined, NOOIT optional (?)
```

### Data-encryptie
- API-keys en OAuth-tokens: **encrypted at rest**
- Gebruik applicatie-level encryptie, niet alleen database-encryptie
- Encryptiesleutels gescheiden van data opslaan

### Inputvalidatie
- **Zod-validatie op alle user input** — geen uitzonderingen
- Valideer op de grens (API-endpoint), niet diep in de businesslogica
- Valideer zowel request body als query parameters

### Audit logging
- Log alle beveiligingsgebeurtenissen: login, logout, failed attempts
- Log alle tenant-boundary crossings
- Log alle superadmin-acties (waar `tenantId = null`)
- Logs zijn append-only, nooit verwijderen

### Elke laag valideert zelfstandig
- Frontend valideert voor UX
- API-laag valideert voor security
- Database valideert via constraints en RLS
- Vertrouw nooit op validatie van een andere laag

## Snelle referentie voor AI-assistenten

Wanneer je werkt aan een Schakel-project, gebruik dit als checklist:

```
[ ] CONTEXT.md gelezen voor huidige status en planning?
[ ] Skills gecheckt voor bestaande kennis?
[ ] Rules toegepast (auth, multi-tenant, validatie)?
[ ] Patterns geraadpleegd voor architectuurbeslissingen?
[ ] Code in het Engels, UI in het Nederlands?
[ ] tenantId: string | null (nooit optional)?
[ ] Zod-validatie op alle input?
[ ] Session-based auth (geen JWT in localStorage)?
[ ] RLS-policies aanwezig voor nieuwe tabellen?
[ ] Audit logging voor security events?
[ ] Geleerde lessen terugpushen naar schakel-core?
```
