# Project Manager

A single-file, browser-only construction/renovation project tracker. Twelve
tabs cover a project end to end — Overview, Scope Summary, Detailed SOW,
Assumptions & Exclusions, Open Items, Budget, Permits & Compliance, Project
Sequence, Next Steps/Tasks, Special Orders, Change Orders, and Photos/Files.

## What it does

- Tracks any number of projects, each with its own set of tabs, entirely in
  your browser — no account, no server.
- Optionally syncs structured tables (Budget, Permits, Detailed SOW, Open
  Items, Change Orders, Subcontractors, and any custom Special Order
  category) to a **Google Sheet** created per project.
- Optionally exports Overview, Scope Summary, and Assumptions & Exclusions
  as one-way **Google Docx snapshots**.
- Optionally organizes **Google Drive** subfolders (Photos, Floor Plans,
  Blueprints) per project and links out to them.
- Next Steps/Tasks stays strictly local and is never synced anywhere.

See [`DESIGN.md`](./DESIGN.md) for the reasoning behind these choices,
including exactly what "sync" does and doesn't mean here.

## Storage & privacy model

Project Manager is a single self-contained HTML file with no server behind
it. Everything — every project's data, your Google OAuth Client ID and API
key — is written to this browser's local storage, on this device, and stays
there. Nothing is sent anywhere except the direct calls this app makes to
Google's own APIs (Sheets, Docs, Drive) once you've connected your own
Google Cloud project and explicitly click a sync action.

**Sync is one-way and on-demand**, not a live link: Sheets get pushed to on
each edit but never read back; Docx snapshots only update when you click
"Generate Docx"; nothing auto-syncs on a timer. Editing a Sheet directly in
Google will be overwritten by the app on your next edit there.

There's no export or backup feature. The **Clear All Data** button in the
sidebar permanently wipes everything in this browser; there's no way to get
it back afterward. Data already synced to Google Sheets/Docs/Drive is
untouched by that button — those live in your own Google account
independently of the app.

Your Google connection doesn't persist across a page reload — you'll click
"Connect / Setup" once per browser session before using any sync feature.

## Usage

Open the deployed page directly, or download `index.html` and open it from
disk — no install or build step needed either way.

To use the Google Sheets/Docs/Drive features, you'll need your own Google
Cloud project (free tier) with the Sheets and Drive APIs enabled, and an
OAuth Client ID + API key from it. Enter those once via **Connect / Setup**
in the sidebar.

## Deployment

This repo is set up for GitHub Pages, serving `index.html` directly from
`main`. Pushing to `main` deploys the update.

## Installing on your phone

`manifest.json`, `sw.js`, and `icons/` make this app installable as a PWA
once hosted over `https://` (e.g. GitHub Pages) — a real app icon, name, and
a minimal offline shell cache instead of just a bookmark.

1. Make this repo public (Settings → Danger Zone → Change visibility), then
   enable GitHub Pages (Settings → Pages → Deploy from branch → `main` → `/`
   root).
2. Once it's live, open `https://<your-username>.github.io/ProjectHub/` on
   your phone.
3. Use your browser's "Add to Home Screen" / "Install app" option.
