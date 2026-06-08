---
title: "Vendor Risk Assessments"
description: "Risk assessments for critical Forager subprocessors — Supabase, Vercel, Cloudflare, Backblaze B2"
---

<Warning>
  This document is for Hiverise Labs internal use and SOC 2 audit evidence. It is not customer-facing.
</Warning>

## Purpose

This document records risk assessments for the critical third-party subprocessors on which Forager depends. It satisfies SOC 2 Type 2 Common Criteria CC9.2 (manage vendor and business partner risks).

**Assessment date:** 2026-06-07
**Owner:** Thaddeus Stepanovich, Hiverise Labs
**Review cadence:** Annually, or when a vendor has a material security event or service change

---

## Subprocessor inventory

| Vendor | Role | Data processed | Risk tier |
|---|---|---|---|
| **Supabase** | Database, Auth, Storage, Edge Functions | All customer data (assets, attestations, profiles, photos) | Critical |
| **Vercel** | Web dashboard hosting, serverless functions | Session tokens, server action inputs/outputs | High |
| **Cloudflare** | DNS for `harbinge.rs` | DNS queries only — no payload data | Medium |
| **Backblaze B2** | Offsite backup storage | Database backup archives | High |

---

## Supabase

**Role in Forager:** Supabase is the foundational infrastructure layer. It hosts the Postgres database (all customer data), Supabase Auth (identity and sessions), Supabase Storage (attestation photos, APK files), and the Deno Edge Function runtime. A Supabase outage or breach would be a critical incident for Forager.

**Data processed:**
- All asset records, attestation records, location hierarchies, floor plans
- User profiles, email addresses, session tokens, MFA factors
- Webhook configurations (secrets stored encrypted in Supabase Vault)
- Attestation photos in Supabase Storage
- Audit log entries

**Security posture:**
- SOC 2 Type 2 certified (report available on request from Supabase)
- ISO 27001 certified
- Data encrypted at rest (AES-256) and in transit (TLS 1.2+)
- Supabase Vault for application-level secret encryption (used for webhook secrets)
- Row Level Security enforced at the database layer
- MFA available for Supabase dashboard access

**BAA status:**
<Warning>
  **Action required:** A Business Associate Agreement (BAA) with Supabase is required before signing any customer who processes Protected Health Information (PHI) through Forager. Supabase BAAs are available on the **Team plan** ($599/month) or higher.

  Asset tracking data (device locations, maintenance schedules, tech names) is generally not PHI. However, if a customer's use case explicitly links asset data to patient care records or patient locations, a BAA is required.

  **Before signing any healthcare enterprise customer:** Confirm the use case does not involve PHI, or upgrade to Supabase Team plan and execute the BAA. Contact Supabase support at supabase.com/contact/enterprise to initiate.
</Warning>

**Current plan:** Confirm and document before the SOC 2 observation period.

**Risk rating:** Critical — no mitigation available if Supabase is unavailable. Accepted: Supabase's security posture and certifications are strong. Mitigated by: weekly automated backups to Backblaze B2, enabling recovery to a self-hosted Supabase instance if needed.

**Vendor contacts:**
- Status page: status.supabase.com
- Security: security@supabase.com
- Enterprise/BAA: supabase.com/contact/enterprise

---

## Vercel

**Role in Forager:** Vercel hosts the Forager Next.js web dashboard (`dashboard.hiveriselabs.com`). Server Actions execute in Vercel's serverless function runtime, which means transient copies of request/response data (including admin form inputs and server action outputs) pass through Vercel's infrastructure. Vercel does not have persistent access to customer data — it is a compute layer, not a storage layer.

**Data processed:**
- HTTP request bodies from the web dashboard (admin form submissions, webhook configs, team management actions)
- Session tokens and JWT claims (passed in cookies on every authenticated request)
- Server-rendered HTML containing customer data (asset lists, team lists, reports)
- Build artifacts and source code

**Security posture:**
- SOC 2 Type 2 certified (report available at vercel.com/security)
- Data encrypted in transit (TLS 1.3)
- No persistent storage of request/response payloads beyond standard logging
- Environment variables encrypted at rest (used for `SUPABASE_SERVICE_ROLE_KEY`, `SUPER_ADMIN_EMAIL`)
- Preview deployments scoped to Vercel team members only — customer data is never in preview environments

