---
title: "Provisioning a New Company"
description: "How Hiverise Labs staff create a new customer company and invite its first admin"
---

<Warning>
  This page is for **Hiverise Labs staff only**. Customers do not have access to the platform admin dashboard.
</Warning>

## Overview

When a new customer is signed up, a Hiverise Labs admin creates their company account and invites their first admin user — all from a single form in the platform admin dashboard. The customer's admin receives an email invite, clicks the link to verify their account, and lands on a welcome page with a link to download the Forager app.

## Accessing the admin dashboard

Log in at [dashboard.hiveriselabs.com](https://dashboard.hiveriselabs.com) with your Hiverise Labs admin account. Navigate to `/admin`. The platform admin page shows a list of all provisioned companies and the **Add Company** form.

<Note>
  Only the account registered as `SUPER_ADMIN_EMAIL` in the Forager environment can access `/admin`. Attempting to reach this page with any other account redirects to the standard dashboard.
</Note>

## Provisioning a new company

<Steps>
  <Step title="Open the Add Company form">
    Click **Add Company** at the top of the `/admin` page. The form expands inline.
  </Step>
  <Step title="Fill in the company details">
    | Field | Required | Notes |
    |---|---|---|
    | **Company name** | Yes | The customer's organization name as it should appear in the system |
    | **Admin email** | Yes | The email address of the first admin at the customer site |
    | **Seat limit** | Yes | Maximum number of team members (default: 6). Can be discussed with the customer during sales. |
    | **Notes** | No | Internal notes — contract details, deployment site, primary contact. Not visible to the customer. |
  </Step>
  <Step title="Submit">
    Click **Create Company & Send Invite**. The form:
    1. Creates the company row in the database
    2. Sends an invite email to the admin email address with `role: admin` embedded in the invite metadata
    3. Returns a success banner confirming the company name and email address the invite was sent to
  </Step>
</Steps>

<Note>
  If the invite email fails (e.g., the address already has a Supabase auth account with a conflicting state), the company row is automatically deleted so you don't end up with an orphaned record. The error message from Supabase is shown in the form — resolve it and retry.
</Note>

## What the customer's admin receives

The admin receives an email with a subject like **"You have been invited"**. When they click the link:

1. Their Forager account is created and verified automatically — no password required at this step.
2. Their profile is seeded with `role: admin` and their company association.
3. They are taken to a welcome page at `dashboard.hiveriselabs.com/auth/accept-invite` with a **Get it on Google Play** button.
4. They download the Forager app and sign in with the same email address.

Once they are in the app, they have full admin access: they can invite team members from the **Team** page in the dashboard and manage all company settings.

## Seat limits

The seat limit controls how many team members the customer can invite. It is enforced at invite time — the `invite-user` function checks the current member count before sending. If the company is at capacity, the invite is rejected with a clear error message.

The default is **6 seats**. Adjust this based on the customer's contract. You cannot edit a seat limit after provisioning from the dashboard — update it directly in the database if needed.

## Your own admin profile

When you visit `/admin`, the dashboard automatically ensures your Hiverise Labs account has `role: admin` in the Forager app. If your account is not yet associated with a company, it creates a **Hiverise Labs** internal company and links you to it. This lets you log in to the Forager Android app with full admin access for testing and support.

This runs on every `/admin` page load and is idempotent — it does nothing if your profile is already correct.
