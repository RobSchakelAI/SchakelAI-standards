# §1 Dependencies & Project Setup

> Onderdeel van de B2B SaaS Infrastructure Skills — Deel A: Fundament
>
> Zie [skills/README.md](README.md) voor het overzicht van alle secties.

### Wanneer toepassen
Altijd. Dit is stap 0 — installeer deze packages voordat je aan iets anders begint.

### Package.json dependencies

```json
{
  "type": "module",
  "dependencies": {
    // === CORE FRAMEWORK ===
    "express": "^4.21.2",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "typescript": "5.6.3",

    // === DATABASE & ORM ===
    "drizzle-orm": "^0.39.1",
    "postgres": "^3.4.7",

    // === SESSION & AUTH ===
    "express-session": "^1.18.1",
    "connect-pg-simple": "^10.0.0",
    "bcrypt": "^6.0.0",
    "express-rate-limit": "^8.2.1",

    // === VALIDATION ===
    "zod": "^3.24.2",
    "drizzle-zod": "^0.7.0",

    // === DATA FETCHING ===
    "@tanstack/react-query": "^5.60.5",

    // === ROUTING (frontend) ===
    "wouter": "^3.3.5",

    // === STRIPE ===
    "stripe": "^20.3.0",

    // === EMAIL ===
    "mailersend": "^2.6.0",

    // === SECURITY ===
    "helmet": "^8.1.0",

    // === SUPABASE (file storage + DB) ===
    "@supabase/supabase-js": "^2.84.0",

    // === DATES ===
    "date-fns": "^3.6.0",
    "date-fns-tz": "^3.2.0",

    // === UI (shadcn/ui + dependencies) ===
    "tailwindcss": "^3.4.17",
    "lucide-react": "^0.453.0",
    "clsx": "^2.1.1",
    "tailwind-merge": "^2.6.0",
    "class-variance-authority": "^0.7.1",
    "@radix-ui/react-dialog": "latest",
    "@radix-ui/react-dropdown-menu": "latest",
    "@radix-ui/react-accordion": "latest"
  },
  "devDependencies": {
    "drizzle-kit": "^0.31.4",
    "vite": "^5.4.20",
    "@vitejs/plugin-react": "^4.7.0",
    "esbuild": "^0.25.0",
    "tsx": "^4.20.5",
    "vitest": "^4.0.18",
    "tailwindcss-animate": "^1.0.7",
    "autoprefixer": "^10.4.20",
    "postcss": "^8.4.49",

    // === TYPE DEFINITIONS (verplicht voor TypeScript) ===
    "@types/express": "^5.0.2",
    "@types/express-session": "^1.18.1",
    "@types/connect-pg-simple": "^7.0.3",
    "@types/bcrypt": "^5.0.2",
    "@types/node": "^22.15.0"
  }
}
```

### Project structuur
```
/client                          # React frontend (Vite)
  /src
    /components                  # Reusable UI (shadcn/ui)
      /ui                        # Base shadcn components
    /pages                       # Route pages
    /hooks                       # React hooks (useAuth, etc.)
    /lib                         # Utilities (queryClient, timezone)
    /contexts                    # React contexts
    App.tsx                      # Route definitions
    main.tsx                     # Entry point
  index.html

/server                          # Express backend
  /routes                        # API route handlers
  /services                      # Business logic
  /middleware                    # Express middleware
  /utils                         # Shared utilities
  auth.ts                        # Auth setup + endpoints
  storage.ts                     # IStorage interface + MemStorage
  supabase-storage.ts            # Production storage (PostgreSQL)
  app.ts                         # Express app setup + middleware
  index.ts                       # Production entry point

/shared                          # Shared types (frontend + backend)
  schema.ts                      # Zod schemas, Drizzle tables, TS types

/migrations                      # SQL migration files
```

### Vite configuratie
```typescript
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "client/src"),
      "@shared": path.resolve(__dirname, "shared"),
    },
  },
  root: path.resolve(__dirname, "client"),
  build: {
    outDir: path.resolve(__dirname, "dist/public"),
    emptyOutDir: true,
  },
});
```

### Drizzle configuratie
```typescript
// drizzle.config.ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  out: "./migrations",
  schema: "./shared/schema.ts",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

### Drizzle migration workflow

Drizzle wordt **alleen gebruikt voor schema-definities en migraties**. Alle queries gebruiken de `postgres` library direct (zie §4).

```bash
# === Development: direct push (geen migration files) ===
npm run db:push
# Introspects database, vergelijkt met shared/schema.ts, past wijzigingen toe.

# === Production: genereer + apply migration files ===
npx drizzle-kit generate    # Genereert SQL in /migrations
npx drizzle-kit migrate     # Past pending migrations toe op de database