**BAA status:** Vercel does not offer BAAs for standard plans. Because Vercel processes transient request data (not persistent storage), the BAA obligation depends on whether the transient data constitutes PHI. Assess per customer use case alongside the Supabase BAA decision.

**Risk rating:** High — a Vercel outage takes the web dashboard offline but does not affect Android app functionality or data integrity. Customer data remains safe in Supabase. Accepted: Vercel's reliability track record is strong (99.99% uptime SLA on Pro).

**Vendor contacts:**
- Status page: vercel-status.com
- Security: security@vercel.com

---

## Cloudflare

**Role in Forager:** Cloudflare manages DNS for `harbinge.rs` (the company domain and docs site). It does not proxy or cache the Forager web dashboard or API — it is used for DNS resolution only. `dashboard.hiveriselabs.com` resolves directly to Vercel; Cloudflare is not in the request path.

**Data processed:**
- DNS query logs for `harbinge.rs` subdomains
- No application payload data

**Security posture:**
- SOC 2 Type 2 certified
- DNSSEC available (not currently enabled — review for enablement)
- Two-factor authentication enforced on Cloudflare account

**BAA status:** Not applicable — Cloudflare processes no PHI or customer application data.

**Risk rating:** Medium — a Cloudflare outage would make `harbinge.rs` domains unresolvable but would not directly affect `dashboard.hiveriselabs.com` (hosted on Vercel with its own DNS). Docs site (`docs.harbinge.rs`) would be affected.

**Vendor contacts:**
- Status page: cloudflarestatus.com
- Security: cloudflare.com/trust-hub

---

## Backblaze B2

**Role in Forager:** Backblaze B2 stores offsite backup archives. Forager does not write to B2 directly — backups originate from the homelab (`l1ghth0use`) and include Supabase database exports. B2 is a recovery resource, not part of the production data path.

**Data processed:**
- Supabase database backup archives (compressed, but not independently encrypted at the application layer — relies on B2's server-side encryption)
- Immich photo library backups (separate from Forager — shared B2 account)

**Security posture:**
- SOC 2 Type 2 certified
- Server-side encryption (AES-256) for all stored objects
- Access controlled via application keys (scoped per bucket)
- No public bucket access

**BAA status:** Not applicable for current use — backup archives do not contain PHI under current customer use cases. If PHI processing is confirmed for a customer, backups containing their data would require a Backblaze BAA. Backblaze offers BAAs — contact sales.backblaze.com.

**Encryption gap:**
<Warning>
  **Action required:** Database backup archives sent to B2 rely on Backblaze's server-side encryption only. The archives themselves are not independently encrypted before upload. If B2 is compromised, backup contents are accessible.

  Recommendation: Encrypt backup archives with a symmetric key (e.g., GPG) before uploading to B2. Key stored separately from the backup. Address before the SOC 2 observation period.
</Warning>

**Risk rating:** High (as a backup system — data loss risk if B2 and Supabase both fail simultaneously). Accepted for current stage. Mitigated by: Supabase's own internal redundancy, which makes B2 a recovery-of-last-resort rather than primary storage.

**Vendor contacts:**
- Status page: status.backblaze.com
- Security / BAA: backblaze.com/business/backblaze-business-storage-agreement

---

## Action items before SOC 2 observation period

| Item | Owner | Priority |
|---|---|---|
| Confirm Supabase plan tier and execute BAA if any customer use case involves PHI | Thaddeus | High |
| Assess Vercel BAA requirement per customer use case | Thaddeus | High |
| Enable DNSSEC on Cloudflare for `harbinge.rs` | Thaddeus | Medium |
| Encrypt Backblaze backup archives at application layer before upload | Thaddeus | Medium |
| Confirm Backblaze BAA needed for any customer processing PHI | Thaddeus | Medium |

---

## Document history

| Date | Change |
|---|---|
| 2026-06-07 | Initial assessment — Supabase, Vercel, Cloudflare, Backblaze B2 |
