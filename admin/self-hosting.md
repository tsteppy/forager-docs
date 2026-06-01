---
title: "Self-Hosting Forager"
description: "Complete guide for enterprise teams deploying Forager on their own infrastructure"
---

<Warning>
  Self-hosting requires comfortable familiarity with Linux server administration, Docker, and SQL. Plan for a half-day of setup time. Contact [support@harbinge.rs](mailto:support@harbinge.rs) before starting — we provide a migration bundle and a custom-compiled APK for self-hosted deployments.
</Warning>

## Overview

A self-hosted Forager deployment consists of four components:

| Component | What it is | Hosted by you |
|---|---|---|
| **Supabase** | Postgres database, Auth, Edge Functions, Storage | ✓ |
| **Web Dashboard** | Next.js admin and reporting UI | ✓ |
| **Edge Functions** | Deno serverless functions (classification, webhooks, invites) | ✓ (via Supabase) |
| **Android APK** | Field tech mobile app | Custom build provided by Hiverise |

The Android app has your Supabase URL and anon key compiled in. Hiverise builds this APK for you once your Supabase instance is running — you cannot reuse the standard APK from the downloads page.

---

## Prerequisites

Before starting, ensure the following are available on your server:

| Requirement | Version | Notes |
|---|---|---|
| Linux server | Ubuntu 22.04 LTS recommended | 4 GB RAM minimum, 8 GB recommended |
| Docker | 24+ | |
| Docker Compose | v2 (`docker compose`) | Not `docker-compose` v1 |
| Node.js | 18+ | For building/running the web dashboard |
| Supabase CLI | Latest | `npm install -g supabase` |
| A domain name | — | For the web dashboard (e.g. `forager.yourcompany.com`) |
| SSL certificate | — | Let's Encrypt via Certbot or your existing reverse proxy |
| SMTP credentials | — | For Auth invite emails (SendGrid, Postmark, or your mail server) |

---

## Step 1 — Deploy Supabase

Self-hosted Supabase runs as a Docker Compose stack.

