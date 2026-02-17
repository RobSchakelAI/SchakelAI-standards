# Deployment Guide: Vercel + Railway + Supabase

This guide explains how to deploy the Schakel AI Meeting Automation app to production using Vercel (frontend), Railway (backend), and Supabase (database).

## Architecture Overview

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│     Vercel      │     │    Railway      │     │    Supabase     │
│   (Frontend)    │ ──► │   (Backend)     │ ──► │   (Database)    │
│  React + Vite   │     │   Express API   │     │   PostgreSQL    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

## Required npm Scripts

Add these scripts to your `package.json`:

```json
{
  "scripts": {
    "build:frontend": "vite build",
    "build:backend": "esbuild server/index-railway.ts --platform=node --packages=external --bundle --format=esm --outfile=dist/index.js",
    "start:railway": "NODE_ENV=production node dist/index.js",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "tsx server/migrate.ts",
    "db:push": "drizzle-kit push"
  }
}
```

## Step 1: Supabase Setup

### Create Two Supabase Projects
1. **Staging**: `schakel-staging`
2. **Production**: `schakel-production`

### For Each Project:
1. Go to Project Settings → API
2. Copy:
   - `Project URL` → `SUPABASE_URL`
   - `anon public` key → `SUPABASE_ANON_KEY`
   - `service_role` key → `SUPABASE_SERVICE_KEY`

### Create Storage Bucket
1. Go to Storage → New Bucket
2. Name: `branding`
3. Public: Yes
4. File size limit: 5MB
5. Allowed MIME types: `image/png, image/jpeg, image/gif, image/webp`

### Apply Database Schema
```bash
# For staging
DATABASE_URL=<staging-connection-string> npm run db:push

# For production
DATABASE_URL=<production-connection-string> npm run db:push
```

## Step 2: GitHub Setup

### Create Repository
```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/your-org/schakel-ai.git
git push -u origin main
```

### Create Branches
```bash
git checkout -b dev
git push -u origin dev
```

### Branch Strategy
- `dev` → Staging deployments
- `main` → Production deployments

## Step 3: Railway Setup

