---
title: Inventory Placement Mode
description: How to set up and run a wall-to-wall asset inventory using floor plan placement
---

## What It Does

Inventory Placement Mode lets you scan asset barcodes and tap their location on a floor plan — without requiring named zones or an RF survey first. Use it for wall-to-wall inventory walks where you want to map where assets are before you have zone names defined.

Assets placed this way are visible on the web dashboard under **Floor Plan → Placement Inventory**.

## Prerequisites

Before contractors start, an admin must:
1. Create a basic location hierarchy: **Sites → Buildings → Floors** (no zones needed)
2. Upload a floor plan image for each floor via the web dashboard (**Floor Plan → Zone Map**, upload button on any floor node)

## Running the Inventory

Contractors (or leads) open **Place Assets** from the Forager home screen:
1. Select a floor
2. Tap **Scan & Place**, scan an asset barcode (or use your hardware trigger)
3. Tap the floor plan where the asset is located
4. Repeat until the floor is complete

Assets placed by other techs appear on the map within 15 seconds.

If a scanned tag is not in the asset roster, Forager creates a minimal asset record automatically. Enrich it later via the Assets page or CSV import.

## Naming Pass (Web Dashboard)

After inventory is complete, an admin assigns zone names:

1. Open **Floor Plan → Placement Inventory** on the web dashboard
2. Select the floor
3. Click asset pins (gray = unassigned) to select them
4. Click **Create Zone** to name and create a zone for the selected assets
5. Or use **Assign to zone** to add to an existing zone

Assigned assets turn green. Once assigned, assets become visible in Device Hunt navigation and presence confirmation.

## Naming Pass (Android)

Admins and leads can also do the naming pass from the Android app:

1. Open **Place Assets** and select a floor
2. Tap **Name Zones** (top right)
3. Tap a cluster of gray pins to select them
4. Tap **Create Zone**, enter a name, and confirm

## Roles

| Action | Admin | Lead | Field Tech |
|---|---|---|---|
| Place assets | ✓ | ✓ | ✗ |
| Create zones (web) | ✓ | ✗ | ✗ |
| Create zones (Android) | ✓ | ✓ | ✗ |
| Assign to existing zone (web) | ✓ | ✓ | ✗ |
