---
title: "Data Retention and Deletion Policy"
description: "Retention periods and deletion procedures for all Forager customer data — active customers, churned customers, and deletion requests"
---

<Warning>
  This document is for Hiverise Labs internal use and SOC 2 audit evidence. It is not customer-facing.
</Warning>

## Purpose

This policy defines how long Forager retains customer data, what happens to data when a customer churns, and how deletion requests are handled. It satisfies SOC 2 Type 2 Common Criteria CC6.5 (dispose of assets), and supports Privacy (P) criteria and applicable data protection regulations.

**Effective date:** 2026-06-07
**Owner:** Thaddeus Stepanovich, Hiverise Labs
**Review cadence:** Annually, or when a material change occurs in customer use cases or applicable law

---

## Data inventory

| Data type | Where stored | Sensitivity | Retention category |
|---|---|---|---|
| Asset records (CMDB items) | Supabase Postgres | Business-confidential | Customer operational data |
| Attestation records (scan history) | Supabase Postgres | Business-confidential | Compliance artifact |
| Location hierarchy (nodes, zones, floor plans) | Supabase Postgres | Business-confidential | Customer operational data |
| User profiles (name, email, role) | Supabase Postgres | PII | Customer operational data |
| Attestation photos | Supabase Storage | Business-confidential | Compliance artifact |
| Audit log entries | Supabase Postgres | Business-confidential | Compliance artifact |
| Webhook configurations and secrets | Supabase Postgres + Vault | Sensitive | Customer operational data |
| Sprint records | Supabase Postgres | Business-confidential | Customer operational data |
| Database backup archives | Backblaze B2 | Business-confidential | Backup |

---

## Retention periods

### Active customers

All data is retained for the duration of the active subscription. There is no scheduled purge of active customer data.

### Churned customers

When a customer's subscription ends (status set to `churned` or `suspended`):

| Phase | Duration | What happens |
|---|---|---|
| **Grace period** | 30 days from subscription end | Data fully retained; admin access suspended. Customer may reactivate and resume. |
| **Post-grace — awaiting deletion request** | Indefinite | Data retained pending deletion request. Access remains suspended. |
| **Post-grace — deletion requested** | Deleted within 30 days of written request | All customer data deleted per procedure below. |

**Rationale for indefinite post-grace retention without a deletion request:** Healthcare IT customers have regulatory obligations to retain asset records and maintenance attestations (typically 3–7 years per Joint Commission and CMS standards). Hiverise Labs retains data on behalf of churned customers until they affirmatively request deletion, ensuring customers can retrieve records during an audit after churning from Forager.

**Exception:** If a customer requests deletion, Hiverise Labs will delete within 30 days regardless of how recently the subscription ended.

### Compliance artifacts

Attestation records and audit log entries are treated as compliance artifacts with extended retention:

| Artifact | Retention period | Rationale |
|---|---|---|
| Attestation records (`attestations` table) | 7 years from creation | Healthcare asset maintenance records — Joint Commission / CMS audit exposure |
| Audit log entries (`audit_logs` table) | 7 years from creation | SOC 2 evidence; admin action accountability |
| Post-mortem documents | 7 years from creation | SOC 2 evidence |

These retention periods apply even after a customer churns and requests deletion of their operational data. Attestation and audit log records may be retained in anonymized or pseudonymized form (company_id and actor_id cleared) after the customer's operational data is deleted.

### Backup archives

| Backup type | Retention period |
|---|---|
| Supabase automated daily backups | 7 days (Supabase Pro) |
| Backblaze B2 weekly backup archives | 90 days, then deleted |

Backup archives are not a substitute for the deletion procedure below — when a customer's data is deleted from the production database, it will be purged from backup archives at the end of the applicable retention period.

---

## Deletion procedure

### Customer-requested deletion

When a churned customer submits a written deletion request (email to support@harbinge.rs):

**Step 1 — Acknowledge within 5 business days:**
Reply confirming receipt and stating that deletion will be completed within 30 days.

**Step 2 — Delete customer operational data:**

```sql
-- Identify the company
SELECT id, name FROM companies WHERE name = '<customer name>';

-- Delete in dependency order (child tables first)
DELETE FROM attestations WHERE company_id = '<company_id>';
DELETE FROM snapshots WHERE company_id = '<company_id>';
DELETE FROM webhook_configs WHERE company_id = '<company_id>';
-- (vault secrets are purged by the trg_cleanup_webhook_vault_secret trigger)
DELETE FROM sprints WHERE company_id = '<company_id>';
DELETE FROM profiles WHERE company_id = '<company_id>';
DELETE FROM location_nodes WHERE company_id = '<company_id>';
DELETE FROM assets WHERE company_id = '<company_id>';

-- Anonymize audit log entries (preserve for compliance, remove PII)
UPDATE audit_logs
SET company_id = NULL, actor_id = NULL
WHERE company_id = '<company_id>';

-- Delete the company record last
DELETE FROM companies WHERE id = '<company_id>';
```

**Step 3 — Delete attestation photos from Supabase Storage:**
```
Supabase Dashboard → Storage → attestation-photos
→ Filter by company_id path prefix → Delete all objects
```

**Step 4 — Delete Supabase auth users:**
```
Supabase Dashboard → Authentication → Users
→ Search by company domain → Delete all associated auth users
```

**Step 5 — Confirm and close:**
- Verify the company record is gone: `SELECT * FROM companies WHERE id = '<company_id>';` should return 0 rows
- Send deletion confirmation email to the customer
- Log the deletion in the table below

### User-level deletion (GDPR right to erasure)

If an individual user (not a company admin) requests deletion of their personal data:

1. Delete their Supabase Auth user account (invalidates sessions)
2. Delete their `profiles` row
3. In `audit_logs`: set `actor_id = NULL` where `actor_id = '<user_id>'` — preserve `actor_email` for accountability unless the user explicitly requests email erasure and no active legal hold applies

Individual user deletion does not affect the company's other data.

---

## Deletion log

Record every deletion here for SOC 2 evidence.

| Date | Customer / User | Type | Requested by | Completed by | Notes |
|---|---|---|---|---|---|
| — | — | — | — | — | Policy effective 2026-06-07 |

---

## Regulatory considerations

### GDPR

If any customer has EU employees whose data is stored in Forager (email addresses, display names in `profiles`), GDPR applies to that data. The deletion procedure above satisfies GDPR Article 17 (right to erasure) when applied within 30 days of a written request.

GDPR also requires a privacy notice informing users of their rights. If EU users are present, ensure the Forager privacy policy covers this.

### HIPAA

If a customer's use case involves Protected Health Information (PHI), HIPAA retention requirements override this policy — HIPAA requires retention of certain records for 6 years from creation or last use. If a BAA is executed with a customer, confirm retention obligations before deleting their data.

At current customer profile, asset tracking data (device locations, maintenance schedules, tech names) is generally not PHI. Confirm per customer before executing deletion.

---

## Document history

| Date | Change |
|---|---|
| 2026-06-07 | Initial version |