<Steps>
  <Step title="Clone the Supabase Docker setup">
    ```bash
    git clone --depth 1 https://github.com/supabase/supabase
    cd supabase/docker
    cp .env.example .env
    ```
  </Step>
  <Step title="Configure the .env file">
    Open `.env` and set the following. Generate secure random values for secrets — do not reuse the example values.

    ```bash
    # Generate secrets (run each separately and paste the output)
    openssl rand -base64 32   # → POSTGRES_PASSWORD
    openssl rand -base64 32   # → JWT_SECRET
    openssl rand -base64 32   # → ANON_KEY  ← use a proper JWT generator (see below)
    openssl rand -base64 32   # → SERVICE_ROLE_KEY  ← same
    ```

    <Note>
      `ANON_KEY` and `SERVICE_ROLE_KEY` must be valid JWTs signed with your `JWT_SECRET`, not raw random strings. Use the Supabase self-hosting JWT generator at [supabase.com/docs/guides/self-hosting#api-keys](https://supabase.com/docs/guides/self-hosting#api-keys) to generate them correctly.
    </Note>

    Also configure your SMTP credentials in `.env`:

    ```
    SMTP_ADMIN_EMAIL=noreply@yourcompany.com
    SMTP_HOST=smtp.yourprovider.com
    SMTP_PORT=587
    SMTP_USER=your-smtp-user
    SMTP_PASS=your-smtp-password
    SMTP_SENDER_NAME=Forager
    ```

    Set the site URL (your dashboard domain):

    ```
    SITE_URL=https://forager.yourcompany.com
    ADDITIONAL_REDIRECT_URLS=https://forager.yourcompany.com/**
    ```
  </Step>
  <Step title="Start Supabase">
    ```bash
    docker compose up -d
    ```

    Supabase exposes several ports. The ones you need:

    | Port | Service |
    |---|---|
    | `8000` | API gateway (Kong) — this is your `SUPABASE_URL` origin |
    | `3000` | Supabase Studio (admin UI) |
    | `5432` | Postgres (internal only — do not expose publicly) |

    Access Studio at `http://your-server-ip:3000` to verify Supabase is running before continuing.
  </Step>
  <Step title="Set your public Supabase URL">
    Your `SUPABASE_URL` for all subsequent steps is `https://your-server-ip:8000` or, if you put a reverse proxy in front, `https://supabase.yourcompany.com`. Make a note of it — you'll need it in every step below.
  </Step>
</Steps>

---

## Step 2 — Run Database Migrations

Forager's schema is managed as ordered SQL migrations. Apply them using the Supabase CLI against your self-hosted instance.

<Steps>
  <Step title="Get the Forager migration bundle">
    Contact Hiverise to receive the migration bundle. It contains all SQL files in `supabase/migrations/`, numbered chronologically. Do not skip or reorder them.
  </Step>
  <Step title="Link the CLI to your instance">
    ```bash
    supabase login   # not needed for self-hosted; skip this
    supabase db push --db-url "postgresql://postgres:YOUR_POSTGRES_PASSWORD@your-server-ip:5432/postgres"
    ```

    Or apply migrations manually in order using psql:

    ```bash
    PGPASSWORD=YOUR_POSTGRES_PASSWORD psql \
      -h your-server-ip -U postgres -d postgres \
      -f 20260429000000_location_hierarchy.sql
    # Repeat for each file in chronological order
    ```

    <Note>
      Apply all 29 migrations in filename order. Each migration is idempotent-safe when applied once, but do not apply any file more than once.
    </Note>
  </Step>
  <Step title="Verify the schema">
    In Supabase Studio → Table Editor, confirm these tables exist: `companies`, `profiles`, `location_nodes`, `assets`, `anchor_snapshots`, `attestations`, `webhooks`, `sprints`, `site_memberships`.
  </Step>
</Steps>

---

## Step 3 — Deploy Edge Functions

Forager uses five Edge Functions (Deno). Deploy them using the Supabase CLI.

<Steps>
  <Step title="Deploy all functions">
    From the root of the Forager source bundle:

    ```bash
    supabase functions deploy classify-room \
      --project-ref YOUR_PROJECT_REF \
      --no-verify-jwt

    supabase functions deploy rebuild-fingerprint
    supabase functions deploy invite-user
    supabase functions deploy import-assets
    supabase functions deploy push-webhooks
    ```

    For self-hosted, replace `--project-ref` with your self-hosted API URL:

    ```bash
    supabase functions deploy classify-room \
      --url http://your-server-ip:8000 \
      --anon-key YOUR_ANON_KEY
    ```
  </Step>
  <Step title="Set function secrets">
    Each function reads secrets from the Supabase Edge Function secrets store. Set them all at once:

    ```bash
    supabase secrets set \
      SUPABASE_URL=https://your-supabase-url \
      SUPABASE_ANON_KEY=your-anon-key \
      SUPABASE_SERVICE_ROLE_KEY=your-service-role-key \
      DASHBOARD_URL=https://forager.yourcompany.com \
      WEBHOOK_INTERNAL_SECRET=$(openssl rand -hex 32)
    ```

    | Secret | Used by | Purpose |
    |---|---|---|
    | `SUPABASE_URL` | All functions | Supabase API endpoint |
    | `SUPABASE_ANON_KEY` | `classify-room`, `import-assets` | Row-level auth for user-scoped queries |
    | `SUPABASE_SERVICE_ROLE_KEY` | `invite-user`, `import-assets`, `rebuild-fingerprint`, `push-webhooks` | Admin operations that bypass RLS |
    | `DASHBOARD_URL` | `invite-user` | Invite email link destination |
    | `WEBHOOK_INTERNAL_SECRET` | `push-webhooks` | Validates internal trigger calls |

    <Warning>
      `SUPABASE_SERVICE_ROLE_KEY` bypasses all Row Level Security. Keep it out of any client-side code or logs.
    </Warning>
  </Step>
</Steps>

---

## Step 4 — Configure Supabase Auth

<Steps>
  <Step title="Enable email auth">
    In Supabase Studio → Authentication → Providers, enable **Email** and disable **Confirm email** (Forager uses invite-based flow — users accept via link, not confirmation code).
  </Step>
  <Step title="Set redirect URLs">
    In Authentication → URL Configuration:

    - **Site URL**: `https://forager.yourcompany.com`
    - **Additional redirect URLs**: `https://forager.yourcompany.com/**`

    This ensures invite links and password reset links redirect to your dashboard.
  </Step>
  <Step title="Configure invite email template (optional)">
    In Authentication → Email Templates → Invite User, you can customise the subject and body. The default Supabase template works, but removing Supabase branding is recommended for enterprise deployments.
  </Step>
</Steps>

---

## Step 5 — Configure Storage

The floor plan images uploaded by admins are stored in a Supabase Storage bucket.

<Steps>
  <Step title="Verify the bucket exists">
    The migration `20260506000004_floor_plans_bucket.sql` creates the `floor-plans` bucket and its RLS policies automatically. In Studio → Storage, confirm the bucket exists and is set to **private** (not public).
  </Step>
  <Step title="Verify storage policies">
    The migration `20260511000001_floor_plans_update_delete_policies.sql` adds update and delete policies. If you see permission errors when admins upload floor plans, re-apply these two migration files.
  </Step>
</Steps>

---

## Step 6 — Deploy the Web Dashboard

The web dashboard is a Next.js application.

<Steps>
  <Step title="Get the dashboard source">
    Request the `forager-web` source bundle from Hiverise.
  </Step>
  <Step title="Install dependencies">
    ```bash
    cd forager-web
    npm install
    ```
  </Step>
  <Step title="Create the .env.local file">
    ```bash
    cat > .env.local << 'EOF'
    NEXT_PUBLIC_SUPABASE_URL=https://your-supabase-url
    NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
    SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
    NEXT_PUBLIC_SITE_URL=https://forager.yourcompany.com
    SUPER_ADMIN_EMAIL=admin@yourcompany.com
    EOF
    ```

    | Variable | Description |
    |---|---|
    | `NEXT_PUBLIC_SUPABASE_URL` | Your Supabase API URL (the `8000` port URL or reverse-proxied URL) |
    | `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase anon key (safe to expose client-side) |
    | `SUPABASE_SERVICE_ROLE_KEY` | Service role key — server-side only, never sent to browser |
    | `NEXT_PUBLIC_SITE_URL` | Your dashboard's public URL — used in auth redirects |
    | `SUPER_ADMIN_EMAIL` | The email address that has platform-admin access to `/admin` |
  </Step>
  <Step title="Build and start">
    ```bash
    npm run build
    npm start   # listens on port 3000 by default
    ```

    For production use, run behind a process manager:

    ```bash
    npm install -g pm2
    pm2 start npm --name forager-web -- start
    pm2 save
    ```
  </Step>
  <Step title="Set up a reverse proxy">
    Point your domain (`forager.yourcompany.com`) at the Next.js process using nginx or Caddy. Example nginx config:

    ```nginx
    server {
        listen 443 ssl;
        server_name forager.yourcompany.com;

        ssl_certificate /etc/letsencrypt/live/forager.yourcompany.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/forager.yourcompany.com/privkey.pem;

        location / {
            proxy_pass http://localhost:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    ```
  </Step>
</Steps>

---

## Step 7 — Get Your Custom Android APK

The Android app has your Supabase URL and anon key compiled into the binary. You cannot use the standard APK from the Hiverise downloads page.

<Steps>
  <Step title="Send Hiverise your credentials">
    Email [support@harbinge.rs](mailto:support@harbinge.rs) with:
    - Your `SUPABASE_URL`
    - Your `SUPABASE_ANON_KEY`
    - Your organisation name (used in the app's build config)

    Hiverise will build, sign, and return a custom APK within one business day.
  </Step>
  <Step title="Distribute the APK">
    Distribute the APK to your field techs via your MDM solution, an internal file share, or direct download from a URL you control. Installation instructions are the same as the standard APK — see the [Downloads page](/downloads) for sideload instructions.
  </Step>
</Steps>

---

## Step 8 — First Login and Provisioning

<Steps>
  <Step title="Log in as super admin">
    Go to `https://forager.yourcompany.com/admin`. Sign in with the email address you set as `SUPER_ADMIN_EMAIL`. This automatically provisions your Hiverise internal company and grants you admin access in the app.
  </Step>
  <Step title="Create your first company">
    Use **Add Company** to create your organisation and invite the first admin user. This is identical to the provisioning flow described in [Provisioning a New Company](/admin/provisioning-companies).
  </Step>
  <Step title="Accept the invite and sign in to the app">
    The invited admin checks their email, accepts the invite, and signs in to the Forager app using the custom APK. From there, setup follows the [Pilot Onboarding Guide](/admin/pilot-onboarding-guide).
  </Step>
</Steps>

---

## Ongoing Operations

### Backups

Back up the Postgres database regularly. The Supabase Docker stack stores data in a named volume (`supabase_db_data`). A simple pg_dump approach:

```bash
PGPASSWORD=YOUR_POSTGRES_PASSWORD pg_dump \
  -h localhost -U postgres -d postgres \
  --format=custom \
  > forager-backup-$(date +%Y%m%d).dump
```

Automate this with cron and ship the dump to offsite storage (S3, Backblaze B2, etc.).

### Applying schema updates

When Hiverise releases a schema update, you will receive new migration SQL files. Apply them in order:

```bash
PGPASSWORD=YOUR_POSTGRES_PASSWORD psql \
  -h localhost -U postgres -d postgres \
  -f YYYYMMDD_description.sql
```

Never skip migrations — each builds on the previous one.

### Updating the web dashboard

```bash
cd forager-web
# Replace source files with updated bundle from Hiverise
npm install
npm run build
pm2 restart forager-web
```

### Updating the Android APK

Send Hiverise the same credentials as Step 7 — your Supabase URL and anon key do not change between updates. A new signed APK will be returned and can be redistributed via your existing channel.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Invite emails not delivered | SMTP not configured in Supabase `.env` | Set `SMTP_HOST`, `SMTP_USER`, `SMTP_PASS` and restart the stack |
| App shows "Invalid API key" | APK built with wrong anon key | Request a new custom APK with the correct `SUPABASE_ANON_KEY` |
| Dashboard login redirects to wrong URL | `SITE_URL` mismatch | Ensure `SITE_URL` in Supabase Auth settings and `NEXT_PUBLIC_SITE_URL` in `.env.local` match exactly |
| Edge Function returns 500 | Missing secret | Run `supabase secrets list` and verify all secrets from Step 3 are set |
| Floor plan images fail to upload | Storage bucket missing or wrong RLS | Verify the `floor-plans` bucket exists and re-apply the two storage migrations |
| `/admin` page redirects away | `SUPER_ADMIN_EMAIL` not matching logged-in user | Confirm the env var matches the email address exactly (case-sensitive) |
| `classify-room` returns no match | Snapshots not reaching the function | Ensure `SUPABASE_ANON_KEY` secret is set on the function; check Edge Function logs in Studio |
