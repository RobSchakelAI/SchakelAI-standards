# Hoe We Bouwen — De Schakel Ontwikkelmethode

> Het systeem achter het systeem: waarom schakel-core bestaat, hoe de multi-repo architectuur werkt, en hoe kennis compound.
> Schakel AI — Februari 2026

---

## Leeswijzer

Dit document is het complement van `vision.md` (waar we naartoe gaan) en `founders-brief.md` (wie er aan het stuur zit). Dit document beschrijft **hoe** we bouwen: de architectuur van onze ontwikkelmethode, niet van onze software.

Lees dit als je wilt begrijpen:
- Waarom schakel-core bestaat
- Hoe project-repo's samenwerken met schakel-core
- Hoe kennis stroomt tussen projecten
- Wat de rol van Claude Code is in ons bouwproces

---

## 1. Het kernprobleem

Elke build levert kennis op. Hoe je auth implementeert. Hoe je multi-tenancy opzet. Hoe je Stripe integreert. Maar zonder systeem om die kennis vast te leggen, is het weg zodra de context van een chat-sessie sluit.

Rob bouwt MAP en leert hoe session-based auth werkt met CSRF-tokens. Twee weken later bouwt hij Easydash en moet hij het opnieuw uitzoeken. Dat is verspilling. Niet van tijd — maar van **compounding**. Elke build zou sneller moeten zijn dan de vorige, niet even snel.

## 2. De oplossing: schakel-core

schakel-core is geen codebase. Het is een kennisbank. Alles wat we weten over het bouwen van B2B SaaS platformen — gevalideerd in productie — leeft hier:

- **Skills** — Hoe je iets bouwt (on-demand, pull wanneer relevant)
- **Patterns** — Hoe je iets ontwerpt (bij architectuurbeslissingen)
- **Rules** — Hoe je code schrijft (altijd actief, niet-onderhandelbaar)
- **Context** — Wie we zijn en waarom we doen wat we doen

De kracht zit niet in de individuele documenten, maar in het systeem: bouwen → leren → vastleggen → hergebruiken → sneller bouwen. Dat is het vliegwiel.

## 3. De multi-repo architectuur

Elk project heeft zijn eigen repository. schakel-core wordt toegevoegd als externe referentie via `--add-dir`:

```
┌─────────────────────────┐      ┌─────────────────────────┐
│  Project repo            │      │  schakel-core            │
│  (bijv. Easydash)        │      │  (gedeelde kennisbank)   │
│                          │      │                          │
│  CLAUDE.md    ← WAT      │◄────►│  CLAUDE.md    ← HOE     │
│  CONTEXT.md   ← status   │      │  CONTEXT.md   ← status  │
│  src/                    │      │  skills/                 │
│  ...                     │      │  patterns/               │
│                          │      │  rules/                  │
│                          │      │  context/                │
└─────────────────────────┘      └─────────────────────────┘

Claude Code leest BEIDE CLAUDE.md bestanden.
Het project bepaalt WAT, schakel-core bepaalt HOE.
```

### Hoe de bestanden samenwerken

| Bestand | In schakel-core | In project repo |
|---------|----------------|-----------------|
| `CLAUDE.md` | **HOE** — tech stack, principes, regels, samenwerkingsmodel, kennistypes | **WAT** — dit project, deze features, deze endpoints, deze specifieke configuratie |
| `CONTEXT.md` | Status van de kennisbank — wat er is, wat er mist, planning | Status van het project — sprint, open bugs, deployment status |
| `README.md` | Introductie tot schakel-core zelf | Introductie tot het project |

De scheiding is bewust. schakel-core verandert langzaam (nieuwe skills, bijgewerkte regels). Project repos veranderen snel (features, bugfixes, deploys). Dezelfde bestandsnamen, verschillende scope, allebei automatisch gelezen door Claude Code.

### Naamconventie

Beide heten `CLAUDE.md` en `CONTEXT.md`. Het verschil zit in de **header op regel 1**:

```markdown
# CLAUDE.md — schakel-core         ← shared methodiek
# CLAUDE.md — Easydash             ← project-specifiek
# CONTEXT.md — schakel-core        ← kennisbank status
# CONTEXT.md — Easydash            ← project status
```

