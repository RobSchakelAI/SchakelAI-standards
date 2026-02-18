# Code Standards — Schakel AI

> Conventies die gelden in elk Schakel-project. Consistentie maakt compounding mogelijk.

## Taal & Naamgeving

- **Code** (variabelen, functies, types): **Engels**. `tenantId`, `getItem()`, `SessionUser`.
- **Error messages en UI teksten**: **Nederlands** (doelgroep). "Niet geauthenticeerd", "Opslaan gelukt".
- **Comments in code**: Engels (zodat elke developer het leest).
- **Database kolommen**: Engels, snake_case. `tenant_id`, `created_at`, `password_hash`.
- **Bestanden**: kebab-case voor componenten (`meeting-detail.tsx`), camelCase voor utilities (`queryClient.ts`).

## TypeScript

- **Strict mode altijd aan**: `"strict": true` in tsconfig.json.
- **Geen `any`**: Gebruik `unknown` als type onbekend is, narrowing voor type-checking.
- **Gebruik Zod voor validatie**: Alle user input, API responses, en form data via Zod schemas.
- **Exporteer types vanuit shared/schema.ts**: Eén source of truth voor types tussen frontend en backend.
- **`tenantId: string | null`**: Nooit optioneel. `null` = bewuste superadmin bypass.

## React

- **Functionele componenten**: Geen class components.
- **React Query voor server state**: `useQuery` voor reads, `useMutation` voor writes.
- **Context voor UI state**: Theme, auth, filters. Geen global state libraries.
- **shadcn/ui als component basis**: Niet herimplementeren wat shadcn al biedt.
- **Wouter voor routing**: Lightweight, simpel.
- **Geen prop drilling**: Als props meer dan 2 niveaus diep gaan, gebruik context of compositie.

## Backend (Express)

- **Route pattern**: Router per domein (`routes/items.ts`, `routes/tenants.ts`).
- **Storage layer pattern**: Alle database-operaties via storage interface met `tenantId` parameter.
- **Error handling**: Gebruik `ApiError` met gecentraliseerde error codes.
- **Middleware volgorde**: CORS → helmet → session → CSRF → routes.
- **Geen business logic in routes**: Routes valideren input en roepen services aan.

## API Design

- **RESTful**: `GET /api/items`, `POST /api/items`, `PATCH /api/items/:id`.
- **Consistent error format**: `{ error: string, code?: string }`.
- **Paginatie**: `?page=1&limit=20` met response `{ data: [], total: number }`.
- **Tenant-scoping**: Elke query filtert op tenantId. Geen uitzonderingen behalve superadmin.

## Database

- **Drizzle ORM**: Voor schema definitie en type-safe queries.
- **Migraties als SQL**: Sequentieel genummerd (`0001_initial.sql`, `0002_add_billing.sql`).
- **UUID primary keys**: `uuid("id").primaryKey().defaultRandom()`.
- **snake_case kolommen**: `created_at`, `tenant_id`, `is_active`.
- **RLS op alle tenant-scoped tabellen**: Defense in depth.

## Git

- **Conventionele commits**: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`.
- **Branch per feature**: `feat/add-werkruimte-documents`, `fix/csrf-token-sync`.
- **Geen force push naar main**: Altijd via PR.

## Kwaliteit

- **`tsc --noEmit` voor commits**: Type-check altijd.
- **Vitest voor tests**: Unit tests voor storage, services, utilities.
- **Geen console.log in productie**: Gebruik gestructureerde logging.

---

*Laatst bijgewerkt: Februari 2026*
