---
title: "Change Management Policy"
description: "Forager software change management controls — branch policy, review requirements, deployment gates, and emergency procedures (SOC 2 CC8)"
---

<Warning>
  This document is for Hiverise Labs internal use and SOC 2 audit evidence. It is not customer-facing.
</Warning>

## Purpose

This policy defines how changes to the Forager platform are reviewed, approved, and deployed. It satisfies SOC 2 Type 2 Common Criteria CC8.1 (manage changes to infrastructure, data, software, and procedures).

**Effective date:** 2026-06-07
**Owner:** Thaddeus Stepanovich, Hiverise Labs
**Review cadence:** Annually, or when the team size or deployment process changes materially

---

## Scope

This policy applies to all changes to:
- The Forager Next.js web dashboard (`forager-web/`)
- The Forager Supabase project (database migrations, edge functions, RLS policies)
- The Forager Android application (`forager/`)
- Infrastructure configuration (Vercel project settings, Supabase project settings, environment variables)
- Compliance documentation in `forager-docs/`

---

## Branch and merge policy

### Feature branches

All changes must be developed on a named feature branch and submitted as a pull request (PR) against the `main` branch. Direct pushes to `main` are prohibited except in the emergency hotfix procedure defined below.

**Branch naming convention:**

| Type | Pattern | Example |
|---|---|---|
| Feature | `feat/<short-description>` | `feat/audit-log` |
| Bug fix | `fix/<short-description>` | `fix/mfa-redirect-loop` |
| Database migration | `db/<migration-name>` | `db/add-audit-logs-table` |
| Documentation | `docs/<topic>` | `docs/change-management-policy` |
| Hotfix | `hotfix/<short-description>` | `hotfix/webhook-secret-exposure` |

### Pull request requirements

Every PR must include:
- A clear title describing what changed
- A description of why the change was made and what it affects
- Self-review: the author must review their own diff before requesting review

**Review requirements:**

| Team size | Minimum approvals |
|---|---|
| Solo founder (current) | 1 — self-approval with documented rationale in PR description |
| 2+ engineers | 1 approval from a reviewer other than the author |
| Changes to RLS policies or migrations | 1 approval + manual test in Supabase Studio before merge |
| Changes to auth flow, MFA, or session management | 1 approval + end-to-end test of the auth path |

**Solo-founder note:** Self-approval is an acknowledged limitation during the solo-founder phase. The compensating controls are: (1) all changes are committed to git history with timestamps, (2) Vercel maintains a deployment log of every production build, and (3) database migrations are version-controlled and applied manually — not auto-applied on merge. This limitation must be disclosed to SOC 2 auditors.

### Merge policy

- Squash merges are preferred for feature branches to keep `main` history readable
- Merge commits are acceptable for larger feature branches with meaningful commit history
- Force-pushes to `main` are prohibited

---

## Database migration policy

Database schema changes are the highest-risk category of change because they are difficult or impossible to roll back cleanly.

### Rules

1. All schema changes must be expressed as a numbered SQL migration file in `forager/supabase/migrations/`
2. Migration files are named `YYYYMMDDHHMMSS_<description>.sql`
3. No ad-hoc schema changes via Supabase Studio SQL editor in production — all changes must go through migration files
4. Migrations are applied manually via psql against the production database — never auto-applied by a CI/CD pipeline without human review
5. Every migration must be reviewed for:
   - RLS policy correctness (does it introduce or close a gap?)
   - Data loss risk (DROP, ALTER with type change, removing NOT NULL)
   - Reversibility (can this be undone if it causes an incident?)

### Migration application procedure

```bash
# Apply a migration to production
psql "$SUPABASE_DB_URL" -f supabase/migrations/<migration_file>.sql

# Verify the migration applied
psql "$SUPABASE_DB_URL" -c "\d <affected_table>"
```

After applying, confirm the affected table structure and run a smoke test of the impacted feature before closing the PR.

---

## Staging environment

Forager uses Vercel preview deployments as the staging environment. Every PR opened against `main` automatically generates a preview deployment at a unique Vercel URL.

**What preview deployments cover:**
- Web dashboard UI and server actions
- Next.js routing and middleware

**What preview deployments do not cover:**
- Database migrations — these must be tested in a local Supabase instance or accepted as production-first (document in PR description when this is the case)
- Edge functions — deployed separately; verify in staging Supabase project if one exists

**Database for preview deployments:** Preview deployments connect to the production Supabase project unless a separate staging Supabase project is configured. Until a staging database is established, database-touching changes must be reviewed with extra care and tested via smoke test after merge.

---

## Deployment policy

### Web dashboard (Vercel)

Merging to `main` automatically triggers a Vercel production deployment. No manual deploy step is required. Vercel maintains a deployment history; any deployment can be rolled back via the Vercel dashboard.

**Post-deployment verification:**
1. Open the production dashboard and log in
2. Confirm the changed feature works as expected
3. Spot-check adjacent features for regressions (auth flow, team management, webhook config)

### Android application

APK builds are produced manually and distributed via the `apk-releases` Supabase Storage bucket. There is no automated Android CI/CD pipeline at this time. Releases are versioned and the build process is documented in the Android project.

### Database migrations

Applied manually per the procedure above. Migrations are not automatically applied on Vercel deploy or git merge.

---

## Emergency hotfix procedure

For P1 or P2 incidents (as defined in the [Incident Response Plan](/compliance/incident-response-plan)) requiring an immediate fix:

1. Create a `hotfix/<description>` branch from `main`
2. Implement the minimal fix — no unrelated changes
3. If a PR review is not feasible within the incident response SLA, document the reason in the PR description and merge with self-approval
4. Tag the commit as a hotfix: `git tag hotfix/<date>-<description>`
5. A post-hotfix review must be completed within 24 hours of the fix going live — open a follow-up PR or conduct a code review in the incident post-mortem

---

## Environment variable changes

Changes to Vercel environment variables (e.g., rotating the Supabase service role key) do not go through a PR. However, they must be:
- Documented in the [Credential Rotation Policy](/compliance/credential-rotation-policy) rotation log
- Applied by an authorized team member only (currently: Thaddeus Stepanovich)
- Followed by a production redeployment to pick up the new values

---

## Document history

| Date | Change |
|---|---|
| 2026-06-07 | Initial version |
