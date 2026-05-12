# BruxTrack / BruxAI — Claude Code Context

> This file is read automatically by Claude Code at the start of every session.
> It is the project's source of truth for architecture, conventions, and current status.

---

## 1. Project Overview

**BruxTrack** is a Progressive Web App (PWA) that detects and quantifies bruxism (teeth grinding) during sleep using only a smartphone's microphone.

- **Live URL:** https://naveshlomoff.github.io/BruxAI/
- **GitHub:** https://github.com/naveshlomoff/BruxAI (currently public — see Security section)
- **Backend:** Supabase project `ukoswihzqpztfypqhdnc`
- **Stage:** Phase 1 — passive detector working, transitioning to learning system
- **IP:** Patent family P1–P7 filed (P2 covers BSI + EBI formulas — see `docs/ROADMAP.md` for phase-to-patent mapping)

The app records audio all night in the browser, detects candidate bruxism events via frequency analysis (250–3500 Hz band), saves rolling 20-second audio clips per event in IndexedDB, and presents a morning report for manual labeling. Phase 0 adds Supabase cloud sync; later phases add the patented BSI severity score, adaptive thresholds, and ML classification.

---

## 2. Architecture (Current State)

**Single-file PWA.** Everything lives in `index.html` (~925 lines):
- HTML structure: 4 screens (Record, Events, History, Settings) toggled via CSS classes
- Inline CSS: CSS variables for theming, mobile-first
- Inline JavaScript:
  - Web Audio API for recording + frequency analysis
  - MediaRecorder for rolling audio clips
  - IndexedDB (`BruxTrackDB`) for local persistence — stores `nights`, `events`, `clips`
  - Supabase JS SDK (via CDN) for cloud sync

**Target architecture (per Roadmap):**
| Layer | Responsibility | Status |
| --- | --- | --- |
| PWA (index.html) | Audio capture, event detection, labeling UI | Built |
| Supabase Postgres | `nights`, `events`, `population_stats` tables | Schema partially live |
| Supabase Storage | `audio-clips` bucket for `.webm` files | Live |
| Supabase Edge Functions (Deno) | BSI computation, adaptive threshold updates | **Not built yet** |

---

## 3. Supabase Schema

Connection is hardcoded in `index.html` (see Security section about this):
- URL: `https://ukoswihzqpztfypqhdnc.supabase.co`
- Anon/publishable key: in `index.html` (publishable, not service_role)

### `nights` table
`id` (uuid PK), `date` (date), `duration_sec` (int), `event_count` (int), `confirmed_count` (int), `rejected_count` (int), `confirmation_ratio` (float — the C input to BSI), `bsi_score` (float), `ebi_cumulative` (float), `ebi2_cumulative` (float), `created_at` (timestamp).

### `events` table
`id` (uuid PK), `night_id` (FK → nights.id), `wall_time` (text HH:MM), `timestamp_sec` (int), `duration_sec` (float), `peak_hz` (int), `amplitude` (float), `db_level` (float), `bruxism_score_raw` (int 0–100), `label` (text: `bruxism` / `not_bruxism` / `unsure` / `unconfirmed`), `has_clip` (bool), `clip_path` (text → Storage path).

### `population_stats` table
For BSI percentile normalization. Columns: `metric` (text), `p10`, `p25`, `p50`, `p75`, `p90` (floats), `n_nights` (int), `updated_at` (timestamp).

### Storage
Bucket `audio-clips`. Path convention: `{date}/{event_id}.webm`.

---

## 4. Where We Are on the Roadmap

| Phase | Description | Status |
| --- | --- | --- |
| **0** | **Supabase connection — zero data loss** | **🟡 IN PROGRESS** |
| 1 | Collect 5–10 labeled nights | ⬜ Waiting on Phase 0 |
| 2 | Implement BSI formula as Edge Function | ⬜ |
| 3 | Adaptive personal threshold | ⬜ |
| 4 | ML classifier (on-device TF.js) | ⬜ |
| 5 | Population registry + true BSI normalization | ⬜ |

**Immediate priority:** finish Phase 0. Until cloud sync is bulletproof, every night of data is at risk if the browser cache clears.

See `docs/ROADMAP.md` for the full phase-by-phase plan.

---

## 5. Security Notes — READ BEFORE TOUCHING ANYTHING SENSITIVE

⚠️ **The GitHub repo is currently PUBLIC.** This means:

1. **The Supabase publishable key in `index.html` is world-readable.** This is OK *only if* Row Level Security (RLS) is properly configured on every table. Verify RLS before any production use.
2. **Never commit `service_role` keys, JWT secrets, or any private keys.** Use Supabase Edge Function secrets for those.
3. **Patent disclosures (`*.docx` in our local docs) must never be committed.** They are protected by `.gitignore`.
4. The roadmap and BSI formula details ARE in the repo — that's intentional, since the patent is filed.