### Create Project
1. Go to [Railway](https://railway.app)
2. New Project → Deploy from GitHub repo
3. Select your repository

### Configure Environments
Create two environments: `staging` (from `dev` branch) and `production` (from `main` branch).

### Environment Variables (for each environment)

```
# Database
DATABASE_URL=<supabase-connection-string>
SUPABASE_URL=<supabase-project-url>
SUPABASE_ANON_KEY=<supabase-anon-key>
SUPABASE_SERVICE_KEY=<supabase-service-role-key>

# Security
ENCRYPTION_KEY=<generate-with-openssl-rand-base64-32>
SESSION_SECRET=<random-string>

# CORS (Vercel frontend URL)
CORS_ORIGIN=https://your-app.vercel.app

# External APIs (or leave empty for frontend configuration)
ANTHROPIC_API_KEY=<optional>
FIREFLIES_API_KEY=<optional>
MICROSOFT_CLIENT_ID=<optional>
MICROSOFT_CLIENT_SECRET=<optional>
MICROSOFT_TENANT_ID=<optional>
PRODUCTIVE_API_KEY=<optional>
PRODUCTIVE_ORG_ID=<optional>
TEAMS_WEBHOOK_URL=<optional>

# Runtime
NODE_ENV=production
PORT=5000
```

### Settings
- Build Command: `npm run build:backend`
- Start Command: `npm run start:railway`
- Health Check Path: `/api/health`

## Step 4: Vercel Setup

### Create Project
1. Go to [Vercel](https://vercel.com)
2. Add New → Project → Import from GitHub
3. Select your repository

### Configure Environments
- Preview: Deploy from `dev` branch
- Production: Deploy from `main` branch

### Environment Variables

```
# API URL (Railway backend)
VITE_API_URL=https://your-app.railway.app
```

### Settings
- Framework: Vite
- Build Command: `npm run build:frontend`
- Output Directory: `dist/public`
- Install Command: `npm install`

## Step 5: Microsoft Azure AD Configuration

### Update Redirect URIs
In Azure Portal → App Registrations → Your App → Authentication:

Add these redirect URIs:
```
# Staging
https://your-staging-app.railway.app/api/auth/microsoft/callback

# Production
https://your-production-app.railway.app/api/auth/microsoft/callback
```

## Step 6: Recall.ai Configuration

### Create Recall.ai Account
1. Go to [Recall.ai Dashboard](https://recall.ai)
2. Create an account and get API key

### Environment Variables (Railway)
```
RECALL_API_KEY=<from Recall.ai dashboard>
RECALL_REGION=eu-central-1
RECALL_WEBHOOK_SECRET=<create a secure random string>
```

### Regional Endpoints
Choose the region closest to your users:
- `us-east-1` - US East (default)
- `us-west-2` - US West
- `eu-central-1` - EU Frankfurt (recommended for EU)
- `ap-northeast-1` - Asia Tokyo

### Configure Webhook in Recall.ai Dashboard
1. Go to Settings → Webhooks
2. Add webhook URL:
   ```
   # Staging
   https://map-api-dev.schakel.ai/api/webhook/recall

   # Production
   https://map-api.schakel.ai/api/webhook/recall
   ```
3. Select events:
   - `bot.status_changed`
   - `transcript.done`
   - `transcript.failed`
4. Set webhook secret (same as `RECALL_WEBHOOK_SECRET`)

### Verify Configuration
```bash
curl https://your-app.railway.app/api/webhook/recall/test
# Should return: {"status":"ok","recallConfigured":true,...}
```

---

## Step 7: Stripe Billing Configuration

### Create Stripe Account
1. Go to [Stripe Dashboard](https://dashboard.stripe.com)
2. Create products and prices for each tier

### Environment Variables (Railway)
```
STRIPE_SECRET_KEY=sk_live_... (or sk_test_... for staging)
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRICE_STARTER_MONTHLY=price_...
STRIPE_PRICE_STARTER_ANNUAL=price_...
STRIPE_PRICE_PRO_MONTHLY=price_...
STRIPE_PRICE_PRO_ANNUAL=price_...
```

### Configure Webhook in Stripe Dashboard
1. Go to Developers → Webhooks
2. Add endpoint:
   ```
   # Staging
   https://map-api-dev.schakel.ai/api/webhook/stripe

   # Production
   https://map-api.schakel.ai/api/webhook/stripe
   ```
3. Select events:
   - `checkout.session.completed`
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
   - `invoice.payment_succeeded`
   - `invoice.payment_failed`

### Configure Customer Portal
1. Go to Settings → Billing → Customer portal
2. Enable subscription management
3. Configure allowed actions (cancel, update payment method)

---

## Step 8: Fireflies.ai Webhook Configuration (Legacy)

> **Note:** Fireflies.ai is being phased out in favor of Recall.ai

In Fireflies.ai dashboard → Integrations → Webhooks:

```
# Staging
https://your-staging-app.railway.app/api/webhooks/fireflies

# Production
https://your-production-app.railway.app/api/webhooks/fireflies
```

## Deployment Workflow

### Development → Staging
```bash
# Make changes in Replit
git add .
git commit -m "Your changes"
git push origin dev
# → Triggers Vercel preview + Railway staging deployment
```

### Staging → Production
```bash
# After testing staging
git checkout main
git merge dev
git push origin main
# → Triggers Vercel production + Railway production deployment
```

## Database Migrations

### Option 1: Push (Simpler, for small changes)
```bash
DATABASE_URL=<connection-string> npm run db:push
```

### Option 2: Migrations (For production)
```bash
# Generate migration
npm run db:generate

# Apply migration
DATABASE_URL=<connection-string> npm run db:migrate
```

## Monitoring

### Health Check
```bash
curl https://your-app.railway.app/api/health
```

### Logs
- Railway: Dashboard → Deployments → Logs
- Vercel: Dashboard → Deployments → Functions

## Troubleshooting

### CORS Errors
Ensure `CORS_ORIGIN` in Railway matches your Vercel URL exactly.

### Database Connection Issues
Check that `DATABASE_URL` uses the correct format:
```
postgresql://user:password@host:port/database?sslmode=require
```

### Supabase Storage 403 Errors
Ensure the `branding` bucket is set to public.

## Step 9: Email Configuration (MailerSend)

### Create MailerSend Account
1. Go to [MailerSend](https://mailersend.com)
2. Verify your domain (schakel.ai)
3. Create API key

### Environment Variables (Railway)
```
MAILERSEND_API_KEY=<from MailerSend dashboard>
MAILERSEND_FROM_EMAIL=noreply@schakel.ai
```

### Email Types Sent
- User invitations
- Password reset
- MFA verification codes
- Account deletion confirmation
- Welcome emails after subscription

---

## Complete Environment Variables Checklist

### Core (Required)
| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | Supabase connection string | `postgresql://...` |
| `SESSION_SECRET` | Random string for sessions | `openssl rand -base64 32` |
| `ENCRYPTION_KEY` | 32-byte hex for secrets | `openssl rand -hex 32` |
| `FRONTEND_URL` | Vercel frontend URL | `https://map.schakel.ai` |
| `CORS_ORIGIN` | Same as FRONTEND_URL | `https://map.schakel.ai` |

### AI (Required)
| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Claude API key |

### Recall.ai (Required for recording)
| Variable | Description |
|----------|-------------|
| `RECALL_API_KEY` | Recall.ai API key |
| `RECALL_REGION` | `eu-central-1` for EU |
| `RECALL_WEBHOOK_SECRET` | Webhook signature secret |

### Stripe (Required for billing)
| Variable | Description |
|----------|-------------|
| `STRIPE_SECRET_KEY` | `sk_live_...` or `sk_test_...` |
| `STRIPE_WEBHOOK_SECRET` | `whsec_...` |
| `STRIPE_PRICE_STARTER_MONTHLY` | `price_...` |
| `STRIPE_PRICE_STARTER_ANNUAL` | `price_...` |
| `STRIPE_PRICE_PRO_MONTHLY` | `price_...` |
| `STRIPE_PRICE_PRO_ANNUAL` | `price_...` |

### Email (Required for notifications)
| Variable | Description |
|----------|-------------|
| `MAILERSEND_API_KEY` | MailerSend API key |
| `MAILERSEND_FROM_EMAIL` | `noreply@schakel.ai` |

### Microsoft 365 (Optional, for SharePoint/Calendar)
| Variable | Description |
|----------|-------------|
| `AZURE_CLIENT_ID` | Azure AD app client ID |
| `AZURE_CLIENT_SECRET` | Azure AD app client secret |
| `AZURE_TENANT_ID` | Azure AD tenant ID |

### Runtime
| Variable | Value |
|----------|-------|
| `NODE_ENV` | `production` |
| `PORT` | `5000` |

---

## Security Checklist

- [ ] ENCRYPTION_KEY is unique per environment
- [ ] SESSION_SECRET is unique per environment
- [ ] SUPABASE_SERVICE_KEY is only in Railway (never in Vercel)
- [ ] All API keys are properly secured
- [ ] CORS_ORIGIN is set to exact frontend URL (no wildcards in production)
- [ ] RECALL_WEBHOOK_SECRET configured for webhook verification
- [ ] STRIPE_WEBHOOK_SECRET configured for webhook verification
- [ ] All webhook endpoints registered in respective dashboards

---

## Post-Deployment Verification

### Health Checks
```bash
# Backend health
curl https://your-app.railway.app/api/health

# Recall.ai integration
curl https://your-app.railway.app/api/webhook/recall/test

# Stripe integration (requires auth)
# Check billing status in app after login
```

### Webhook Verification
1. **Recall.ai**: Check Recall dashboard → Webhooks → Delivery logs
2. **Stripe**: Check Stripe dashboard → Developers → Webhooks → Logs

### Common Issues

#### CORS Errors
- Ensure `CORS_ORIGIN` exactly matches Vercel URL (including `https://`)
- No trailing slash

#### Recall Bot Not Joining
- Check `RECALL_REGION` matches your API key region
- Verify webhook URL is correct in Recall dashboard
- Check Railway logs for `[Recall Webhook]` entries

#### Stripe Checkout Failing
- Verify all 4 price IDs are set
- Check Stripe dashboard for webhook delivery
- Ensure `FRONTEND_URL` is set for redirect

#### Pipeline Not Running
- Check Railway logs for `[Pipeline]` entries
- Verify `ANTHROPIC_API_KEY` is set
- Check meeting status in database
