# Skills

> Actionable how-to kennis. Gebruik on-demand wanneer je een specifieke feature bouwt.

## B2B SaaS Infrastructure Skills

Productie-gevalideerde patronen voor het bouwen van multi-tenant B2B SaaS applicaties. Opgesplitst in 10 bestanden, georganiseerd per thema.

### Overzicht

| Bestand | Secties | Onderwerp |
|---------|---------|-----------|
| [`00-introduction.md`](00-introduction.md) | Intro, Brownfield, Visie | Hoe dit te gebruiken, migratiegidsen, architectuuroverzicht |
| [`01-project-setup.md`](01-project-setup.md) | §1 | Dependencies, project structuur, Vite/Drizzle/Tailwind config |
| [`02-database-schema.md`](02-database-schema.md) | §2 | Alle SQL tabellen, indexes, brownfield migratie |
| [`03-shared-types.md`](03-shared-types.md) | §3 | Drizzle definities, Zod schemas, TypeScript types |
| [`04-storage-interface.md`](04-storage-interface.md) | §4 | IStorage interface, MemStorage, SupabaseStorage |
| [`05-multi-tenancy.md`](05-multi-tenancy.md) | §5-6 | App-layer multi-tenancy + PostgreSQL Row Level Security |
| [`06-auth-security.md`](06-auth-security.md) | §7-12 | Session auth, CSRF, MFA, rate limiting, impersonation |
| [`07-billing-onboarding.md`](07-billing-onboarding.md) | §13-17 | Invitations, tenant provisioning, Stripe billing, tier limits |
| [`08-platform-quality.md`](08-platform-quality.md) | §18-24 | Error handling, audit logging, email, OAuth, webhooks, GDPR, queues |
| [`09-build-deploy.md`](09-build-deploy.md) | §25-27 | Dev/prod server, build config, implementatie checklist |

### Hoe te gebruiken

1. Je bouwt een feature (bijv. Stripe billing)
2. Open het relevante bestand (bijv. `07-billing-onboarding.md`, §15)
3. Lees eerst het **Principes-blok** — dit zijn non-negotiables
4. Gebruik de referentie-implementatie als startpunt
5. Pas aan waar nodig, maar respecteer altijd de principes

### Principes vs. Implementatie

Elke sectie bevat optioneel:
> **Principes (niet-onderhandelbaar)**

Dit zijn invarianten die ALTIJD moeten gelden. Alles buiten een Principes-blok is een referentie-implementatie — één bewezen manier om het principe te realiseren.

## Toekomstige skills

Naarmate we meer bouwen, worden individuele skills toegevoegd:
- `webhook-patterns.md` — Specifieke webhook handling patterns
- `calendar-integration.md` — Microsoft 365 kalender integratie
- etc.

---

*Versie: 2.8 — Oorspronkelijk één document (B2B-SAAS-INFRASTRUCTURE.md), opgesplitst in februari 2026.*