**Before making the repo public-facing or adding more sensitive integrations:** consider switching to private, or splitting into a public frontend repo and a private backend repo.

---

## 6. The BSI Formula (P2 Patent — Acoustic-Only Embodiment)

Implementation target for Phase 2. Computed nightly in a Supabase Edge Function:

```
BSI = min(100, W₁·Fₚ + W₂·Dₚ + W₃·Aₚ) × C

Where:
  Fₚ = percentile rank of (events/hour) vs population stats
  Dₚ = percentile rank of (mean event duration) vs population stats
  Aₚ = percentile rank of (mean 250–3500 Hz energy) vs population stats
  C  = confirmed_events / total_candidate_events  (0–1)
  W₁ = 0.45, W₂ = 0.30, W₃ = 0.25  (sum to 1.0)
```

**EBI (Energy Bruxism Index)** — longitudinal integral:
```
EBI(N)  = Σ BSI(n) / N           for n = 1..N
EBI²(N) = Σ BSI(n)² / N          (quadratic embodiment — weights heavy nights)
```

Population stats are bootstrapped from the user's own first 5–10 nights and improved as the registry grows (Phase 5). This bootstrapping is patent-covered — do not "optimize" it away.

---

## 7. Coding Conventions

**General principles for this codebase:**

- **Stay single-file until Phase 2.** Don't pre-emptively split `index.html` into modules. We'll split when it actually hurts, not before. When we do split, the target structure is: `index.html` (shell) + `js/audio.js`, `js/supabase.js`, `js/ui.js`, `js/db.js`, `js/bsi.js`.
- **No build step.** This is a static PWA. No webpack, no bundler, no npm install. Libraries come from CDN (Supabase JS SDK is already loaded this way).
- **Mobile-first.** The user runs this on a phone, in Chrome, in a bedroom, in the dark. Touch targets ≥44px, no hover-only UI, dark mode acceptable but not required yet.
- **Battery-aware.** This thing runs all night. Avoid setInterval timers shorter than 100ms unless audio analysis requires it. Avoid keeping the screen on if not needed.
- **CSS variables for everything.** All colors, radii, fonts come from `:root`. Don't hardcode hex values.
- **Vanilla JS, no framework.** No React, no Vue. Keep the dependency graph flat.
- **IndexedDB is the local truth; Supabase is the cloud truth.** A successful sync means both agree. Never delete from IndexedDB just because something synced — keep it as offline fallback.
- **Every Supabase call wrapped in try/catch.** Network failures at 3 AM are real. Failures should never crash the recorder.

**Naming:**
- Functions: `camelCase` (`syncNightToCloud`, `analyzeLoop`)
- DB columns and Supabase tables: `snake_case` (`night_id`, `wall_time`)
- JS state object keys: `camelCase` to match function naming
- Map between them explicitly at the sync layer (don't rely on Supabase auto-conversion)

---

## 8. Don'ts

- **Don't break the recorder.** It's the one thing that works reliably. Any change to `analyzeLoop()` or `startRecording()` requires testing a full overnight run before merging.
- **Don't add dependencies casually.** Each new CDN script is a point of failure if the user is offline.
- **Don't change the IndexedDB schema without a migration.** Users have nights of data locally. Bump the DB version in `openDB()` and write an `onupgradeneeded` migration.
- **Don't commit large audio files** (`*.webm`, `*.wav`, `*.mp3`). Test clips go in `test/clips/` which is gitignored.
- **Don't store user-identifying info in the repo or in Supabase tables** beyond what the app already collects. No emails, no names, no device IDs.
- **Don't reformulate the BSI formula.** The weights (0.45 / 0.30 / 0.25), the band (250–3500 Hz), the bootstrap approach — these are patent claims. Implement them exactly.

---

## 9. Useful Commands

```bash
# Run the app locally (no build, just serve)
python -m http.server 8000
# Then open http://localhost:8000 on your phone (same Wi-Fi)

# Check the deployed version
# https://naveshlomoff.github.io/BruxAI/

# Deploy: just push to main — GitHub Pages auto-deploys
git add -A && git commit -m "..." && git push

# Pull latest
git pull --rebase origin main
```

---

## 10. When You're Unsure

If you (Claude Code) are about to do something that touches:
- The BSI formula or its inputs
- The Supabase schema (adding/renaming columns, RLS policies)
- The recorder's audio loop
- The patent-aligned algorithmic logic (adaptive threshold, EBI, etc.)

**Stop and ask.** Don't guess. The user is the inventor on the patent — defer to their judgment on anything that might affect patent scope or clinical validity.

For pure plumbing — UI tweaks, refactors, error handling, sync robustness — proceed normally and show diffs.
