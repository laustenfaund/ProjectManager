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

## Modularity: where it lives and where it deliberately doesn't

The 12 core tabs (Overview through Photos/Files) are fixed and are staying
that way. They aren't "a kind of thing you might have several of" — every
project has exactly one Overview, one task list, one Budget. Making them
pluggable would add configuration decisions with no corresponding
flexibility gained, and would cost the app its main usability advantage:
a new project starts pre-built and ready to type into, not assembled from
parts first.

Special Orders is the one place modularity is real, because it's the one
category that's inherently open-ended — you don't know in advance how many
trades/material categories a given project needs, and they're all the same
*kind* of thing. That's where the module system lives:

- Each category gets its **own configurable field list**, instead of every
  category being forced onto the same fixed 9 columns.
- Categories can be **deleted** (removes the category and its local data;
  does not touch anything already synced to Google).
- **Not built yet:** subgroups/nesting (e.g. a "Materials" parent holding
  "Cabinets"/"Flooring" as children) and alternate module types (local
  checklist, freeform notes) beyond the existing Sheets-synced table. Both
  are real future possibilities, not ruled out — just not needed until a
  flat list of categories actually gets unwieldy in practice.

**Constraint for whenever subgroups do get built:** any emergent
group/subgroup structure needs to show up in how the Drive artifacts are
organized too, not just in the app's own UI. Google Sheets has no
folder-of-tabs concept, so the honest way to do this *within* a
spreadsheet is tab naming + tab color (e.g. "Materials — Cabinets" /
"Materials — Flooring", adjacent, same color) rather than pretending Sheets
tabs can be literally nested. Whether a subgroup ever warrants becoming its
*own* separate Drive artifact (rather than just tabs within the one
project spreadsheet) is a real design fork, not a default — ask before
building that, don't assume it.

## Explicit non-goals (for now)

- Two-way Sheets sync, or any read-back from Google.
- A loading/progress indicator during Drive/Sheets operations.
- Data export/backup beyond the existing one-way Google sync.
- Multi-user / multi-device support of any kind.
- An LLM/AI feature of any kind (deliberately parked — the mechanism
  (bring-your-own API key, same pattern as two other apps) is understood,
  but no concrete in-app use case has been chosen yet).

None of these are ruled out permanently — they're just out of scope for the
current single-user, single-device, bring-your-own-Google-project design.
