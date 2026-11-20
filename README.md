# Zman

**A Jewish life app built around time, not features.**

Most Jewish apps are separate tools: a calendar here, zmanim there, a siddur somewhere else, a Talmud reader in a fourth place. Zman treats them all as layers of one personal timeline, filtered through your own nusach and customs, and designed RTL-first with real Hebrew typography. It works fully offline, including on Shabbat.

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Feature Overview](#feature-overview)
3. [Tech Stack](#tech-stack)
4. [Architecture](#architecture)
5. [Content Pipeline](#content-pipeline)
6. [Data Model](#data-model)
7. [Offline and Sync Strategy](#offline-and-sync-strategy)
8. [Repository Structure](#repository-structure)
9. [Getting Started](#getting-started)
10. [MVP Roadmap](#mvp-roadmap)
11. [Licensing and Content Sources](#licensing-and-content-sources)

---

## Core Concepts

- **Layered Timeline:** one scrollable day view where Hebrew/Gregorian dates, zmanim, parsha, Daf Yomi, yahrzeits, fasts, and personal events are toggleable layers.
- **Adaptive Siddur:** opens at the right prayer for the current time, day, and nusach, and inserts the correct additions automatically (Ya'aleh VeYavo, Al HaNissim, Mashiv HaRuach, Tachanun skipping, Hallel type, Israel/diaspora logic).
- **Study that teaches:** Gemara with Rashi and Tosafot in a mobile-friendly flow, plus (post-MVP) argument maps of each sugya.
- **Personal halacha context:** you set your community's customs once, and everything is filtered through them.
- **Offline-first:** every feature works without a connection. Network is used only to download content packs and sync user data.

---

## Feature Overview

| Area | MVP | Post-MVP |
|---|---|---|
| Timeline | Hebrew date, parsha, zmanim, holidays, Daf Yomi | Yahrzeits, family calendar, custom layers |
| Siddur | Shacharit / Mincha / Maariv, nusach selection, auto-open, saved position | Word-level translation and grammar |
| Talmud | Daf reader (Gemara + Rashi + Tosafot, Hebrew/English) | Argument maps, vocab SRS, audio sync |
| Shabbat mode | Minimal UI, no notifications | Scheduled auto-enable |
| Reminders | Candle lighting, Omer | Customizable per minhag |
| Sync | Bookmarks, highlights, settings | Chavruta mode (real-time) |
| Extensibility | None | Community plugin system for shul schedules |

---

## Tech Stack

### Mobile app (`/app`)
| Concern | Choice |
|---|---|
| Framework | React Native + Expo (managed workflow, dev client) |
| Language | TypeScript (strict) |
| Navigation | Expo Router |
| State | Zustand |
| Animation / drawing | React Native Reanimated, React Native Skia (timeline rendering) |
| Local database | `expo-sqlite` with FTS5 full-text search |
| Fonts | Frank Ruhl Libre (bundled), with niqqud and cantillation support |
| i18n / RTL | `I18nManager` forced RTL for Hebrew, RTL-first layout primitives |
| Notifications | `expo-notifications` (local, scheduled on-device) |

### Calendar and zmanim (on-device)
- `@hebcal/core` for Hebrew dates, parsha, holidays, zmanim, and Omer
- `@hebcal/learning` for Daf Yomi and other learning schedules

All calendar data is computed algorithmically on the device, so there is nothing to download or sync and it works offline by default.

### Content pipeline (`/pipeline`)
- Node.js + TypeScript
- Source: the `Sefaria-Export` repository
- Output: versioned, content-hashed SQLite packs

### Backend (thin)
| Concern | Choice |
|---|---|
| Content hosting | S3 + CloudFront (static packs + `manifest.json`) |
| Auth + user data | Supabase (Auth, Postgres, Row Level Security) |
| CI/CD | GitHub Actions (pipeline build, pack upload, app builds via EAS) |

---

## Architecture

```
                         +--------------------------+
                         |   Sefaria-Export (git)   |
                         +------------+-------------+
                                      |
                              (CI: GitHub Actions)
                                      v
                         +--------------------------+
                         |  /pipeline (Node + TS)   |
                         |  - parse nested JSON     |
                         |  - license filter        |
                         |  - flatten to schema     |
                         |  - build SQLite packs    |
                         |  - gzip + sha256         |
                         +------------+-------------+
                                      |
                                      v
                         +--------------------------+
                         |   S3 + CloudFront        |
                         |   packs/*.sqlite.gz      |
                         |   manifest.json          |
                         +------------+-------------+
                                      |  (download, verify, swap)
                                      v
+------------------------------------------------------------------+
|                          Mobile App (Expo)                       |
|                                                                  |
|  +-------------+   +----------------+   +---------------------+  |
|  |  UI Layer   |<->|  Zustand State |<->|   Domain Services   |  |
|  | Timeline    |   |                |   | CalendarService     |  |
|  | Siddur      |   +----------------+   | SiddurService       |  |
|  | Daf Reader  |                        | TextService         |  |
|  | Settings    |                        | PackManager         |  |
|  +-------------+                        | SyncService         |  |
|                                         +----------+----------+  |
|                                                    |             |
|        +-------------------+      +----------------v----------+  |
|        | @hebcal/core      |      |   SQLite (expo-sqlite)    |  |
|        | @hebcal/learning  |      |   content packs (read)    |  |
|        | (computed, no I/O)|      |   user.db (read/write)    |  |
|        +-------------------+      +----------------+----------+  |
+---------------------------------------------------|--------------+
                                                    |  (push/pull when online)
                                                    v
                                      +---------------------------+
                                      |   Supabase                |
                                      |   Auth + Postgres + RLS   |
                                      |   bookmarks, highlights,  |
                                      |   settings                |
                                      +---------------------------+
```

### Key design decisions

1. **Compute, don't fetch, for calendar data.** Hebcal runs on-device. No API calls, no staleness.
2. **Immutable content packs.** Each pack is a read-only SQLite file identified by version and hash. Updating means swapping in a new file, so there is no partial-merge risk and rollback is trivial.
3. **Two databases on device.** Content packs are read-only and replaceable. `user.db` holds user-generated data and is never overwritten by pack updates.
4. **Thin backend.** The server stores only static files and per-user sync data. Everything else is client-side.
5. **Local-first writes.** Every user action writes to local SQLite first. Sync is a background concern.

---

## Content Pipeline

Runs in CI, never on a device.

1. **Fetch:** clone or update `Sefaria-Export`.
2. **Select:** filter to the configured MVP set (Siddur, Tanakh, Bavli with Rashi and Tosafot).
3. **License check:** read each text version's license metadata and exclude anything that is not openly licensed. Excluded versions are logged in the build report.
4. **Normalize:** flatten the nested JSON structure into rows with a stable reference (`ref`) and order.
5. **Build packs:** write one SQLite file per pack (for example `siddur-ashkenaz`, `bavli-berakhot`), with FTS5 indexes.
6. **Package:** gzip each pack and compute its sha256.
7. **Publish:** upload to S3 and regenerate `manifest.json`.

### Manifest format

```json
{
  "schemaVersion": 1,
  "generatedAt": "2026-11-20T00:00:00Z",
  "packs": [
    {
      "id": "bavli-berakhot",
      "version": "2026.11.1",
      "url": "https://cdn.example.com/packs/bavli-berakhot-2026.11.1.sqlite.gz",
      "sizeBytes": 4200000,
      "sha256": "<hex>",
      "bundled": false
    }
  ]
}
```

---

## Data Model

### Content pack schema (read-only)

```sql
CREATE TABLE segments (
  id        INTEGER PRIMARY KEY,
  ref       TEXT NOT NULL,        -- e.g. "Berakhot 2a:1"
  work      TEXT NOT NULL,        -- e.g. "Berakhot"
  layer     TEXT NOT NULL,        -- 'gemara' | 'rashi' | 'tosafot' | 'siddur' ...
  ord       INTEGER NOT NULL,     -- display order within the work
  text_he   TEXT,
  text_en   TEXT,
  version   TEXT NOT NULL,        -- source version identifier
  license   TEXT NOT NULL
);

CREATE TABLE links (
  ref_a TEXT NOT NULL,
  ref_b TEXT NOT NULL,
  type  TEXT NOT NULL             -- 'commentary' | 'reference' | ...
);

CREATE VIRTUAL TABLE segments_fts USING fts5(text_he, text_en, content='segments', content_rowid='id');
CREATE INDEX idx_segments_ref ON segments(ref);
CREATE INDEX idx_links_a ON links(ref_a);
CREATE INDEX idx_links_b ON links(ref_b);
```

### User database schema (read/write)

```sql
CREATE TABLE settings (
  key        TEXT PRIMARY KEY,
  value      TEXT NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE TABLE bookmarks (
  id         TEXT PRIMARY KEY,    -- uuid
  ref        TEXT NOT NULL,
  kind       TEXT NOT NULL,       -- 'position' | 'saved'
  updated_at INTEGER NOT NULL,
  deleted    INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE highlights (
  id         TEXT PRIMARY KEY,
  ref        TEXT NOT NULL,
  start_off  INTEGER,
  end_off    INTEGER,
  color      TEXT,
  note       TEXT,
  updated_at INTEGER NOT NULL,
  deleted    INTEGER NOT NULL DEFAULT 0
);
```

---

## Offline and Sync Strategy

### Content packs
- On launch (and periodically), the app fetches `manifest.json`.
- Missing or outdated packs are downloaded in the background (Wi-Fi only by default).
- Each download is verified against its sha256, decompressed to a temp file, then atomically renamed into place.
- Core packs (siddur and the current period's data) ship in the app bundle, so first launch is fully functional offline.

### User data
- All writes go to local `user.db` first.
- A background `SyncService` pushes rows changed since the last sync to Supabase and pulls remote changes.
- Conflict resolution for MVP is **last-write-wins on `updated_at`**, with soft deletes (`deleted` flag) so deletions propagate.
- Row Level Security ensures each user can access only their own rows.
- CRDT-based sync is deferred until real-time chavruta mode.

---

## Repository Structure

```
zman/
├── app/                      # Expo app
│   ├── app/                  # Expo Router routes
│   ├── src/
│   │   ├── components/       # UI primitives (RTL-first)
│   │   ├── features/
│   │   │   ├── timeline/
│   │   │   ├── siddur/
│   │   │   ├── daf/
│   │   │   └── settings/
│   │   ├── services/         # Calendar, Siddur, Text, PackManager, Sync
│   │   ├── db/               # SQLite access, migrations
│   │   ├── store/            # Zustand stores
│   │   └── theme/            # Typography, colors, Shabbat mode
│   ├── assets/fonts/
│   └── package.json
├── pipeline/                 # Content build pipeline
│   ├── src/
│   │   ├── fetch.ts
│   │   ├── license.ts
│   │   ├── normalize.ts
│   │   ├── build-pack.ts
│   │   └── manifest.ts
│   └── package.json
├── docs/
├── .github/workflows/
└── README.md
```

---

## Getting Started

### Prerequisites
- Node.js 20+
- A clone of [`Sefaria-Export`](https://github.com/Sefaria/Sefaria-Export)
- Xcode and/or Android Studio for device builds

### Build a content pack
```bash
cd pipeline
npm install
SEFARIA_EXPORT_PATH=../../Sefaria-Export npm run build -- --pack bavli-berakhot-2a
# Output: pipeline/out/bavli-berakhot-2a.sqlite
```

### Run the app
```bash
cd app
npm install
npx expo prebuild
npx expo run:ios      # or run:android
```

Copy the generated `.sqlite` file into `app/assets/packs/` to bundle it for offline use.

### Verify offline behavior
Enable airplane mode on a physical device and confirm that the home screen (Hebrew date, parsha, candle lighting for Jerusalem) and the daf reader both work.

---

## MVP Roadmap

| Weeks | Deliverable |
|---|---|
| 1-2 | Expo scaffold, RTL and typography system, Hebcal-powered timeline with 3-4 layers |
| 3-4 | Content pipeline, pack downloader, siddur reader with nusach selection and auto-open |
| 5-6 | Daf Yomi reader (Gemara + Rashi + Tosafot), saved position, Shabbat mode |
| 7-8 | Account sync, notifications (candle lighting, Omer), TestFlight / beta |
| Post-MVP | Argument maps, chavruta mode, vocabulary SRS, plugin system |

### Milestone 0 (first deliverable)
A single vertical slice that proves the riskiest pieces:

- `/pipeline` parses Berakhot 2a (Gemara, Rashi, Tosafot; Hebrew + English) from a local `Sefaria-Export` clone, checks each version's license, and writes `bavli-berakhot-2a.sqlite`.
- `/app` loads that file with `expo-sqlite` and renders the daf in a scrollable RTL view.
- The home screen shows today's Hebrew date, parsha, and candle-lighting time for Jerusalem using `@hebcal/core`.
- **Acceptance:** works in airplane mode on a physical device, and this README documents the schema and how to run the pipeline.

---

## Licensing and Content Sources

- **Texts:** sourced from Sefaria. Licenses vary per text version, and the pipeline includes only openly licensed versions and records each segment's license in the pack. Attribution to Sefaria and the original contributors must be shown in the app.
- **Calendar and zmanim:** computed with Hebcal's open-source libraries (see their repositories for license terms).
- **Fonts:** Frank Ruhl Libre is licensed under the SIL Open Font License.
- **App code:** license to be decided by the project owner.

> Halachic times and customs vary by community. The app presents computed times and sourced texts as study and reminder aids and does not replace a rabbi's guidance.