# === Schema wijzigen (workflow): ===
# 1. Pas shared/schema.ts aan (Drizzle pgTable definitie)
# 2. Draai db:push (dev) of generate + migrate (prod)
# 3. Update IStorage interface + implementaties (MemStorage + SupabaseStorage)
```

**RLS apart:** Drizzle beheert geen RLS policies. Die staan in aparte `.sql` bestanden in `/migrations` en worden handmatig uitgevoerd na de initiële schema-migratie (zie §6).

**Transitie van `db:push` naar `generate+migrate`:**
Wanneer je van development naar productie gaat, schakel je over van `db:push` naar migratie-bestanden:
1. Zorg dat je schema in `shared/schema.ts` up-to-date is met de huidige database
2. Draai `npx drizzle-kit generate` — dit genereert de eerste migratie-snapshot
3. Markeer deze migratie als "al toegepast": `npx drizzle-kit migrate --skip` (of verwijder het gegenereerde SQL bestand als de database al up-to-date is)
4. Vanaf nu: elke schema-wijziging → `npx drizzle-kit generate` → review SQL → `npx drizzle-kit migrate`

### Scripts
```json
{
  "scripts": {
    "dev": "NODE_ENV=development tsx server/index-dev.ts",
    "build": "vite build && esbuild server/index.ts --bundle --platform=node --outdir=dist",
    "start": "NODE_ENV=production node dist/index.js",
    "check": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "db:push": "drizzle-kit push"
  }
}
```

### TypeScript configuratie (tsconfig.json)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": ".",
    "paths": {
      "@/*": ["client/src/*"],
      "@shared/*": ["shared/*"]
    }
  },
  "include": [
    "client/src/**/*",
    "server/**/*",
    "shared/**/*"
  ],
  "exclude": ["node_modules", "dist"]
}
```

**KRITIEK:** De `paths` moeten exact overeenkomen met de `resolve.alias` in `vite.config.ts`. Zonder dit geeft TypeScript errors op elke `@/` import.

### Tailwind configuratie (tailwind.config.js)

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  darkMode: "class",
  content: ["./client/index.html", "./client/src/**/*.{ts,tsx}"],
  theme: {
    extend: {
      // shadcn/ui CSS variabelen
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: { DEFAULT: "hsl(var(--primary))", foreground: "hsl(var(--primary-foreground))" },
        secondary: { DEFAULT: "hsl(var(--secondary))", foreground: "hsl(var(--secondary-foreground))" },
        destructive: { DEFAULT: "hsl(var(--destructive))", foreground: "hsl(var(--destructive-foreground))" },
        muted: { DEFAULT: "hsl(var(--muted))", foreground: "hsl(var(--muted-foreground))" },
        accent: { DEFAULT: "hsl(var(--accent))", foreground: "hsl(var(--accent-foreground))" },
        card: { DEFAULT: "hsl(var(--card))", foreground: "hsl(var(--card-foreground))" },
      },
      borderRadius: { lg: "var(--radius)", md: "calc(var(--radius) - 2px)", sm: "calc(var(--radius) - 4px)" },
    },
  },
  plugins: [require("tailwindcss-animate")],
};
```

### PostCSS configuratie (postcss.config.js)

```javascript
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

### shadcn/ui setup

shadcn/ui is **geen npm package** maar een CLI die componenten als broncode in je project plaatst. Setup:

```bash
# Initialiseer shadcn/ui (eenmalig)
npx shadcn-ui@latest init
# Kies: TypeScript, Default style, CSS variables, client/src/components/ui

# Componenten toevoegen
npx shadcn-ui@latest add button card dialog dropdown-menu input label toast
```

De `cn()` utility (class merging) wordt automatisch aangemaakt in `client/src/lib/utils.ts`:

```typescript
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

**CSS variabelen** staan in `client/src/index.css` (aangemaakt door shadcn-init). Dit bestand importeer je in `client/src/main.tsx`.

**Extra dependencies:** `clsx`, `tailwind-merge`, `tailwindcss-animate`, `class-variance-authority`, `autoprefixer` — worden automatisch geïnstalleerd door shadcn-init. Als je shadcn-init niet gebruikt, installeer ze handmatig:

```bash
npm install clsx tailwind-merge class-variance-authority
npm install -D tailwindcss-animate autoprefixer
```

### Frontend entry bestanden (VERPLICHT)

Deze drie bestanden zijn nodig voordat Vite je frontend kan bouwen. Ze worden niet automatisch aangemaakt.

#### client/index.html (Vite entry point)

```html
<!DOCTYPE html>
<html lang="nl">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>App</title>
    <meta name="description" content="B2B SaaS Platform" />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

**KRITIEK:** Het `src` attribuut verwijst naar `/src/main.tsx` (relatief aan de `client/` directory, niet de project root). Vite gebruikt `client/` als `root` (zie vite.config.ts).

#### client/src/main.tsx (React entry point)

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { QueryClientProvider } from "@tanstack/react-query";
import { queryClient } from "@/lib/queryClient";
import { AuthProvider } from "@/hooks/use-auth";
import App from "./App";
import "./index.css";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <AuthProvider>
        <App />
      </AuthProvider>
    </QueryClientProvider>
  </StrictMode>
);
```

**Provider volgorde:** `QueryClientProvider` → `AuthProvider` → `App`. AuthProvider gebruikt useQuery intern, dus moet binnen QueryClientProvider staan.

#### client/src/index.css (Tailwind + shadcn CSS variabelen)

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;
    --primary: 222.2 47.4% 11.2%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96.1%;
    --secondary-foreground: 222.2 47.4% 11.2%;
    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96.1%;
    --accent-foreground: 222.2 47.4% 11.2%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 222.2 84% 4.9%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;
    --primary: 210 40% 98%;
    --primary-foreground: 222.2 47.4% 11.2%;
    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;
    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 212.7 26.8% 83.9%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

**Dit bestand wordt aangemaakt door `npx shadcn-ui@latest init`** maar als je handmatig setup doet, moet je het zelf aanmaken. De CSS variabelen matchen met de Tailwind config in `tailwind.config.js`.

---

