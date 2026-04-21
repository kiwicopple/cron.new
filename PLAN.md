# cron.new — Implementation Plan

A SaaS cron job manager for Supabase, distributed as an OAuth app in the Supabase marketplace. Users pay per active cron job. Jobs can trigger HTTP endpoints, SQL queries, or Edge Functions against the user's Supabase project.

---

## Architecture Overview

```
User's Browser
     │
     ▼
Next.js App (cron.new)
     │
     ├── Supabase (cron.new's own DB — users, jobs, run logs)
     │
     ├── QStash (Upstash) ──► POST /api/execute/[jobId] ──► User's Supabase Project
     │
     └── Stripe (billing — pay per active job)
```

**Key design choice:** QStash fires a webhook to our app on schedule. Our app then executes the action against the user's Supabase project using their stored OAuth token. This keeps us in the critical path for logging, billing, and error handling.

---

## Tech Stack

| Concern | Tool |
|---|---|
| Framework | Next.js 15 (App Router) |
| App database | Supabase (Postgres + RLS) |
| Auth | Supabase OAuth (the integration itself doubles as login) |
| Cron execution | QStash (Upstash) |
| Billing | Stripe (usage-based, per active job) |
| Deployment | Vercel |
| Token encryption | AES-256-GCM (Node crypto) |

---

## Database Schema

All tables live in cron.new's own Supabase project.

### `organizations`
Represents a connected Supabase project (one OAuth grant = one organization).

```sql
create table organizations (
  id               uuid primary key default gen_random_uuid(),
  supabase_user_id text unique not null,         -- from OAuth token claims
  project_ref      text not null,                -- e.g. "abcdefghijklmnop"
  project_name     text,
  access_token     text not null,                -- AES-256 encrypted
  refresh_token    text not null,                -- AES-256 encrypted
  token_expires_at timestamptz not null,
  stripe_customer_id text,
  plan             text not null default 'free', -- 'free' | 'pro'
  created_at       timestamptz default now()
);
```

### `jobs`
One row per cron job.

```sql
create table jobs (
  id                  uuid primary key default gen_random_uuid(),
  organization_id     uuid references organizations(id) on delete cascade,
  name                text not null,
  cron_expression     text not null,             -- standard 5-field cron
  job_type            text not null,             -- 'http' | 'sql' | 'edge_function'

  -- HTTP jobs
  http_url            text,
  http_method         text default 'POST',
  http_headers        jsonb default '{}',
  http_body           text,

  -- SQL jobs
  sql_query           text,

  -- Edge Function jobs
  edge_function_name  text,
  edge_function_body  jsonb default '{}',

  qstash_schedule_id  text,                      -- QStash schedule ID for deletion
  is_active           boolean default true,
  last_run_at         timestamptz,
  last_run_status     text,                      -- 'success' | 'failed'
  created_at          timestamptz default now(),
  updated_at          timestamptz default now()
);
```

### `job_runs`
Immutable log of every execution attempt.

```sql
create table job_runs (
  id              uuid primary key default gen_random_uuid(),
  job_id          uuid references jobs(id) on delete cascade,
  status          text not null,       -- 'running' | 'success' | 'failed'
  started_at      timestamptz default now(),
  completed_at    timestamptz,
  duration_ms     int,
  response_status int,                 -- HTTP status code from target
  response_body   text,               -- truncated to 10KB
  error_message   text,
  created_at      timestamptz default now()
);
```

Enable RLS on all tables. Policy: `organization_id` must match the org derived from the session token.

---

## Supabase OAuth App Registration

