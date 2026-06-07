---
title: "Remediation Sprints"
description: "How to create targeted work queues for incident response, firmware recalls, and pre-audit sweeps"
---

## What is a Remediation Sprint?

A **Remediation Sprint** is a targeted work queue that drives field techs to physically locate and confirm a specific list of assets. Unlike the gamification sprint (which rewards techs for confirming any asset and tracks points on a leaderboard), a Remediation Sprint is incident-driven: you define the exact assets that need attention, assign them to specific techs, and track resolution to completion.

Every resolved item requires a barcode scan — there is no way to mark an item resolved without physically finding and scanning the device. This gives you a verified, attestation-backed record that each asset was physically located by a named tech at a specific time.

<Note>
  Remediation Sprints and gamification Sprints are separate features. A Remediation Sprint does not affect leaderboard scores or sprint points.
</Note>

---

## When to use it

| Scenario | Why it fits |
|---|---|
| **Firmware recall or security patch** | You have a list of affected serial numbers or asset tags — assign each device to the tech covering that area and track resolution |
| **Misconfiguration sweep** | A config error was pushed to a batch of devices; confirm each one is corrected before closing the incident ticket |
| **Pre-audit PMI sweep** | Joint Commission or other auditor arriving — load the assets flagged in your last CMMS export and verify they are all findable and accounted for |
| **Onboarding unfamiliar contractors** | Give a new tech a bounded list of devices in their assigned area without exposing the full asset registry |

---

## Creating a sprint

You can create a sprint manually by entering asset tags, or automatically from the PMI Compliance report.

### Auto-create from PMI Compliance

If you have assets with an overdue PMI schedule, go to **Dashboard → Reports → PMI Compliance** and click **Create Sprint from Overdue**. Forager:

1. Pulls every asset currently in **Overdue** status
2. Creates a sprint named `PMI Overdue — YYYY-MM-DD`
3. Pre-populates it with all overdue assets (unassigned)
4. Redirects you to the sprint detail page

Skip the manual steps below and go straight to [Assigning items to techs](#assigning-items-to-techs). The asset list and sprint name are already set.

### Manual creation

From the web dashboard:

1. Go to **Dashboard → Sprints**.
2. Click **Create Sprint**.
3. Fill in the form:

| Field | Required | Notes |
|---|---|---|
| **Name** | Yes | Something that identifies the incident — e.g. "Pump Firmware Recall June 2026" |
| **Description** | No | Ticket number, scope, or any context the team needs |
| **Asset tags** | Yes | Paste a newline-separated or comma-separated list of asset tags |

4. Click **Create**.

Forager attempts to match each tag against your asset registry. After creation, the sprint detail page shows:

- **Matched items** — assets found in your registry, ready to be assigned
- **Unmatched tags** — tags that did not match any known asset; these appear as unresolvable rows so you can identify data entry errors or decommissioned devices

<Tip>
  Copy the asset tag column directly from your CMMS or CMDB export and paste it into the field — Forager handles leading/trailing whitespace and skips blank lines.
</Tip>

---

## Assigning items to techs

Open the sprint detail page and locate the **Items** table. Each row represents one asset.

1. Find the asset row you want to assign.
2. Click the **Assigned To** dropdown for that row.
3. Select a team member.

The assignment is saved immediately — no Save button needed. The tech will see the item in their queue the next time they open the app.

<Tip>
  **Bulk assignment strategy:** sort the Items table by location (building → floor → room), then assign consecutive rows to the tech who covers that area. This minimizes travel and gets the sprint done faster.
</Tip>

---

## Tech workflow on the app

Field techs access their remediation queue from the home screen.

<Steps>
  <Step title="Open the sprint card">
    The home screen shows an active Remediation Sprint card with the sprint name and a count of assigned items. Tap **Open Queue**.
  </Step>
  <Step title="Select My Items">
    The queue screen has two tabs: **All Items** (full sprint list) and **My Items** (items assigned to this tech). Switch to **My Items**.
  </Step>
  <Step title="Start the hunt">
    Tap the **Hunt** button on any unresolved item. This launches the Device Hunt flow with the floor plan navigation view centered on the asset's last known location.
  </Step>
  <Step title="Navigate to the asset">
    Use the floor plan to navigate to the room. The asset's position is pinned on the map based on its last confirmed location.
  </Step>
  <Step title="Scan the barcode">
    When you locate the physical device, scan its barcode using the camera. Forager matches the scan against the sprint item.
  </Step>
  <Step title="Mark Resolved">
    A **Sprint Item Found** bottom sheet appears confirming the match. Tap **Mark Resolved**. The item is closed, an attestation record is written, and your assigned item count decrements.
  </Step>
</Steps>

<Note>
  A tech cannot mark a sprint item resolved without completing a barcode scan. The Mark Resolved button is only available after a successful scan match. This ensures every resolved item has a physical attestation on record.
</Note>

---

## Monitoring progress

The sprint detail page gives you a live view of the sprint as techs work through their queues.

- **Progress bar** — shows the percentage of items resolved out of total items
- **By Tech table** — one row per assigned tech; columns show assigned count, resolved count, and pending count
- **Items table** — full item list with status filter (All / Pending / Resolved / Unmatched); clicking a row shows the full attestation record for resolved items

You do not need to refresh — the page updates in real time as techs scan and resolve items.

---

## Closing a sprint

When all items are resolved (or you have resolved everything that can be resolved):

1. Open the sprint detail page.
2. Click **Close Sprint**.

The sprint status changes to **Completed**. The sprint card disappears from the home screen for all techs. All data — item list, assignments, attestation records — is retained indefinitely and remains accessible from the Sprints list.

<Note>
  Closing a sprint is permanent. A completed sprint cannot be re-opened. If you close prematurely, create a new sprint with the remaining unresolved tags.
</Note>

---

## Resolution requires a scan

This cannot be bypassed. Every resolved sprint item has:

- The asset tag that was scanned
- The tech who scanned it
- The timestamp of the scan
- The attestation ID linking back to the full scan record

This attestation trail is what makes Remediation Sprints audit-ready. When an auditor asks "how do you know this device was physically located on June 3rd?", the answer is a timestamped barcode scan by a named tech — not a checkbox.
