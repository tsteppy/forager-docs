---
title: "Audit Log Policy"
description: "Forager audit log scope, immutability guarantees, access controls, and retention — for SOC 2 Type 2 auditors and internal review"
---

<Warning>
  This document is for Hiverise Labs internal use and SOC 2 audit evidence. It is not customer-facing.
</Warning>

## Purpose

This policy defines how Forager records, protects, and retains audit log entries for privileged administrative actions. It supports the SOC 2 Type 2 Trust Service Criteria for Security (CC6, CC7) and Processing Integrity.

---

## Scope

The audit log covers all privileged actions taken by:
- **Hiverise Labs super-admins** — platform-level operations (company provisioning, billing, notices)
- **Customer admins** — company-scoped operations (team management, webhooks, sprints, location structure)

End-user field tech actions (barcode scans, attestations, presence confirmations) are recorded separately in the `attestations` and `snapshots` tables, which are immutable by design (no application code path updates or deletes those records).

---

## What is logged

Every entry in `audit_logs` captures:

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Unique log entry identifier |
| `actor_id` | UUID | Auth user ID of the person who performed the action |
| `actor_email` | TEXT | Email address of the actor — denormalized and preserved even if the account is later deleted |
| `action` | TEXT | Snake-case action verb (see table below) |
| `resource_type` | TEXT | Category of the affected object (`user`, `webhook`, `sprint`, `company`, `notice`, `location_node`) |
| `resource_id` | TEXT | UUID or other identifier of the specific affected record |
| `company_id` | UUID | Company scope of the action — NULL for super-admin platform actions |
| `meta` | JSONB | Action-specific context (old/new values, counts, names) |
| `created_at` | TIMESTAMPTZ | UTC timestamp of the action |

### Logged action types

| Action | Who | What happened |
|---|---|---|
| `provision_company` | Super-admin | New customer company created |
| `set_company_status` | Super-admin | Subscription status changed |
| `set_photo_attestation` | Super-admin | Photo attestation feature toggled |
| `create_notice` | Super-admin | In-app broadcast notice created |
| `deactivate_notice` | Super-admin | In-app notice deactivated |
| `invite_member` | Admin | User invited to join a company |
| `update_member_role` | Admin | User role changed (admin / lead / tech) |
| `remove_member` | Admin | User removed from a company |
| `create_webhook` | Admin | Webhook endpoint configured |
| `update_webhook` | Admin | Webhook endpoint modified |
| `toggle_webhook` | Admin | Webhook enabled or disabled |
| `delete_webhook` | Admin | Webhook endpoint deleted |
| `create_sprint` | Admin | Remediation sprint created |
| `close_sprint` | Admin | Remediation sprint closed |
| `create_zone` | Admin | Location zone created on floor plan |
| `assign_assets_to_zone` | Admin / Lead | Assets bulk-assigned to a zone |

---

## Immutability

The `audit_logs` table has Row Level Security enabled. There are **no client-facing INSERT, UPDATE, or DELETE policies** — only a SELECT policy for company-scoped admin reads.

All writes go through the Supabase service role key (`SUPABASE_SERVICE_ROLE_KEY`), which is a server-side secret stored in Vercel environment variables and never exposed to the browser. This key is used exclusively by:
- Server Actions in the Next.js web dashboard (`lib/audit.ts → logAction()`)
- The `invite-user` Supabase Edge Function

This means:
- No authenticated user can insert, modify, or delete audit log entries through the application
- No RLS policy exists that would permit such operations
- Tampering requires direct database access, which requires Supabase project credentials

### Failure handling

`logAction()` is designed to never throw. If the write fails (e.g., transient DB error), the failure is recorded to server-side stdout (captured by Vercel logs) but does **not** block or roll back the underlying operation. This ensures audit log failures are observable without disrupting customer workflows.

---

## Access controls

| Role | Access |
|---|---|
| **Super-admin** | Full read access via Supabase Studio or psql |
| **Customer admin** | Read access to their own company's log entries via RLS (`company_id = get_my_company_id()`) |
| **Customer lead / tech** | No access |
| **Unauthenticated** | No access |

A web-based audit log viewer within the admin dashboard is planned. Until it ships, super-admins use Supabase Studio or psql; customer admins do not currently have a UI view.

---

## Retention

Audit log entries are retained **indefinitely** with no scheduled purge.

If a customer company is offboarded and its `companies` row is deleted, the corresponding `company_id` on log entries is set to NULL (via `ON DELETE SET NULL` on the foreign key). The entries themselves are preserved.

If an admin user account is deleted, their `actor_id` is set to NULL on their entries. Their `actor_email` (denormalized at write time) is preserved for identity traceability.

---

## Review cadence

Hiverise Labs will review the audit log for anomalies on the following schedule:

| Frequency | Scope |
|---|---|
| **Monthly** | Super-admin actions: provisioning, billing status changes, notice creation |
| **Quarterly** | Cross-company anomaly scan: role escalations, bulk webhook changes |
| **On incident** | Full log review for affected company and actor, scoped to the incident window |

---

## Evidence for SOC 2 auditors

Auditors requesting evidence of logging controls should be directed to:

1. **Schema**: `supabase/migrations/20260607000004_audit_logs.sql` — table definition, indexes, RLS policies
2. **Write path**: `forager-web/lib/audit.ts` — `logAction()` function using service_role
3. **RLS verification**: Run `SELECT * FROM pg_policies WHERE tablename = 'audit_logs';` — confirms no client INSERT/UPDATE/DELETE policies exist
4. **Sample log entries**: Available via Supabase Studio → Table Editor → audit\_logs, or via psql

To demonstrate a specific logged action to an auditor, perform the action and then show the resulting row in `audit_logs`.