1. Apply at https://supabase.com/dashboard/org/_/integrations to register an OAuth app.
2. Required scopes: `database.read`, `database.write`, `edge_functions.write` (only request what's needed).
3. Set redirect URI to: `https://cron.new/api/auth/callback`
4. Store `SUPABASE_CLIENT_ID` and `SUPABASE_CLIENT_SECRET` in environment.

### OAuth Flow

```
1. User visits cron.new and clicks "Connect Supabase Project"
2. We redirect to:
   https://api.supabase.com/v1/oauth/authorize
     ?client_id=CLIENT_ID
     &redirect_uri=https://cron.new/api/auth/callback
     &response_type=code
     &scope=database.read database.write edge_functions.write
     &state=<random CSRF token stored in cookie>

3. User approves in Supabase dashboard

4. Supabase redirects to /api/auth/callback?code=AUTH_CODE&state=...

5. We exchange code for tokens:
   POST https://api.supabase.com/v1/oauth/token
     { grant_type: "authorization_code", code, redirect_uri, ... }

6. Response: { access_token, refresh_token, expires_in, project_ref }

7. Encrypt tokens, upsert into organizations table, set session cookie
```

### Token Refresh

- Check `token_expires_at` before every API call to the user's project.
- If expired (or within 5 minutes), call `/v1/oauth/token` with `grant_type=refresh_token`.
- Update encrypted tokens in DB.
- Run a background cron (ironic) every hour to proactively refresh expiring tokens.

---

## API Routes

### Auth
| Route | Method | Description |
|---|---|---|
| `/api/auth/connect` | GET | Redirect to Supabase OAuth |
| `/api/auth/callback` | GET | Exchange code, store tokens, set session |
| `/api/auth/logout` | POST | Clear session cookie |

### Jobs
| Route | Method | Description |
|---|---|---|
| `/api/jobs` | GET | List all jobs for authenticated org |
| `/api/jobs` | POST | Create job + register QStash schedule |
| `/api/jobs/[id]` | GET | Get job + recent runs |
| `/api/jobs/[id]` | PATCH | Update job (reschedule QStash if cron changed) |
| `/api/jobs/[id]` | DELETE | Delete job + remove QStash schedule |
| `/api/jobs/[id]/runs` | GET | Paginated run history |
| `/api/jobs/[id]/trigger` | POST | Manual trigger (bypass QStash) |

### Execution (called by QStash)
| Route | Method | Description |
|---|---|---|
| `/api/execute/[jobId]` | POST | QStash receiver — run the job, log result |

This endpoint must verify the `Upstash-Signature` header using the QStash signing key before executing.

### Billing
| Route | Method | Description |
|---|---|---|
| `/api/billing/portal` | POST | Create Stripe billing portal session |
| `/api/billing/webhook` | POST | Handle Stripe webhook events |

---

## QStash Integration

### Creating a schedule (on job creation)
```typescript
const client = new QStashClient({ token: process.env.QSTASH_TOKEN })

const schedule = await client.schedules.create({
  destination: `https://cron.new/api/execute/${jobId}`,
  cron: job.cron_expression,
  body: JSON.stringify({ jobId }),
  headers: { "Content-Type": "application/json" },
  retries: 3,
})

// Store schedule.scheduleId in jobs.qstash_schedule_id
```

### Deleting a schedule (on job deletion/pause)
```typescript
await client.schedules.delete(job.qstash_schedule_id)
```

### Execution endpoint
```typescript
// /api/execute/[jobId]/route.ts
import { verifySignatureAppRouter } from "@upstash/qstash/nextjs"