Geen hernoeming nodig. Claude Code pikt beide automatisch op via `--add-dir`.

## 4. Het kennisstroommodel

```
Bouwen (project repo)
    │
    ▼
Leren (tijdens de build)
    │
    ▼
Vastleggen (push naar schakel-core)
    │
    ▼
Hergebruiken (volgende project haalt het op)
    │
    ▼
Sneller bouwen
    │
    └──► terug naar Bouwen
```

### Concreet voorbeeld

1. **MAP**: Rob bouwt session-based auth met CSRF-tokens. Kost 3 dagen trial-and-error.
2. **Vastleggen**: De werkende aanpak wordt gedocumenteerd in `skills/06-auth-security.md`.
3. **Easydash**: Rob zegt "bouw auth". Claude leest de skill. Auth werkt in 2 uur.
4. **Verbetering**: Easydash heeft multi-role auth nodig. De skill wordt bijgewerkt.
5. **DRG**: Auth + roles in 1 uur, inclusief de multi-role variant.

Klant 1 kost maanden. Klant 5 kost weken. Klant 10 kost dagen. Dat is compound interest op kennis.

## 5. De rol van Claude Code

Claude Code is geen tool — het is de mede-developer. Het model werkt het beste als het context heeft. schakel-core is die context.

### Hoe Claude schakel-core gebruikt

| Situatie | Wat Claude doet |
|----------|----------------|
| Nieuwe feature bouwen | Checkt `skills/` voor bestaande kennis |
| Architectuurbeslissing | Raadpleegt `patterns/` |
| Elke regel code | Past `rules/` toe (altijd actief) |
| Begrijpen wie de klant is | Leest `context/` |

### Het samenwerkingsmodel

- **Rob bepaalt WAT** — richting, prioriteiten, scope
- **Claude implementeert, denkt mee, en waarschuwt** — code schrijven, problemen signaleren, alternatieven voorstellen
- **Bij twijfel: vraag** — liever één vraag te veel dan een verkeerde aanname
- **Kennis vloeit terug** — wat we leren wordt vastgelegd

## 6. Een nieuw project opstarten

Wanneer er een nieuw project begint (bijv. een nieuwe klant):

### Stap 1: Repo aanmaken
```
project-naam/
├── CLAUDE.md       # Project-specifieke instructies
├── CONTEXT.md      # Project status en planning
├── src/            # Code
└── ...
```

### Stap 2: schakel-core koppelen
```bash
claude --add-dir /pad/naar/schakel-core
```

### Stap 3: Claude oriënteert zich
Claude leest automatisch beide CLAUDE.md bestanden en heeft nu:
- De gedeelde methodiek, stack, en regels (uit schakel-core)
- De project-specifieke context (uit de project repo)
- Toegang tot alle skills, patterns, en rules

### Stap 4: Bouwen
Vraag Claude wat er gebouwd moet worden. Claude trekt de juiste skills in wanneer nodig. Rules zijn altijd actief.

### Stap 5: Terugpushen
Na de build: push geleerde lessen terug naar schakel-core.
```
"Push dit als nieuw pattern naar schakel-core"
"Update skills/06-auth-security.md met deze verbetering"
```

## 7. Waarom dit werkt

### Het lost drie problemen op

1. **Kennisverval** — Zonder systeem verdwijnt kennis na elke sessie. Met schakel-core is het permanent.
2. **Inconsistentie** — Zonder gedeelde regels bouwt elk project anders. Met schakel-core is de kwaliteit uniform.
3. **Snelheid** — Zonder hergebruik begint elk project bij nul. Met schakel-core begint elk project waar het vorige stopte.

### Het schaalt met ons mee

- 1 project: schakel-core is een geheugensteun
- 5 projecten: schakel-core is een versneller
- 10 projecten: schakel-core is een concurrentievoordeel

De investering in schakel-core betaalt zichzelf terug bij elke nieuwe klant. Dat is de definitie van compound interest.

---

*Dit document beschrijft het systeem zoals het nu werkt. Het evolueert mee met onze ervaring.*
