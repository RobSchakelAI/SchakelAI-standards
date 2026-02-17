# Skills

> Actionable how-to kennis. Gebruik on-demand wanneer je een specifieke feature bouwt.

## Hoofddocument

**`B2B-SAAS-INFRASTRUCTURE.md`** — Het complete technische naslagwerk (276KB, 27 secties). Bevat productie-gevalideerde patronen voor het bouwen van multi-tenant B2B SaaS applicaties.

### Secties in B2B-SAAS-INFRASTRUCTURE.md

| Deel | Secties | Onderwerp |
|------|---------|-----------|
| **A: Fundament** | §1-6 | Project setup, database schema, shared types, storage interface, multi-tenancy, RLS |
| **B: Auth & Security** | §7-12 | Session auth + CSRF, registration, frontend auth, password security, MFA, rate limiting, impersonation |
| **C: Billing & Onboarding** | §13-17 | Invitations, tenant provisioning, Stripe billing, subscription access, tier limits |
| **D: Platform Kwaliteit** | §18-24 | Error handling, audit logging, email service, OAuth encryption, webhooks, GDPR, queue processing |
| **E: Build & Deploy** | §25-27 | Dev/prod server setup, build config, deployment, implementatie checklist |

### Hoe te gebruiken

1. Je bouwt een feature (bijv. Stripe billing)
2. Open `B2B-SAAS-INFRASTRUCTURE.md`, navigeer naar §15 (Stripe Billing)
3. Lees eerst het **Principes-blok** — dit zijn non-negotiables
4. Gebruik de referentie-implementatie als startpunt
5. Pas aan waar nodig, maar respecteer altijd de principes

### Principes vs. Implementatie

Elke sectie bevat optioneel:
> **Principes (niet-onderhandelbaar)**

Dit zijn invarianten die ALTIJD moeten gelden. Alles buiten een Principes-blok is een referentie-implementatie — één bewezen manier om het principe te realiseren.

## Toekomstige skills

Naarmate we meer bouwen, worden individuele skills toegevoegd naast het hoofddocument:
- `webhook-patterns.md` — Specifieke webhook handling patterns
- `calendar-integration.md` — Microsoft 365 kalender integratie
- etc.
