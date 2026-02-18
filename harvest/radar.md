# Schakel Radar

> Configuratie voor de wekelijkse knowledge harvest. Dit bestand definieert **breed** waar we scannen — niet wat we zoeken. Het doel is ontdekken wat we niet weten dat we niet weten.

## Wie we zijn (context voor relevantie-filter)

Schakel AI — twee-mans B2B agency die Managed Operations Centers bouwt voor het MKB.
Stack: React 18, TypeScript, Express, PostgreSQL (Drizzle ORM), shadcn/ui, TailwindCSS.
Business: Per-client Operations Centers + SaaS products (MAP, LinkedIn tool).
Builder: Rob + Claude Code (Opus 4.6). Non-traditional developer, AI-first workflow.

## Scan deze gebieden breed

### AI-Assisted Development
- Claude Code: workflows, skills, hooks, MCP servers, CLAUDE.md best practices
- Hoe andere builders Claude Code gebruiken in productie
- Codex, Devin, en andere AI coding agents: wat kunnen ze, waar falen ze
- Cursor, Windsurf, Replit Agent: welke workflows adopteren power users
- AI-first development patterns: hoe verandert software development als AI 80% schrijft
- Claude Code extensions, custom skills, automation patterns
- Prompt engineering voor code generation: wat werkt het beste

### B2B SaaS Architecture
- Multi-tenant architectuur patterns (per-tenant deploy vs shared infra)
- Module-based platform architectuur (hoe bouw je een configurable platform)
- Nieuwe aanpakken voor auth, billing, en access control
- Edge computing, serverless, en nieuwe hosting paradigma's
- Database patterns: RLS updates, Drizzle ORM nieuws, PostgreSQL features
- API design: tRPC, GraphQL vs REST trends voor B2B

### Onze Stack (React + Node + PostgreSQL)
- React 19+: nieuwe features, server components adoptie
- shadcn/ui: nieuwe componenten, community patterns
- TailwindCSS: v4 updates, nieuwe utilities
- Drizzle ORM: nieuwe features, migration patterns
- Express vs alternatives (Hono, Elysia, Fastify): benchmarks en DX
- Vite: nieuwe features, build optimalisatie
- TypeScript: nieuwe features die onze code verbeteren

### MKB Operations & Business
- Wat MKB-bedrijven daadwerkelijk nodig hebben van software
- Concurrentie: wat doen Odoo, Monday, ClickUp, Notion nieuw
- Nederlandse markt: compliance, AVG/GDPR updates, KvK integraties
- Integratie-landschap: Moneybird API updates, Microsoft Graph changes, Stripe nieuws
- Pricing models voor managed services: wat werkt

### Knowledge Management & Compounding
- Hoe andere solo-developers/small teams hun kennis organiseren
- Second brain systemen voor developers
- Documentation-as-code patterns
- Hoe AI-first teams hun learnings systematiseren

## Filter op relevantie

Bij elke finding, stel deze vragen:
1. **Is dit ACTIONABLE voor ons?** — Kunnen we dit toepassen in een concrete build?
2. **Is dit NIEUW?** — Weten we dit al? Check bestaande skills/patterns/rules.
3. **Past dit bij onze SCHAAL?** — We zijn 2 mensen, niet een enterprise team.
4. **Verbetert dit onze COMPOUNDING?** — Maakt dit de volgende build sneller/beter?

## Niet interessant

- Pure enterprise/Fortune 500 oplossingen (we zijn MKB-gericht)
- Talen/stacks die we niet gebruiken (Python ML, Rust, Go, Java, etc.)
- Theoretische AI research zonder praktische toepassing
- Crypto/Web3 (niet relevant voor onze business)
- Mobile app development (we bouwen web apps)

## Output format

Bevindingen worden als markdown files in `harvest/` geplaatst:
```
harvest/
├── 2026-02-24-claude-code-hooks-patterns.md
├── 2026-02-24-drizzle-orm-v0.35-features.md
└── 2026-02-24-shadcn-new-components.md
```

Na review worden ze gepromoveerd naar skills/, patterns/, of rules/.
