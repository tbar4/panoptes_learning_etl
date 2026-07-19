# Part VII (Wrap-Up) — Author Report

**Date:** 2026-07-19
**Files written:**
- `src/06-wrap/panoptes-integration.md` — Where This Plugs Into Panoptes
- `src/06-wrap/graduation-path.md` — The Graduation Path

Both were placeholder stubs ("_Chapter under construction._"); now full prose chapters. No quiz, no runnable code (all snippets marked `rust,ignore`). Voice matched to `intro.md` and the first course's `thesis-integration.md` (opening `> **Kind:** Wrap-up.`, sparing `<div class="callout">`, second person, argue-why-before-how, em-dash cadence).

## panoptes-integration.md — what it argues
- **The connection is a file contract, not a call.** ETL writes normalized JSONL; `panoptes-gen` reads it later. No shared DB, no in-process crate dep — mirrors Panoptes' `scenarios/` directory. The contract is one shape: the `#[serde(tag = "kind", rename_all = "snake_case")]` `Row` enum, so every line is self-describing.
- **File-carries table** (verified against shipped code): `space_objects.jsonl` (`Row::SpaceObject`), `conjunctions.jsonl` (keystone `Row::Conjunction`), `launches.jsonl` (`Row::Launch`, prose `description`), `embeddings.jsonl` (`EmbeddedRow {id,text,vector}`), `runs.jsonl` (`RunRecord` ledger).
- **Accuracy note surfaced:** `OrbitalElements` is NOT a top-level file/`Row` variant — it lives inside `SpaceObject.elements`; derived quantities (period/apogee/perigee/`orbit_class`) are computed on demand, not stored. Chapter states this explicitly.
- **Keystone → `ca_geo`:** mapped the real `Conjunction` fields (`tca`, `primary`/`secondary` → join into `space_objects.jsonl` + `operator`, `miss_distance_km`, `collision_probability`) onto scenario params (`time_pressure`, `attribution_confidence`, stakes). ETL owns *what is true*; `reversibility` + `info_request` are Panoptes' framing knobs, not ETL output.
- **Retrieval feeds Panoptes two ways:** grounding a vignette's prose, or as a retrieval eval treatment arm — both over the same `embeddings.jsonl`/`top_k`. Only prose embedded (`Row::prose()` returns `Some` only for `Launch`). Augment+generate is Panoptes' job, out of scope (design spec 4.5).

## graduation-path.md — what it argues
Design-only (per spec 2.1/2.3/5). Each item named with the EXACT seam:
- **DuckDB + VSS over Parquet** → behind the `Sink` trait + `VectorIndex` (`from_jsonl`/`top_k`) boundary; replaces `JsonlSink` + brute-force `cosine`. Deferred: thousands of rows → brute-force is *correct*, not a compromise.
- **`governor`** → the `TokenBucket` constructor argument (`SpaceTrackSource::new(base, creds, limiter)`); `extract` untouched. Deferred: rate limiting *is* the Part III lesson.
- **`sgp4`** → *consumes* the parsed `OrbitalElements` (7 Keplerian fields); explicitly distinct from parsing — a downstream consumer, not a pipeline stage. Deferred: propagation is a rabbit hole; real conjunctions arrive pre-computed from Space-Track/SOCRATES.
- **Real scheduler/daemon** → loops over `Executor::run()` + reads the ledger watermark (`RunRecord.watermark`, `Ledger::last_success`, `Pipeline::is_fresh`/`refresh_after`); retries reused from Part IV. Deferred: painful to TDD (time/concurrency/flakiness); earns its own sequel.
- Closes on the four-seams recap proving each addition is additive, not a rewrite.

## Accuracy checks performed
Read `domain.rs`, `vector.rs`, `pipeline.rs`, `content.rs`, `ledger.rs`, `sink_jsonl.rs`, `retry.rs`, `executor.rs`, `main.rs`, and grepped `limiter.rs`. All type names, field lists, method signatures, constructor shapes, and output filenames in both chapters were cross-checked against the shipped source. `TokenBucket::new(30, 0.5)` matches `main.rs`; `Row::prose()` Launch-only behavior confirmed in `vector.rs`.

Did NOT run `mdbook build` (per instructions).
