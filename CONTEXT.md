# CONTEXT.md — schakel-core

> Huidige status, planning en to-do's voor deze repository.
> Laatst bijgewerkt: 18 februari 2026

---

## Status

schakel-core is operationeel als kennisbank. Alle Schakel-projecten (MAP, Easydash, DRG, LinkedIn-tool) kunnen deze repo raadplegen voor skills, patterns en rules.

### Wat er is

| Onderdeel | Status | Toelichting |
|-----------|--------|-------------|
| **Skills** (10 bestanden) | Compleet | B2B SaaS Infrastructure Skills v2.8, opgesplitst in `00-introduction` t/m `09-build-deploy` |
| **Patterns** | Eerste versie | `platform-blueprint.md` v0.2 — architectuur + Scout concept |
| **Rules** | Compleet | `code-standards.md`, `security-checklist.md`, `design-system.md` |
| **Context** | Compleet | `founders-brief.md`, `vision.md` |
| **Harvest** | Alleen config | `radar.md` met scan-gebieden, nog geen output |

### Wat er nog niet is

- **Schakel Scout** — de web-app die schakel-core beheert (kennishub + geautomatiseerde harvest). Ontwerp staat in `patterns/platform-blueprint.md` sectie 7. Nog niet gebouwd.
- **Harvest output** — `harvest/` bevat alleen de radar config, geen scan-resultaten
- **Individuele skill-bestanden** — de 10 bestanden zijn secties uit één document; op termijn worden ze verder verfijnd naar zelfstandige skills
- **CI/CD** — geen automatische checks, linting of validatie op deze repo

---

## Recente wijzigingen

| Datum | Wijziging |
|-------|-----------|
| 18 feb 2026 | Skills opgesplitst van 1 monoliet (276KB) naar 10 bestanden |
| 18 feb 2026 | Stale referenties opgeschoond (B2B-SAAS-SKILLS → correct, passport.js → express-session, MAP-specifieke voorbeelden → generiek) |
| 17 feb 2026 | Schakel Scout geïntegreerd in platform-blueprint (v0.2) |
| 17 feb 2026 | Repo geherstructureerd als schakel-core (skills/, patterns/, rules/, context/, harvest/) |

---

## To-do's

### Hoge prioriteit

- [ ] Skills inhoudelijk reviewen — zijn alle 10 bestanden intern consistent na de split?
- [ ] Cross-referenties tussen skill-bestanden controleren (§-verwijzingen kloppen ze nog?)
- [ ] `platform-blueprint.md` sectie 13 bijwerken — "schakel-core repo aanmaken" en "Skills opsplitsen" zijn nu gedaan

### Gemiddelde prioriteit

- [ ] Harvest workflow voor het eerst draaien — scan op gebieden uit `radar.md`, resultaten naar `harvest/`
- [ ] Skills verder verfijnen naar zelfstandige documenten (niet alleen secties uit een monoliet)
- [ ] Pattern toevoegen: module-ontwerp (hoe een nieuwe module te bouwen binnen het platform)

### Lage prioriteit

- [ ] Scout v1 bouwen (15-20 uur, zie blueprint sectie 7)
- [ ] design-system.md verrijken met meer component-specifieke richtlijnen
- [ ] Automatische checks toevoegen (bijv. broken link checker op markdown)

---

## Beslissingen

| Beslissing | Datum | Reden |
|------------|-------|-------|
| Skills in 10 bestanden, niet 27 | 18 feb 2026 | 27 zou te fragmentarisch zijn, 1 was te groot. 10 gegroepeerd op thema is de sweet spot. |
| `CONTEXT.md` op root-niveau | 18 feb 2026 | Elke repo heeft een CONTEXT.md voor status/planning. Naast CLAUDE.md (instructies) en README.md (introductie). |
| Nederlands voor documenten, Engels voor code | 17 feb 2026 | Doelgroep is Nederlands. Code is universeel. |
| Geen Scout web-app (nog) | 17 feb 2026 | Handmatig beheer via Claude Code + PRs is voldoende zolang schakel-core klein is. |

---

*Dit bestand wordt bijgewerkt bij elke significante wijziging aan deze repo.*
