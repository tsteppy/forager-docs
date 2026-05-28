---
title: "Notice Management"
description: "How to post, target, and deactivate in-app notices for Forager users"
---

<Warning>
  This page is for **Hiverise Labs staff only**. Customers do not have access to the platform admin dashboard.
</Warning>

## Overview

In-app notices let you push a banner message directly to Forager users without requiring an app update or email. Use notices for scheduled maintenance windows, new version announcements, billing reminders, or any urgent service communication.

Notices appear on the HomeScreen of the Android app, below the location pill. Users can dismiss a notice, but a new notice (new ID) always appears fresh regardless of prior dismissals.

---

## Accessing the Notices section

1. Go to `/admin` on the dashboard.
2. Scroll down below the company list.
3. The **Notices** section shows all current and past notices, followed by the create form.

---

## Severity levels

Each notice has a severity that controls the accent color and icon shown in the app:

| Severity | App color | When to use |
|---|---|---|
| **Info** | Blue | General announcements — new versions, feature releases, routine maintenance that has no user impact |
| **Warning** | Amber | Upcoming disruptions, billing reminders, actions the user should take soon |
| **Error** | Red | Active outages, data sync failures, or anything requiring immediate user attention |

Choose the lowest severity appropriate for the message. Over-using Warning or Error will desensitize users to them.

---

## Posting a notice

In the **Post New Notice** form at the bottom of the Notices section:

<Steps>
  <Step title="Write the message">
    Enter the text users will see in the banner. Keep it under two sentences — the app truncates long messages in the admin list view, but the full text is shown to users. Be specific: "Scheduled maintenance Saturday May 31, 10 PM – 2 AM PT. Sync will be unavailable." is better than "We are performing maintenance."
  </Step>
  <Step title="Select severity">
    Choose Info, Warning, or Error based on the table above.
  </Step>
  <Step title="Select recipients">
    - **All Companies** — the notice appears for every authenticated user. Use for platform-wide messages.
    - **Specific company** — the notice appears only for users in that company. Use for account-specific billing notices or targeted communications. A company-specific notice takes priority over a global notice if both are active simultaneously — only the company notice is shown.
  </Step>
  <Step title="Set the active window">
    - **Active From** — defaults to now. Set a future time to schedule a notice in advance (it will not appear until that time).
    - **Active Until** — leave blank for no expiry. Set an end time to automatically retire the notice after the event has passed (e.g., set Active Until to the end of a maintenance window).
  </Step>
  <Step title="Post">
    Click **Post Notice**. The notice is live immediately for the selected recipients. Users will see it the next time they open the app or return to it from the background.
  </Step>
</Steps>

---

## Managing active notices

The notice list at the top of the Notices section shows all notices ordered by creation date, newest first.

| Column | What it shows |
|---|---|
| **Severity** | Color-coded badge (blue / amber / red) |
| **Message** | Truncated to one line — the full text is what users see |
| **Recipients** | Company name or "All Companies" |
| **Active Window** | Start date → end date, or "No expiry" |
| **Status** | Active, Upcoming, or Expired |

### Status meanings

| Status | Meaning |
|---|---|
| **Active** | Notice is live — users can see it right now |
| **Upcoming** | Active From is in the future — not yet visible to users |
| **Expired** | Active Until has passed — no longer visible to users |

### Deactivating a notice

Click **Deactivate** on any Active notice to end it immediately. This sets the notice's expiry to the current time. Users will no longer see it on their next app open or resume. Deactivated notices stay in the list as Expired for reference.

There is no way to edit a posted notice — if you need to change the message or targeting, deactivate the current notice and post a new one.

---

## Common scenarios

### Scheduled maintenance window
Post an Info notice 24 hours before the window. Set Active Until to the end of the maintenance period so it retires automatically. Example message: *"Scheduled maintenance Saturday May 31, 10 PM – 2 AM PT. Data sync will resume automatically afterward."*

### Billing reminder (single customer)
Post a Warning notice targeted to the specific company. Leave Active Until blank. Deactivate it manually once the invoice is paid or you escalate to Grace Period status.

### Active outage
Post an Error notice to All Companies immediately. Deactivate it as soon as the issue is resolved. Follow up with an Info notice summarizing what happened if the outage lasted more than 30 minutes.

### New app version available
Post an Info notice to All Companies with a link or instructions for updating. Set Active Until to 2–4 weeks out so it retires once most users have updated.
