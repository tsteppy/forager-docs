---
title: "Credential Rotation Policy"
description: "Rotation schedule and procedures for all Forager platform credentials — Supabase keys, Vercel secrets, and customer webhook secrets"
---

<Warning>
  This document is for Hiverise Labs internal use and SOC 2 audit evidence. It is not customer-facing.
</Warning>

## Purpose

This policy defines the rotation schedule and procedures for all credentials used by the Forager platform. It satisfies SOC 2 Type 2 Common Criteria CC6.1 (implement logical access security) and CC6.7 (manage credentials).

**Effective date:** 2026-06-07
**Owner:** Thaddeus Stepanovich, Hiverise Labs
**Review cadence:** Annually, or when a credential is suspected of compromise

---

## Credential inventory

| Credential | Where stored | Who has access | Rotation schedule |
|---|---|---|---|
| Supabase service role key | Vercel env var (`SUPABASE_SERVICE_ROLE_KEY`) | Hiverise Labs engineering only | Annually |
| Supabase anon key | Vercel env var (`NEXT_PUBLIC_SUPABASE_ANON_KEY`) | Public (by design — low privilege) | Annually or on compromise |
| Supabase JWT secret | Supabase project settings | Supabase-managed | On compromise only (rotation invalidates all sessions) |
| Vercel deploy token | Vercel project (implicit) | Vercel-managed | On compromise only |
| Customer webhook secrets | Supabase Vault | Encrypted at rest, decrypted at delivery only | On customer request or suspected exposure |
| Backblaze B2 application keys | Configured on backup host (`l1ghth0use`) | Hiverise Labs only | Annually |
| Supabase dashboard credentials | Supabase account (SSO or email) | Thaddeus Stepanovich | Per Supabase account password policy |

---

## Rotation schedule

### Annual rotation (scheduled)

Perform the following rotations once per calendar year, during a low-traffic window.

| Credential | Month | Notes |
|---|---|---|
| Supabase service role key | January | Highest privilege — rotate first, verify before rotating others |
| Supabase anon key | January | Low-risk; rotate alongside service role key for operational simplicity |
| Backblaze B2 application keys | January | One key per bucket; rotate sequentially |

**Annual rotation log:** Record each rotation in the table at the bottom of this document.

### Immediate rotation (on compromise or off-boarding)

Rotate immediately when:
- A credential is suspected or confirmed to have been exposed (logs, incident, employee off-boarding)
- An authorized team member with access to the credential leaves the organization
- A breach or unauthorized access is confirmed

Treat an immediate rotation as a P1 incident response action. Follow the containment steps in the [Incident Response Plan](/compliance/incident-response-plan).

---

## Rotation procedures

### Supabase service role key

The service role key bypasses RLS and has full database access. This is the highest-privilege credential in the Forager stack.

```
1. Generate new key:
   Supabase Dashboard → Project Settings → API → Service Role Key → Reveal → Generate new

2. Update Vercel:
   Vercel Dashboard → forager-web project → Settings → Environment Variables
   → Update SUPABASE_SERVICE_ROLE_KEY to new value
   → Apply to Production, Preview, Development

3. Redeploy:
   Vercel Dashboard → Deployments → Redeploy latest production deployment
   (or push a no-op commit to trigger automatic redeploy)

4. Verify:
   - Open production dashboard and confirm login works
   - Perform an admin action (e.g., view team list) — this exercises the service role path
   - Check Vercel function logs for any auth errors

5. Revoke old key:
   Supabase Dashboard → Project Settings → API → revoke the previous key

6. Log the rotation in the table below.
```

### Supabase anon key

The anon key is low-privilege (subject to RLS). It is also publicly visible in the browser — its rotation does not need to be treated urgently unless RLS is suspected to be bypassed.

```
1. Generate new key:
   Supabase Dashboard → Project Settings → API → Anon Key → Generate new

2. Update Vercel:
   Vercel Dashboard → forager-web project → Settings → Environment Variables
   → Update NEXT_PUBLIC_SUPABASE_ANON_KEY to new value

3. Redeploy (same as service role key procedure)

4. Verify: Open the production dashboard login page — confirm auth flow works

5. Revoke old key in Supabase

6. Log the rotation.
```

### Customer webhook secrets

Customer webhook secrets are stored in Supabase Vault and are never exposed to Hiverise Labs staff in plaintext. Rotation is customer-initiated via the Forager dashboard.

**Customer-initiated rotation:**
The customer admin navigates to Settings → CMDB Webhooks → Edit → enters a new secret → saves. The old vault entry is replaced. No Hiverise Labs action is required.

**Hiverise Labs-initiated rotation (incident response):**
If a specific customer's webhook secret is suspected compromised (e.g., found in a log):
1. Identify the affected webhook via the audit log (`action = 'update_webhook'`, `meta.secret_rotated = true`)
2. Contact the customer admin at their registered admin email
3. Instruct them to rotate the secret via the dashboard immediately
4. Log the incident per the IR plan

### Backblaze B2 application keys

```
1. Log in to Backblaze B2 at backblaze.com

2. Navigate to App Keys → Create a new application key
   - Scope to the specific bucket (stepanovich-immich-upload or forager-backups)
   - Set permissions: Read and Write (or Read Only for restore-only keys)

3. Copy the new key ID and application key — this is shown only once

4. Update the backup configuration on l1ghth0use:
   SSH root@l1ghth0use
   Update the rclone or backup script config with the new key

5. Run a test backup sync to confirm the new key works

6. Delete the old application key in the Backblaze B2 dashboard

7. Log the rotation.
```

---

## Supabase JWT secret (emergency only)

**Do not rotate this on a regular schedule.** Rotating the JWT secret invalidates all active user sessions platform-wide — every logged-in user will be immediately signed out, including admins.

Rotate only if:
- The JWT secret is confirmed compromised
- A forensic investigation requires invalidating all active sessions

```
Supabase Dashboard → Project Settings → API → JWT Secret → Generate new
```

After rotating: notify all admin users that they will need to log in again. Monitor for support requests.

---

## Rotation log

Record every rotation here. This log is evidence for SOC 2 auditors that the rotation policy is followed.

| Date | Credential | Reason | Performed by | Notes |
|---|---|---|---|---|
| — | — | — | — | Policy effective 2026-06-07; first annual rotation due January 2027 |

---

## Document history

| Date | Change |
|---|---|
| 2026-06-07 | Initial version. First scheduled rotation due January 2027. |
