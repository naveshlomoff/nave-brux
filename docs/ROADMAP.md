# BruxTrack Roadmap — Phase-by-Phase

> Condensed reference for Claude Code. Full disclosure document lives outside the repo (patent-protected).

## Goal

Move BruxTrack from a passive frequency-threshold detector to a learning system that computes the patent-protected BSI severity score, adapts to each user, and eventually classifies events with on-device ML.

---

## Phase 0 — Supabase Connection (THIS WEEK) 🟡

**Goal:** Every night and every labeled event saved to Supabase automatically. Audio clips uploaded to Storage. Zero data loss from this point forward.

**Tasks:**
- [x] Create Supabase project (`bruxtrack` / `ukoswihzqpztfypqhdnc`)
- [x] Create tables: `nights`, `events`, `population_stats`
- [x] Create Storage bucket: `audio-clips`
- [x] Add Supabase JS SDK via CDN
- [ ] **Verify RLS policies on all tables** (currently the repo is public)
- [ ] Implement `syncNightToCloud()` — already exists in `index.html`, needs hardening
- [ ] Implement `uploadClip()` — already exists, needs retry-on-failure
- [ ] Implement `loadLastNightFromCloud()` — already exists, needs fallback paths
- [ ] Add visible sync status indicator (✓ synced / ⚠ pending / ✗ failed)
- [ ] Test end-to-end: clear IndexedDB, reopen app, verify data loads from cloud

**Definition of done:** Clearing browser storage does not lose a single night.

---

## Phase 1 — Data Collection (2–3 weeks) ⬜

**Goal:** Accumulate 5–10 nights of fully labeled data. Every candidate event listened to and labeled (bruxism / not bruxism / unsure).

**Targets:**
- 30+ confirmed bruxism events
- 30+ confirmed non-bruxism events
- Notes on false positive rate per night

**App tasks:**
- [ ] CSV export of all labeled events
- [ ] False-positive-rate display in settings

---

## Phase 2 — BSI Formula (1 week) ⬜

**Goal:** Patent-protected BSI score computed in Supabase Edge Function after every night.

**Tasks:**
- [ ] Set up Deno Edge Function runtime
- [ ] Implement `compute_bsi()` — Fp, Dp, Ap, C, weighted sum
- [ ] Seed `population_stats` from initial nights
- [ ] Trigger BSI computation on night sync
- [ ] Display BSI in morning report (replace raw score)
- [ ] Severity bands: None / Mild / Moderate / Severe / Very Severe (per P2 Section 4.3)
- [ ] Compute and store EBI = Σ BSI(n) / N
- [ ] Compute and store EBI² = Σ BSI(n)² / N

---

## Phase 3 — Adaptive Threshold (1 week) ⬜

**Goal:** Per-user, per-environment threshold replaces the static `sensitivity=45`.

**Algorithm:**
- Compute personal_baseline = 90th percentile of non-bruxism event amplitudes (over 5+ labeled nights)
- New threshold = personal_baseline × 1.15
- Threshold updated nightly, synced to app on open

---

## Phase 4 — ML Classifier (15+ labeled nights) ⬜

**Goal:** TensorFlow.js binary classifier replaces threshold detector. Runs in browser, <1ms per event.

**Features (9):** duration, peakHz, amplitude, dB, spectral centroid, spectral spread, spectral flatness, periodicity score, time-of-night.

**Pipeline:** Export labeled events as CSV → train sklearn model in Python → convert to TF.js → embed in app. Retrain trigger: every 50 new labels.

---

## Phase 5 — Population Registry (multi-user) ⬜

**Goal:** Opt-in anonymized data → true population percentile normalization for BSI. Supports future clinical validation. Maps to Patent P7.

---

## Patent Alignment Quick Reference

| Phase | Patent Family | Note |
| --- | --- | --- |
| 0 | P2 §4.5 | Data persistence prerequisite for normalization |
| 1 | P2 §4.1 | C ratio = confirmed/total — core BSI input |
| 2 | P2 claims 1–14 | BSI + EBI direct implementation |
| 3 | P1 | Personalized acoustic detection |
| 4 | P4 | Acoustic-only embodiment of sensor fusion |
| 5 | P7 | Population registry |
