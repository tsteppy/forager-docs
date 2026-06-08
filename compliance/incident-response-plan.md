---
title: "Incident Response Plan"
description: "Forager security incident response procedures — detection, triage, containment, notification, and post-mortem"
---

<Warning>
  This document is for Hiverise Labs internal use and SOC 2 audit evidence. It is not customer-facing.
</Warning>

## Purpose

This plan defines how Hiverise Labs detects, responds to, and recovers from security incidents affecting the Forager platform. It satisfies SOC 2 Type 2 Common Criteria CC7.3 (evaluate security events) and CC7.4 (respond to identified security incidents).

**Effective date:** 2026-06-07
**Owner:** Thaddeus Stepanovich, Hiverise Labs
**Review cadence:** Annually, and after any activated incident

---

## Scope

This plan covers incidents affecting:
- The Forager web dashboard (`dashboard.hiveriselabs.com`)
- The Forager Android application
- The Forager Supabase project (database, auth, storage, edge functions)
- The Forager Vercel deployment
- Any subprocessor (Supabase, Vercel, Cloudflare, Backblaze B2) incident that affects customer data

It does not cover routine operational issues (slow page loads, failed deployments) unless those events reveal or result from a security failure.

---

## Roles and responsibilities

| Role | Responsibility |
|---|---|
| **Incident Commander** | Thaddeus Stepanovich — declares incident severity, coordinates response, makes containment decisions, owns external notification |
| **Technical Responder** | Hiverise Labs engineering staff — investigates root cause, implements containment and remediation |
| **Customer Contact** | Thaddeus Stepanovich — drafts and sends customer notifications |

At current company size, the Incident Commander and Technical Responder are the same person. As the team grows, these roles should be separated.

---

## Severity classification

| Severity | Definition | Examples | Response SLA |
|---|---|---|---|
| **P1 — Critical** | Confirmed or suspected unauthorized access to customer data, credential compromise, or active exploit | Database breach, stolen service role key, admin account takeover, data exfiltration | Respond within 1 hour, contain within 4 hours |
| **P2 — High** | Security control failure with potential for data exposure, but no confirmed access | MFA bypass discovered, RLS policy gap found, webhook secret exposed in logs | Respond within 4 hours, contain within 24 hours |
| **P3 — Medium** | Vulnerability identified with no active exploitation, or suspicious activity with inconclusive evidence | Anomalous audit log pattern, failed login spike, dependency CVE with high CVSS | Respond within 24 hours, remediate within 7 days |
| **P4 — Low** | Minor security finding requiring remediation but no immediate risk | Expired certificate, outdated dependency with low CVSS, configuration drift | Remediate within 30 days |

---

## Detection sources

Incidents may be detected through any of the following:

- **Supabase Auth logs** — failed login spikes, unusual geographic access, MFA bypass attempts
- **Vercel deployment logs** — unusual request patterns, error rate spikes
- **Forager audit log** — unexpected admin actions, role escalations, bulk data operations
- **Customer report** — customer admin reports suspicious activity in their account
- **Subprocessor notification** — Supabase, Vercel, Cloudflare, or Backblaze reports a breach or security event affecting Forager
- **Dependency vulnerability alert** — GitHub Dependabot, npm audit, or security advisory for a package in use
- **Routine review** — anomaly found during monthly or quarterly audit log review

---

## Response procedures

### Phase 1 — Identify and declare

1. **Confirm the event is security-related.** Distinguish a security incident from an operational failure (downtime, bug). An incident involves unauthorized access, data exposure, credential compromise, or a deliberate attack.

2. **Classify severity** using the table above. When in doubt, classify higher and downgrade after investigation.

3. **Declare the incident.** Open a private incident record (a dated document or note) capturing:
   - Date and time detected
   - How it was detected
   - Initial severity classification
   - Known scope (which systems, which customers, what data)
   - First responder

4. **Preserve evidence.** Before making any changes, capture logs, screenshots, and timestamps. Do not delete or modify anything that could be relevant to forensics.

### Phase 2 — Contain

The goal of containment is to stop the bleeding without destroying evidence.

**Immediate containment actions by incident type:**

| Incident type | Containment action |
|---|---|
| Compromised admin account | Supabase Auth dashboard → disable user → invalidate all sessions → rotate MFA factor |
| Compromised service role key | Supabase project settings → rotate service role key → update Vercel environment variable → redeploy |
| Compromised Supabase anon key | Rotate anon key → update `NEXT_PUBLIC_SUPABASE_ANON_KEY` in Vercel → redeploy |
| Suspected database breach | Supabase → pause project (takes DB offline) → assess → restore to new project if needed |
| Active exploit of a web endpoint | Vercel → roll back deployment to last known good → investigate root cause before re-deploying fix |
| Exposed webhook secret | Identify affected webhook via audit log → contact customer admin → rotate the secret via the dashboard |
| Malicious insider (admin account) | Remove or suspend the account → review audit log for all actions taken → notify affected customers |

