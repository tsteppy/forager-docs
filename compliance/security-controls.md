---
title: "Security Controls — SOC 2 Type 2"
description: "Inventory of implemented security controls for Forager, mapped to SOC 2 Trust Service Criteria"
---

<Warning>
  This document is for Hiverise Labs internal use and SOC 2 audit evidence. It is not customer-facing.
</Warning>

## Purpose

This document is a living inventory of Forager's implemented security controls, mapped to SOC 2 Type 2 Trust Service Criteria (TSC). It is updated as controls are added or changed. Use it to:
- Brief SOC 2 auditors on control coverage
- Track gaps before the observation period begins
- Onboard new Hiverise Labs staff to the security posture

**Audit target:** SOC 2 Type 2, all five TSCs (Security, Availability, Confidentiality, Processing Integrity, Privacy).

---

## Implemented controls

### CC6 — Logical and Physical Access

#### MFA enforcement for admin accounts
- **Status:** Implemented (2026-06-07)
- **Mechanism:** Supabase Auth TOTP MFA. Authenticator Assurance Level (AAL) is checked server-side in the Next.js protected layout on every page load. Any admin or super-admin session with `aal1` (password-only) is redirected to `/mfa` before accessing any dashboard page.
- **Scope:** All accounts with `role: admin` or designated as `SUPER_ADMIN_EMAIL`. Field tech and lead accounts are not subject to this requirement (no access to sensitive admin operations).
- **Defense-in-depth:** RLS policies on `webhook_configs`, `profiles`, and `companies` require `get_my_aal() = 'aal2'` for write operations, independent of the application-layer redirect.
- **Re-enrollment:** Admins can replace their authenticator from Settings → Security → Replace authenticator.
- **Evidence:** `supabase/migrations/20260607000003_admin_mfa_rls.sql`, `forager-web/app/(protected)/layout.tsx`, `forager-web/app/mfa/page.tsx`

#### Webhook secret encryption at rest
- **Status:** Implemented (2026-06-07)
- **Mechanism:** Supabase Vault (`vault.secrets`). The `webhook_configs.secret_value` plaintext column was dropped and replaced with `secret_value_vault_id` (a UUID reference to the vault). Secrets are decrypted only at webhook delivery time via the `get_webhook_secrets()` Postgres function (service_role only). The decrypted value is never stored in application logs.
- **Client exposure:** The vault ID is stripped before query results reach the browser. Clients receive only a boolean `has_secret` indicating whether a secret is configured.
- **Cleanup:** A BEFORE DELETE trigger (`trg_cleanup_webhook_vault_secret`) purges the vault entry when a webhook row is deleted.
- **Evidence:** `supabase/migrations/20260607000002_webhook_vault_secrets.sql`

#### Role-based access control (RBAC)
- **Status:** Implemented
- **Mechanism:** Three-tier role model: `admin`, `lead`, `tech`. Enforced at the RLS layer via `get_my_role()` helper function on every protected table. Leads have site-scoped access via `is_in_my_scope()`. Techs have read-only access to their company's location and asset data.
- **Multi-tenancy:** Every query is scoped to `company_id` via `get_my_company_id()`. Cross-company data access is structurally impossible through the application.
- **Evidence:** All migrations from `20260521000001` onward

#### Session management
- **Status:** Implemented (Supabase Auth defaults)
- **Mechanism:** JWT-based sessions via Supabase Auth. Access tokens expire per Supabase Auth configuration. Refresh tokens are rotated on use. Sessions are invalidated server-side on sign-out.
- **Gap:** Explicit session expiry timeout and failed-login lockout policy are not yet configured beyond Supabase defaults. See backlog.

---

### CC7 — System Operations (Monitoring, Anomaly Detection)

#### Audit log
- **Status:** Implemented (2026-06-07)
- **Mechanism:** Append-only `audit_logs` table. Written exclusively via service_role — no client INSERT/UPDATE/DELETE RLS policies exist. Covers 16 action types across all admin and super-admin operations. See [Audit Log Policy](/compliance/audit-log-policy) for full details.
- **Evidence:** `supabase/migrations/20260607000004_audit_logs.sql`, `forager-web/lib/audit.ts`

