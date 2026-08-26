# Project Manager — design notes

This document exists so the shape of the app doesn't have to be re-derived
from memory later, and so future changes can be checked against the
reasoning that produced them, not just the resulting feature list.

## What this is

A single-file, static, browser-only construction/renovation project tracker,
deployed on GitHub Pages. No backend, no build step, no account system.
Twelve tabs cover a project's lifecycle: Overview, Scope Summary, Detailed
SOW, Assumptions & Exclusions, Open Items, Budget, Permits & Compliance,
Project Sequence, Next Steps/Tasks, Special Orders (dynamic per-category
tabs), Change Orders, and Photos/Files.

## Storage model — local-first, Google as a one-way mirror

**The browser's `localStorage` is the only source of truth.** Every tab
reads from and writes to `localStorage` first; the app never reads anything
back from Google. Google Sheets, Google Docs, and Google Drive are treated
as **one-way exports**, not live storage:

- **Sheets-backed tabs** (Detailed SOW, Open Items, Budget, Permits, Project
  Sequence, Change Orders, Subcontractors, and any Special Order category)
  push each edit to a matching Sheet tab when connected, but never pull.
  Editing the Sheet directly is invisible to the app and will be silently
  overwritten by the next edit made in-app.
- **Docx-backed tabs** (Overview, Scope Summary, Assumptions & Exclusions)
  only sync on an explicit "Generate Docx" click — a snapshot, not a live
  link. Regenerating overwrites the previous snapshot; it never reads
  anything back.
- **Next Steps/Tasks** is local-only by design and never touches Google at
  all, on any tab or in any code path.
- **Photos/Files** holds no file data in the app itself — it only links out
  to three Drive subfolders (Photos, Floor Plans, Blueprints) that live
  under the project's own Drive folder.

**Why one-way instead of two-way sync:** two-way sync means conflict
resolution, and conflict resolution means either a backend to arbitrate or a
lot of client-side complexity for a single-user tool that doesn't need it.
The tradeoff was made explicitly: keep the app simple and offline-capable at
the cost of Google never being an authoritative copy. See "Known risks"
below for what that costs in practice.

**Why `localStorage` and not IndexedDB:** project records here are small
(text fields and short tables, not bulk file data), well within
`localStorage`'s per-origin quota. IndexedDB would add complexity with no
benefit at this data volume. Revisit this if Photos/Files ever starts
storing file bytes instead of just Drive links.

## Google integration

Bring-your-own Google Cloud project: the user supplies an OAuth Client ID
and API key (Google Setup modal), then signs in each session via Google
Identity Services' token client. This is a deliberate architectural
consequence, not an oversight: **the Google connection never persists
across a page reload.** Access tokens from the implicit OAuth flow are
short-lived (~1 hour) and aren't meant to be stored client-side long-term —
there's no backend here to hold a refresh token safely. Every session,
before using any sync feature, the user has to click "Connect / Setup" once.

All Sheets/Docs/Drive-folder creation funnels through one shared
find-or-create helper (`findOrCreateFolder`). Every sync action resolves the
same "Ongoing Projects" folder at the Drive root, then the same
"[Project Name]" subfolder beneath it, and creates whatever it needs inside
that. This makes the three sync paths (Sheet, Docx, Photo folders)
independent of call order — any one of them can run first and the others
will still land in the same place.

## Known risks (accepted, not hidden)

- **`localStorage` is the only copy of anything not explicitly synced.**
  Clearing site data, switching browsers/devices, or losing the browser
  profile loses everything with no recovery. There is no export/backup
  feature. Mitigation is procedural, not technical: sync Sheets/Docx
  regularly as a de facto backup.
- **Folder and Sheet-tab lookups are name-based, not ID-based.** Two
  projects with the same name will resolve to the same Drive folder and
  Sheet and silently merge their cloud data. Renaming a project after its
  Drive resources exist creates a *new* folder rather than moving the old
  one — the old folder is orphaned, not renamed.
- **Manually renaming a tab inside Google Sheets breaks recognition.** The
  app matches tabs by exact expected title, not by ID; a renamed tab looks
  "missing" and gets duplicated on the next sync.
- **No cross-tab awareness.** Two browser tabs open on the same project can
  silently overwrite each other's edits — `localStorage` writes are
  last-write-wins with no coordination.
- **Storage key names aren't namespaced beyond `sl_` prefixes**
  (`sl_projects`, `sl_active_project`, `sl_google_config`). `localStorage`
  is scoped per-domain, not per-path — another GitHub Pages project under
  the same account using the same key names would collide. Keep other
  apps' keys distinct if this ever comes up.

## Explicit non-goals (for now)

- Two-way Sheets sync, or any read-back from Google.
- Category deletion (Special Orders categories can be added but not
  removed via the UI).
- A loading/progress indicator during Drive/Sheets operations.
- Data export/backup beyond the existing one-way Google sync.
- Multi-user / multi-device support of any kind.

None of these are ruled out permanently — they're just out of scope for the
current single-user, single-device, bring-your-own-Google-project design.