export const POST = verifySignatureAppRouter(async (req) => {
  const { jobId } = await req.json()
  const job = await getJob(jobId)
  const org = await getOrganization(job.organization_id)
  const token = await decryptAndRefreshToken(org)

  const runId = await createJobRun(jobId, "running")
  const start = Date.now()

  try {
    const result = await executeJob(job, token)
    await updateJobRun(runId, "success", Date.now() - start, result)
    await reportStripeUsage(org.stripe_customer_id)
  } catch (err) {
    await updateJobRun(runId, "failed", Date.now() - start, err)
  }
})
```

### Job execution by type
- **HTTP**: `fetch(job.http_url, { method, headers, body })`
- **SQL**: Call Supabase Management API `POST /v1/projects/{ref}/database/query` with the SQL string
- **Edge Function**: `fetch(\`https://${project_ref}.supabase.co/functions/v1/${name}\`, { body })`

---

## Billing (Stripe)

### Model
- **Free tier**: up to 2 active jobs, minimum 1-hour interval
- **Pro tier**: $1 per active job per month (usage-based)

### Setup
1. Create a Stripe product + meter for "active_jobs"
2. On org creation, create a Stripe customer and store `stripe_customer_id`
3. When a job is created: `stripe.billing.meterEvents.create({ event_name: 'active_jobs', value: 1 })`
4. When a job is deleted/paused: report -1 meter event
5. Stripe handles monthly invoicing

### Webhook events to handle
- `invoice.payment_failed` → suspend all jobs for org, email user
- `customer.subscription.deleted` → downgrade to free, pause jobs over limit

---

## Pages & UI

### `/` — Landing page
Keep the existing cron expression generator. Add:
- Hero section explaining the product
- "Connect Supabase Project" CTA button
- Pricing section (Free / Pro)

### `/dashboard` — Job list
- Table of all jobs: name, cron expression, type, last run status, last run time
- "New Job" button
- Active/paused toggle per job

### `/dashboard/jobs/new` — Create job
- Job name input
- Cron expression builder (reuse the existing widget from index.html)
- Job type selector: HTTP / SQL / Edge Function
- Dynamic form fields based on type
- "Test run" button before saving

### `/dashboard/jobs/[id]` — Job detail
- Job config (editable inline)
- Run history table: timestamp, status, duration, response code
- Expandable row to see response body / error
- Manual trigger button

### `/dashboard/billing` — Billing
- Current plan + active job count
- Stripe billing portal link

---

## Project Structure

```
cron.new/
├── app/
│   ├── page.tsx                        # Landing page (replaces index.html)
│   ├── dashboard/
│   │   ├── layout.tsx
│   │   ├── page.tsx                    # Job list
│   │   ├── jobs/
│   │   │   ├── new/page.tsx
│   │   │   └── [id]/page.tsx
│   │   └── billing/page.tsx
│   └── api/
│       ├── auth/
│       │   ├── connect/route.ts
│       │   └── callback/route.ts
│       ├── jobs/
│       │   ├── route.ts
│       │   └── [id]/
│       │       ├── route.ts
│       │       ├── runs/route.ts
│       │       └── trigger/route.ts
│       ├── execute/
│       │   └── [jobId]/route.ts        # QStash receiver
│       └── billing/
│           ├── portal/route.ts
│           └── webhook/route.ts
├── lib/
│   ├── supabase.ts                     # Supabase client (app's own DB)
│   ├── supabase-oauth.ts               # Token exchange + refresh logic
│   ├── qstash.ts                       # QStash client + helpers
│   ├── stripe.ts                       # Stripe client + helpers
│   ├── crypto.ts                       # AES-256 encrypt/decrypt for tokens
│   └── execute-job.ts                  # Job execution by type
├── components/
│   ├── CronBuilder.tsx                 # Reuse existing cron expression UI
│   ├── JobForm.tsx
│   ├── JobRunsTable.tsx
│   └── JobStatusBadge.tsx
├── supabase/
│   └── migrations/
│       └── 0001_initial.sql
├── index.html                          # Keep until migration to Next.js is done
├── CNAME
└── PLAN.md
```

---

## Environment Variables

```bash
# Supabase (cron.new's own project)
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Supabase OAuth App credentials
SUPABASE_OAUTH_CLIENT_ID=
SUPABASE_OAUTH_CLIENT_SECRET=

# QStash
QSTASH_TOKEN=
QSTASH_CURRENT_SIGNING_KEY=
QSTASH_NEXT_SIGNING_KEY=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_METER_ID=

# Token encryption (32-byte hex secret)
TOKEN_ENCRYPTION_KEY=

# App
NEXT_PUBLIC_APP_URL=https://cron.new
```

---

## Implementation Phases

### Phase 1 — Foundation
- [ ] Initialize Next.js 15 project with TypeScript, Tailwind, ESLint
- [ ] Create Supabase project for cron.new's own data
- [ ] Write and apply DB migrations
- [ ] Set up `lib/supabase.ts` (server + client helpers)
- [ ] Set up `lib/crypto.ts` for token encryption
- [ ] Session cookie management (encrypted httpOnly cookie with org ID)

### Phase 2 — Supabase OAuth
- [ ] Register OAuth app with Supabase
- [ ] Implement `/api/auth/connect` redirect
- [ ] Implement `/api/auth/callback` token exchange
- [ ] Implement `lib/supabase-oauth.ts` token refresh logic
- [ ] Protect dashboard routes (redirect to `/` if no session)

### Phase 3 — Job Management + QStash
- [ ] Implement CRUD API routes for jobs
- [ ] Implement `lib/qstash.ts` schedule helpers
- [ ] Wire up QStash schedule creation/deletion on job create/delete
- [ ] Implement `/api/execute/[jobId]` with signature verification
- [ ] Implement `lib/execute-job.ts` for HTTP, SQL, and Edge Function execution
- [ ] Job run logging

### Phase 4 — Dashboard UI
- [ ] Build dashboard layout with sidebar nav
- [ ] Job list page with status indicators
- [ ] Create/edit job form (reuse CronBuilder from index.html)
- [ ] Job detail page with run history
- [ ] Manual trigger button

### Phase 5 — Billing
- [ ] Set up Stripe product + meter
- [ ] Create Stripe customer on first OAuth connection
- [ ] Report meter events on job create/delete
- [ ] Implement `/api/billing/webhook` for payment failure handling
- [ ] Free tier enforcement (2-job limit)
- [ ] Billing portal page

### Phase 6 — Production Hardening
- [ ] Update landing page with product description and pricing
- [ ] Rate limiting on execute endpoint
- [ ] Background token refresh job (via QStash cron on our own app)
- [ ] Error monitoring (Sentry)
- [ ] Migrate DNS from GitHub Pages to Vercel
- [ ] Load test QStash execute endpoint

---

## Security Considerations

- **OAuth tokens** are AES-256-GCM encrypted at rest; decrypted only in-memory during execution.
- **QStash signature** verified on every incoming webhook before executing any job.
- **CSRF state parameter** validated on OAuth callback.
- **SQL jobs** execute via Supabase's own Management API under the user's own credentials — we never have unrestricted DB access beyond what OAuth scopes allow.
- **Session cookies** are httpOnly, Secure, SameSite=Lax with short expiry (7 days); refreshed on activity.
- **RLS** on all cron.new DB tables ensures organizations can only see their own data.