#### Attestation immutability
- **Status:** Implemented
- **Mechanism:** The `attestations` table has no application-layer UPDATE or DELETE path. Records are insert-only (written by the Android app at scan time and by the presence confirmation flow). Any correction is a new record, not an edit. This makes the attestation history a tamper-evident audit artifact.
- **Evidence:** No UPDATE/DELETE mutations on `attestations` anywhere in the codebase

---

### CC8 — Change Management

#### Database migrations as version-controlled schema changes
- **Status:** Implemented
- **Mechanism:** All schema changes are applied via numbered SQL migration files in `forager/supabase/migrations/`. Migrations are applied in order using psql. No ad-hoc schema changes are made in the Supabase Studio SQL editor.
- **Gap:** No formal PR review gate enforces this — a developer could apply an unapproved migration. Change management policy formalization is in backlog.

---

### C — Confidentiality

#### Multi-tenant data isolation
- **Status:** Implemented
- **Mechanism:** Every table holding customer data has `company_id` scoped RLS policies. `get_my_company_id()` is a `SECURITY DEFINER` function that returns the company ID from the authenticated user's profile — it cannot be spoofed by a client passing a different company_id. Cross-tenant data access is structurally impossible through any application path.

#### Webhook credential protection
- **Status:** Implemented (Supabase Vault)
- See [Webhook secret encryption at rest](#webhook-secret-encryption-at-rest) above.

#### Photo attestation storage
- **Status:** Implemented
- **Mechanism:** Photos stored in a private Supabase Storage bucket (`attestation-photos`). Not publicly accessible. Dashboard generates short-lived (1-hour) signed URLs for viewing. Bucket RLS is scoped to company.

---

### PI — Processing Integrity

#### Attestation record completeness
- **Status:** Implemented
- **Mechanism:** Every confirmed presence event writes an `attestations` row with: actor ID, asset ID, location node ID, result (match/mismatch/new_asset), timestamp, and optional photo URL. No scan can succeed without producing a record — the confirmation flow is synchronous before returning a result to the tech.

#### PMI schedule tracking
- **Status:** Implemented
- **Mechanism:** `next_pmi_date` and `pmi_interval_days` are first-class columns on `assets`. The `asset_report` view computes `pmi_status` (overdue/due_soon/ok/none) from these values. PMI data is included in CSV import/export, making it auditable alongside other asset attributes.

---

## Backlog — Controls not yet implemented

These are planned but not yet in place. They must be addressed before the SOC 2 observation period begins.

| Control | Priority | Status |
|---|---|---|
| Incident response plan | High | **Done** — see [Incident Response Plan](/compliance/incident-response-plan) |
| Vendor risk assessments (Supabase, Vercel, Cloudflare, Backblaze) | High | **Done** — see [Vendor Risk Assessments](/compliance/vendor-risk-assessments) |
| Supabase BAA (required if any customer use case involves PHI) | High | **Action required** — confirm use cases and execute BAA if needed |
| Formal change management gates (PR review requirements) | Medium | Not enforced |
| Session expiry and failed-login lockout policy | Medium | At Supabase defaults |
| Credential rotation schedule | Medium | Not defined |
| Annual penetration test | Medium | Not scheduled |
| Data retention and deletion policy | Medium | Not implemented |
| Super-admin `/admin` route MFA step-up | Lower | Not implemented |
| Storage bucket policy audit | Lower | Not reviewed |
| Failed auth event surfacing in Grafana/Loki | Lower | Not wired |

---

## Document history

| Date | Change |
|---|---|
| 2026-06-07 | Initial version. MFA enforcement, webhook Vault encryption, and audit log all shipped. |
| 2026-06-07 | IR plan and vendor risk assessments documented. Supabase BAA action item flagged. |