**Do not:**
- Delete logs or audit entries to hide the incident
- Apply unreviewed "fixes" to a production system under pressure — rushed patches often introduce new vulnerabilities
- Communicate details publicly or to customers before scope is understood

### Phase 3 — Eradicate

Remove the root cause of the incident:
- Patch the vulnerability or misconfiguration
- Rotate any credentials that were or may have been exposed
- Remove attacker-placed artifacts (unauthorized accounts, modified data)
- Verify RLS policies and access controls are intact

### Phase 4 — Recover

Restore normal operations:
- Verify the fix is effective before re-enabling access
- Monitor closely for 24–48 hours post-recovery for signs of reinfection
- Confirm audit logging is functioning normally
- Update the incident record with recovery timestamp

### Phase 5 — Post-mortem

Every P1 and P2 incident requires a written post-mortem within 5 business days of resolution. P3 incidents require a post-mortem at the discretion of the Incident Commander.

**Post-mortem format:**

```
Incident: [title]
Date: [when it occurred]
Severity: [P1/P2/P3]
Duration: [detection to resolution]

Summary
[2-3 sentences describing what happened and the impact]

Timeline
[Chronological list of key events with timestamps]

Root cause
[What underlying condition made this incident possible]

Contributing factors
[What made it worse, harder to detect, or slower to resolve]

Customer impact
[Was customer data affected? Which customers? What data?]

Remediation
[What was done to fix it]

Corrective actions
[What will be done to prevent recurrence — each with owner and due date]
```

Post-mortems are stored in `compliance/post-mortems/` (dated files).

---

## Notification requirements

### Internal notification

The Incident Commander is notified immediately upon incident declaration. No formal escalation chain exists at current company size — all incidents are owned by Thaddeus Stepanovich.

### Customer notification

**When required:** Any P1 or P2 incident that involved or may have involved unauthorized access to that customer's data.

**Timing:** Initial notification within 72 hours of confirmed incident scope. Follow-up with full details within 5 business days of resolution.

**Channel:** Email to the customer's primary admin contact from `support@harbinge.rs`.

**Template — Initial notification:**

> Subject: Forager Security Incident Notice — [date]
>
> We are writing to inform you of a security incident affecting the Forager platform that may have impacted your account.
>
> **What happened:** [Brief, factual description — no speculation]
>
> **What data was potentially involved:** [Specific to their account]
>
> **What we have done:** [Containment and remediation steps taken]
>
> **What you should do:** [Any actions required from the customer]
>
> We are continuing to investigate and will provide a full update within [X] business days. If you have questions, reply to this email or contact us at support@harbinge.rs.
>
> — Thaddeus Stepanovich, Hiverise Labs

**Template — Full resolution notice:**

> Subject: Forager Security Incident — Resolution Summary
>
> We are following up on our notice of [original date] regarding the security incident affecting the Forager platform.
>
> **What happened:** [Complete factual description]
>
> **Root cause:** [What caused the incident]
>
> **Your data:** [Confirmed scope — what was and was not affected]
>
> **What we did:** [Full remediation actions]
>
> **What we are doing to prevent recurrence:** [Corrective actions]
>
> We apologize for any disruption. Please contact us at support@harbinge.rs if you have further questions.
>
> — Thaddeus Stepanovich, Hiverise Labs

### Regulatory notification

If a breach involves personal data of EU residents, GDPR Article 33 requires notification to the relevant supervisory authority within 72 hours of becoming aware of the breach. If the breach is likely to result in high risk to individuals, Article 34 requires notification to those individuals.

At current customer profile (US healthcare IT), GDPR exposure is limited. However, if a customer has EU employees whose data is in Forager (email, display name), GDPR may apply. Consult legal counsel before making or waiving regulatory notifications.

HIPAA breach notification rules apply if Forager processes Protected Health Information (PHI). Asset tracking data (device location, maintenance schedules) is generally not PHI — but if a customer's use case connects asset data to patient records, a BAA and HIPAA breach procedures apply. Assess per customer.

---

## Tabletop exercise

This plan must be exercised at least annually via a tabletop exercise. The exercise walks through a hypothetical incident scenario to verify that responders know their roles and that the plan is current.

**Suggested scenario for first tabletop:** A customer admin reports that their webhook secret appears in their ITSM system logs in plaintext — possible that a third party captured it. Walk through: severity classification, containment, determining scope, customer notification, post-mortem.

Document each tabletop exercise in `compliance/post-mortems/` with date, scenario, participants, and any gaps identified.

---

## Document history

| Date | Change |
|---|---|
| 2026-06-07 | Initial version |
