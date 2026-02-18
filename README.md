## Schakel Core

### Wat is dit?

schakel-core is de centrale kennisbank van Schakel AI. Alles wat we weten over het bouwen van B2B SaaS platformen — productie-gevalideerde skills, architectuurpatronen, codeerstandaarden, en strategische context — leeft hier.

Dit is geen codebase. Dit is kennis. Elke Schakel-project (MAP, Easydash, DRG, LinkedIn tool) trekt hieruit. Elke build voedt het terug.

### Waarom bestaat dit?

**Compounding knowledge.** Elke klant die we bouwen maakt de volgende sneller. Maar alleen als we wat we leren ook vastleggen. Deze repo is het mechanisme dat dat mogelijk maakt.

Zonder schakel-core is kennis ad-hoc — verspreid over chats, commit messages, en hoofden. Met schakel-core is het gecentraliseerd, geversioned, doorzoekbaar, en direct toepasbaar.

### Structuur

```
schakel-core/
├── CLAUDE.md          # AI assistant instructies
├── CONTEXT.md         # Status, planning, to-do's
├── README.md          # Dit bestand
├── skills/            # Hoe je iets bouwt (on-demand)
│   ├── 00-introduction.md           # Intro, brownfield guide, architectuuroverzicht
│   ├── 01-project-setup.md          # §1 Dependencies & config
│   ├── 02-database-schema.md        # §2 SQL tabellen & indexes
│   ├── 03-shared-types.md           # §3 TypeScript types & Zod schemas
│   ├── 04-storage-interface.md      # §4 IStorage pattern
│   ├── 05-multi-tenancy.md          # §5-6 Multi-tenancy & RLS
│   ├── 06-auth-security.md          # §7-12 Auth, CSRF, MFA
│   ├── 07-billing-onboarding.md     # §13-17 Billing & onboarding
│   ├── 08-platform-quality.md       # §18-24 Error handling, audit, GDPR
│   └── 09-build-deploy.md           # §25-27 Build & deploy
│
├── patterns/          # Architectuur referenties
│   └── platform-blueprint.md        # Schakel Platform Blueprint
│
├── rules/             # Standaarden (altijd actief)
│   ├── code-standards.md             # Code conventies
│   ├── security-checklist.md         # Security vereisten
│   └── design-system.md             # Design richtlijnen
│
├── context/           # Wie we zijn, visie, strategie
│   ├── founders-brief.md            # Founders Context Brief
│   └── vision.md                    # Van AI Agency naar Managed Operations Partner
│
└── harvest/           # Radar scan output (drafts)
    └── radar.md               # Externe scan-configuratie
```

### Drie kennistypes

| Type | Wanneer | Voorbeeld |
|------|---------|-----------|
| **Skills** | On-demand — als je een feature bouwt | "Hoe implementeer ik session auth met CSRF?" |
| **Patterns** | Bij architectuur beslissingen | "Hoe is het Schakel Platform gestructureerd?" |
| **Rules** | Altijd — in elk project | "Welke security checks zijn verplicht?" |

### Hoe gebruik je dit?

**Als AI-assistent (Claude Code):**
```
# Voordat je een feature bouwt:
"Lees skills/06-auth-security.md voor de referentie-implementatie van auth"

# Bij elke build:
"Volg rules/code-standards.md en rules/security-checklist.md"

# Bij architectuur-keuzes:
"Refereer aan patterns/platform-blueprint.md"
```

**Na een build — kennis terugpushen:**
```
"Push dit als nieuw pattern naar schakel-core"
"Update het relevante skills-bestand met deze learning"
"Voeg dit toe aan rules/security-checklist.md"
```

### De Harvest Workflow

schakel-core groeit niet alleen door eigen builds, maar ook door externe kennis. `harvest/radar.md` definieert brede gebieden om te scannen (Claude Code ecosystem, AI coding tools, B2B SaaS patterns, etc.).

**Wekelijks:**
1. Claude Code scant breed op de gebieden in harvest/radar.md
2. Vergelijkt met bestaande kennis
3. Stelt nieuwe/bijgewerkte content voor via PRs
4. Rob reviewed en merged

### Wie we zijn

Schakel AI B.V. — Rob (builder) & Simon (verkoper). We bouwen Managed Operations Centers voor het MKB. Elk systeem ziet eruit alsof het speciaal voor die klant is gebouwd, maar onder de motorkap deelt het een bewezen, modulair platform.

Lees `context/founders-brief.md` en `context/vision.md` voor het volledige verhaal.

---

*Dit is een levend document. Het wordt beter met elke build.*
