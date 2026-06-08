---
title: "Forager Admin Operations Guide"
description: "Complete reference for Hiverise Labs staff: provisioning, onboarding, billing, sprints, and self-hosting"
---
	
<Warning>
  This guide is for **Hiverise Labs staff only**. All workflows described here require access to the platform admin dashboard or direct database access not available to customers.
</Warning>

## Overview

As a Forager platform admin, you are responsible for the full lifecycle of a customer account: provisioning it, onboarding the team, managing billing, and supporting the customer through their pilot and into production. This guide consolidates every operational workflow into one reference.

**Quick navigation:**
- [Provisioning a company](#provisioning-a-company) — creating new accounts and sending invites
- [Pilot onboarding runbook](#pilot-onboarding-runbook) — end-to-end from first contact to first attestation
- [Billing management](#billing-management) — subscription status, dunning, grace period, suspension
- [Sprint management](#sprint-management) — creating and managing tech competitions
- [Self-hosting](#self-hosting) — enterprise on-premises deployments

---

## Accessing the platform admin dashboard

Log in at [dashboard.hiveriselabs.com](https://dashboard.hiveriselabs.com) with your Hiverise Labs admin account. Navigate to `/admin`.

<Note>
  Only the account registered as `SUPER_ADMIN_EMAIL` in the Forager environment can access `/admin`. Any other account is redirected to the standard customer dashboard.
</Note>

The `/admin` page shows:
- A list of all provisioned companies with color-coded status badges (green = Active, amber = Grace Period, red = Suspended)
- The **Add Company** form for provisioning new accounts
- Clickable company names that open the detail page for billing management

When you visit `/admin`, the dashboard automatically ensures your Hiverise Labs account has `role: admin` in the Forager app and creates an internal **Hiverise Labs** company if needed. This is idempotent — it does nothing if your profile is already correct.

---

## Provisioning a company

When a new customer is signed, create their account from the `/admin` dashboard. This creates the company row, assigns their first admin, and sends the invite email in one step.

<Steps>
  <Step title="Open the Add Company form">
    Click **Add Company** at the top of the `/admin` page. The form expands inline.
  </Step>
  <Step title="Fill in the company details">
    | Field | Required | Notes |
    |---|---|---|
    | **Company name** | Yes | Exactly as it should appear in the system |
    | **Admin email** | Yes | The IT admin's email at the customer site |
    | **Seat limit** | Yes | Admin + number of field techs. Default: 6. |
    | **Notes** | No | Internal only — contract type, pilot scope, primary contact, start date |
  </Step>
  <Step title="Submit">
    Click **Create Company & Send Invite**. On success, a green banner confirms the company name and the email the invite was sent to.

    If the invite email fails (e.g., the address already has a conflicting Supabase auth state), the company row is automatically deleted to prevent orphaned records. The Supabase error is shown inline — resolve it and retry.
  </Step>
</Steps>

**What happens next:** The admin receives an email with subject "You have been invited." When they click the link, their Forager account is created and verified automatically — no password required. Their profile is seeded with `role: admin` and their company association. They land on a welcome page at `dashboard.hiveriselabs.com/auth/accept-invite` with a **Get it on Google Play** button.

**Seat limits:** Enforced at invite time by the `invite-user` function. You cannot edit seat limits from the dashboard after provisioning — update `seat_limit` directly in the database if needed.

---

## Pilot onboarding runbook

A Forager pilot runs from first contact through a customer's first full attestation cycle — typically 2–4 weeks. Follow each phase in order. Each phase has a verification step; do not move on until it passes.

### Phase 0 — Pre-engagement checklist

Collect before you touch the admin dashboard:

| Item | Notes |
|---|---|
| **Customer organization name** | Exactly as it should appear in the system |
| **Admin's name and email** | The person who will manage the Forager account |
| **Pilot scope** | Which building(s), floor(s), and approximate room count |
| **Asset source** | ServiceNow/CMDB export? CSV? Manual entry? |
| **Tech device count** | How many Android devices for field techs |
| **Seat limit needed** | Admin + field tech count (default 6) |
| **Floor plan files** | JPEG or PNG preferred. PDFs work but lose interactive pin support. |
| **Timeline** | Pilot start date and expected completion |

### Phase 1 — Provision the company

Follow the [Provisioning a company](#provisioning-a-company) workflow above.

**Verify:** You see the green success banner and the company appears in the `/admin` list.

### Phase 2 — Customer admin accepts the invite

Send the customer admin this message (adapt as needed):

> *"I've set up your Forager account. Look for an email with the subject "You have been invited" — it may take a few minutes. Click the link and you'll land on a welcome page confirming your account is active. No password setup is required at this step.*
>
> *After that, go to [dashboard.hiveriselabs.com](https://dashboard.hiveriselabs.com) and sign in with this email address. You'll have full admin access."*

**Watch for:**
- No email within 5 minutes → have them check spam. Sender is `no-reply@mail.app.supabase.io`.
- Expired invite link (valid for 24 hours) → re-provision a new invite. This does not create a duplicate company.

**Verify:** Ask the admin to confirm they can log into the dashboard and see the **Team** page.

### Phase 3 — Team setup

Coach the customer admin through inviting their field techs from **Dashboard → Team → Invite Team Member**. Each tech receives an invite email, clicks the link, lands on the welcome page, downloads the app, and signs in.

<Note>
  Invites expire after 24 hours. If a tech misses the window, the admin can re-send from the Team page. Techs do not set a password — they sign in with the email from the invite.
</Note>

**Verify:** At least one field tech appears in the Team list and can sign in to the Forager app.

### Phase 4 — Location hierarchy

Walk the admin through building the location tree in the Forager app: **Admin → Location Hierarchy → Manage Locations**. Build top-down using the **⋮** overflow menu on any node and choosing **Add child**.

Build order:
1. Campus or site *(optional — skip for single-building pilots)*
2. Building
3. Floor *(one node per floor in scope)*
4. Room *(one node per room to be tracked)*

For a typical pilot, scope to **1–2 floors, 10–15 rooms**. Keep it small.

<Note>
  **Enterprise multi-building customers:** The **Site** level picker in the Location Hierarchy card controls which depth counts as an access boundary (default: Building). If the hierarchy has a regional level above buildings (Region → Campus → Building → Floor → Room), update the site level to **Building** after the tree is built.
</Note>

Then upload floor plans: **⋮** on a floor node → **Add / replace floor plan**. Upload a JPEG or PNG. The admin places each room's pin on the floor plan image — 5–10 minutes per floor, one-time.

**Verify:** Every room in scope has a node. Floor plan images are uploaded and room pins are placed.

### Phase 5 — Survey

Brief the tech team before they start:
- One person with the app per room — they do not need to stay long
- Location permission and Bluetooth must be enabled
- Move room to room; one capture per room
- Do not scan assets during survey — survey and attestation are separate steps

**Survey steps (in the app):** Survey Mode → Select floor → Select room → Tap Capture → hold still ~30 seconds → move to next room.

After the sweep: **Coverage** screen. All pilot rooms should show **green** before the first attestation run. Yellow rooms still work but may have slightly lower confidence.

**Verify:** All rooms in scope are green in Coverage.

### Phase 6 — Asset seeding

Assets must exist in Forager before techs can scan them.

**Path A — CMDB import (preferred):** Dashboard → **Assets** → **Import CSV**. Customer exports a CSV from ServiceNow or their CMDB with columns: `asset_tag`, `name`, `description`, `serial_number`, `cmdb_id`, `cmdb_location`. Only `asset_tag` is required. Import only assets in the rooms being piloted — 50–200 assets is the right pilot scope.

**Path B — Manual:** If no CMDB is available, prepare a small CSV with known asset tags or have the customer add assets manually in the dashboard.

**Verify:** Assets appear in the dashboard Assets list and a specific asset tag can be found by search.

### Phase 7 — First attestation run

<Steps>
  <Step title="Tech enters a surveyed room">
    The tech opens the Forager app and walks into a surveyed room. The app automatically detects the room from the RF fingerprint.
  </Step>
  <Step title="Tech scans an asset barcode">
    The tech scans a barcode of a known asset in that room. The app confirms the current room, looks up the asset's recorded location, and records a confirmation (match) or mismatch.
  </Step>
  <Step title="Verify in the dashboard">
    Go to **Floor Plan** → select the floor. The scanned room's pin should show the asset with a green indicator (confirmed within 7 days). Or check **Reports** → filter by today → the attestation appears with tech name, asset, room, and timestamp.
  </Step>
</Steps>

**Pilot is live when:** at least one attestation appears in the dashboard from a real scan in a real room.

### Weekly pilot health checks

Review these with the customer admin after the first week:

| Check | What to look for |
|---|---|
| **Coverage drift** | Rooms turning yellow/gray since survey? Re-survey if the physical layout changed. |
| **Mismatch rate** | A small number of mismatches is normal. High rate → re-survey the room or assets have moved. |
| **Inactive techs** | Team → Last Active column. No recent activity → tech may not have completed setup. |
| **Seat limit** | Team shows member count vs. limit. Need more? Update `seat_limit` in the database. |

### Pilot success criteria

A pilot is considered successful when all five are met:

- [ ] All rooms in scope are green in Coverage
- [ ] All pilot assets are loaded in the system
- [ ] At least one full team (admin + field techs) has active accounts
- [ ] Attestations appear in the dashboard from real scans
- [ ] The customer admin can pull a report showing confirmed asset locations

When all five are checked, the pilot is ready to convert to production. Raise the seat limit and onboard additional buildings following Phases 4–6.

### Common issues

| Symptom | Likely cause | Fix |
|---|---|---|
| Admin never received invite | Spam filter, or email typo during provisioning | Check spam; if typo, provision new invite with correct email |
| Admin accepted invite but no company data in dashboard | Profile seeding failed | Have them sign out and back in; ensure they clicked the email link, not the Supabase dashboard |
| Tech accepted invite but app shows no company | Same profile seeding issue | Tech signs out and back in |
| Room not detected during attestation | Room not surveyed, or survey too brief | Re-survey with a longer capture |
| Asset not found on scan | Asset tag not in the system | Check Assets list; import the missing asset |
| Seat limit error when inviting a tech | Company at capacity | Increase `seat_limit` in the database |

---

## Billing management

Each company has a `subscription_status` that controls what its users can do. You update this manually from the company detail page in `/admin`. There is no automated billing system — invoices are sent manually and status is updated by hand.

### Subscription statuses

| Status | Customer experience | Database access |
|---|---|---|
| **Active** | Full access — normal operation | Read and write |
| **Grace Period** | Amber warning banner on every dashboard page. App and dashboard fully functional. | Read only — new scans, asset updates, and team changes are blocked |
| **Suspended** | Dashboard redirects to a suspension screen. App scans fail silently at the database layer. | No access |

The grace period banner reads: *"Your subscription payment is overdue. Access will be restricted soon. Contact support."*

The suspension screen reads: *"Your Forager account has been suspended due to a lapsed subscription. Your data is safe and will be restored when your account is reinstated."*

### Changing a company's status

<Steps>
  <Step title="Open the company detail page">
    Go to `/admin` and click the company name. The detail page shows an activity dashboard and a **Billing** card at the top.
  </Step>
  <Step title="Select the new status">
    In the Billing card, select the desired status using the radio buttons:
    - **Active** — normal operation
    - **Grace Period (read-only)** — payment overdue, reads allowed, writes blocked
    - **Suspended (no access)** — full lockout

    The Save button is disabled until you change from the current value.
  </Step>
  <Step title="Save">
    Click **Save**. The change takes effect immediately — no restart or cache clear needed. The next time any user from that company loads a page, they see the updated behavior.
  </Step>
</Steps>

### Dunning workflow

| Day | Action |
|---|---|
| 0 | Send invoice (net-30 terms) |
| 7 | Send payment reminder email |
| 14 | Send warning email — account will enter read-only mode in 7 days |
| 21 | Set status → **Grace Period** in the admin dashboard |
| 30 | Set status → **Suspended** if still unpaid |

When payment is received, set status back to **Active** immediately. The customer regains full access on their next page load.

---

## Photo Attestation

Photo Attestation captures a tight crop of the physical asset tag at the moment of each camera-mode barcode scan, stores it in Supabase Storage, and links it to the attestation record. Admins and compliance officers can then view the photos in the Attestation Log on the web dashboard.

This feature is opt-in per company and disabled by default.

### When to enable it

Enable Photo Attestation for customers who need physical proof-of-presence for audits — typically those working toward Joint Commission accreditation, ISO 55000 compliance, or internal IT policy that requires evidence a tech directly witnessed a device.

### Enabling Photo Attestation

<Steps>
  <Step title="Open the company detail page">
    Go to `/admin` and click the company name.
  </Step>
  <Step title="Find the Photo Attestation toggle">
    In the **Billing** card, scroll below the subscription status controls to the **Photo Attestation** section.
  </Step>
  <Step title="Enable">
    Check the **Capture asset tag photo on each camera scan** checkbox. The setting saves immediately — no separate Save button.
  </Step>
</Steps>

The Android app picks up the setting on the tech's next login (or app restart). From that point on, every camera scan automatically captures a photo.

### What techs see

Techs see no additional UI when photo attestation is active — the capture is silent and non-blocking. The scan result (match, mismatch, etc.) appears immediately; the photo uploads in the background.

### Limitations

- **Hardware scanner mode only:** No photo is captured. Attestations are still recorded normally without a photo.
- **Camera mode only:** The feature has no effect on Zebra/Honeywell hardware trigger devices unless the tech switches to camera fallback.
- **Best-effort upload:** If the device is offline at scan time, the attestation is recorded locally and syncs when connectivity returns. The photo upload also retries, but there is no guarantee a photo will be available for every offline scan.

### Storage

Photos are stored in a private Supabase Storage bucket (`attestation-photos`) scoped to each company's folder. Photos are not publicly accessible — the dashboard generates short-lived signed URLs (1 hour) for viewing.

Estimated storage: 15–40 KB per scan. At 500 scans/day, one active company uses roughly 300–600 MB/month.

### What is and is not blocked in Grace Period

Grace Period is read-only at the database layer via RLS — applies to both the web dashboard and the Android app.

**Blocked:**
- Submitting new scans (Android app)
- Adding or updating asset records
- Confirming asset presence (attestations)
- Assigning team members to sites
- Modifying the location hierarchy

**Still accessible:**
- Viewing the dashboard, reports, floor plans, and asset registry
- Viewing team membership and location hierarchy
- Downloading existing data

### Data retention

Customer data is never deleted when a company is suspended. All records remain intact and are immediately restored when the account is reinstated. Leaving an account in Suspended status indefinitely does not risk data loss.

---

## Sprint management

Sprints are time-limited competitions that give field techs a focused window to earn points for confirming assets. Regular sprints keep confirmation rates high and surface stale inventory before it becomes a compliance problem.

Sprints are managed from **Admin → Sprints** in the Forager Android app.

### Creating a sprint

1. Open the app → **Admin** → scroll to **Sprints** → tap **+ New Sprint**
2. Fill in the form:

| Field | Required | Notes |
|---|---|---|
| **Sprint name** | Yes | e.g. "May Blitz" or "Q2 Week 1" |
| **Description** | No | Optional context for the team |
| **Start date** | Yes | `YYYY-MM-DD`. Begins midnight UTC. |
| **End date** | Yes | `YYYY-MM-DD`. Ends 11:59 PM UTC. |

3. Tap **Create Sprint**.

If the start date is today or in the past, the sprint immediately becomes active and a **LIVE** badge appears.

<Warning>
  Only one sprint can be active at a time. If a sprint is running, schedule the next one to start the day after it ends. Overlapping sprints are rejected.
</Warning>

### Active sprint behavior

While a sprint is running:
- A **LIVE** badge appears in Admin → Sprints
- A sprint banner appears on the home screen of all techs, showing sprint name, countdown, personal score, and rank
- The Sprint period becomes available in Most Wanted → Leaderboard
- Assets confirmed during the sprint are locked and cannot be re-confirmed by another tech until it ends

### Ending a sprint early

Open **Admin → Sprints** → find the LIVE sprint → tap **End Early**. The end date is updated to now. All points and confirmation history are preserved.

### Sprint strategy

- **1–2 week windows** produce the best results — long enough for full building coverage, short enough to maintain urgency
- **Schedule the next sprint before the current one ends** to avoid a gap with no visible banner
- **Target stale assets** — 🔥 10-point assets are devices not confirmed in 90+ days; a sprint focused on clearing those is more valuable than one that re-confirms recently-seen devices
- **Name sprints clearly** — techs see the name on their home screen throughout the competition

---

## Self-hosting

Enterprise customers deploying Forager on their own infrastructure require a custom build and a full stack deployment. Contact the customer before starting — Hiverise provides a migration bundle and a custom-compiled APK.

**Components:**

| Component | What it is | Hosted by customer |
|---|---|---|
| **Supabase** | Postgres, Auth, Edge Functions, Storage | ✓ |
| **Web Dashboard** | Next.js admin and reporting UI | ✓ |
| **Edge Functions** | Deno serverless functions | ✓ (via Supabase) |
| **Android APK** | Field tech mobile app | Custom build from Hiverise |

The Android APK has the customer's Supabase URL and anon key compiled in. The standard APK from the downloads page cannot be reused — request a custom build once the Supabase instance is running.

For the full step-by-step deployment walkthrough, see the dedicated [Self-Hosting Guide](/admin/self-hosting).

**High-level steps:**
1. Deploy Supabase via Docker Compose on a Linux server (4 GB RAM minimum)
2. Apply all Forager database migrations in order (migration bundle from Hiverise)
3. Deploy the five Edge Functions and set secrets
4. Configure Supabase Auth (email auth, redirect URLs, invite template)
5. Verify the `floor-plans` Storage bucket and RLS policies
6. Deploy the Next.js web dashboard with `.env.local` configured
7. Request a custom APK from Hiverise (provide `SUPABASE_URL` and `SUPABASE_ANON_KEY`)
8. Log in as super admin, provision the first company, and complete onboarding

**Ongoing operations:**
- **Backups:** `pg_dump` on a cron schedule → offsite storage (S3, Backblaze B2)
- **Schema updates:** Apply new migration SQL files in order using `psql`
- **Dashboard updates:** Replace source bundle → `npm install && npm run build && pm2 restart forager-web`
- **APK updates:** Send Hiverise your unchanged Supabase credentials — a new signed APK is returned within one business day

---

## Reference

### Pricing

| Tier | Rate |
|---|---|
| 1–999 devices | $15/device/year |
| 1,000–9,999 devices | $12/device/year |
| 10,000+ devices | Custom quote |
| Onboarding fee | $1,000 one-time (at contract signing) |
| Monthly option | Available at 20% premium over annualized rate |
| Self-hosting | Custom quote only, annual prepaid |

Billing: manual PDF invoicing, net-30. Stripe deferred until ~10+ customers.

### Environment variables (cloud deployment)

| Variable | Location | Purpose |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Vercel env | Supabase API URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Vercel env | Supabase anon key (client-safe) |
| `SUPABASE_SERVICE_ROLE_KEY` | Vercel env (server-only) | Admin operations bypassing RLS |
| `SUPER_ADMIN_EMAIL` | Vercel env | Gates the `/admin` panel |

### Key contacts

| Role | Contact |
|---|---|
| Customer support | support@harbinge.rs |
| Custom APK requests | support@harbinge.rs |
| Self-hosting migration bundles | support@harbinge.rs |
