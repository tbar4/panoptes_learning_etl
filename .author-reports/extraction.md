# Author Report — Part II: The Extraction Arc (CelesTrak)

Authored 2026-07-18/19. Source of truth: the verified `panoptes_etl` crates
(`etl-core`: `pipeline.rs`, `domain.rs`, `sink_jsonl.rs`, `error.rs`, `ids.rs`;
`etl-sources`: `tle.rs`, `celestrak.rs`, `http.rs`). Everything below was checked
against that code, which compiles and passes tests.

## Files written (overwrote stubs)

- `src/01-extraction/concept-source-etl.md` — Source/Transform/Sink traits, the
  E/T/L boundary, the tagged `Row` enum, `dyn Source` object-safety, and the
  explicit one-for-one mapping paragraph (receipt-fetcher toy ↔ real ETL).
- `src/01-extraction/concept-tle.md` — fixed-width TLE parsing: 1-based↔0-based
  column mapping, the mod-10 checksum (minus = 1), epoch year pivot, implied
  eccentricity decimal. Uses the canonical ISS TLE from `tle.rs` as the worked
  example. Trap sections: silent off-by-one slice, and forgotten `'-'` in checksum.
- `src/01-extraction/build-workspace-domain.md` — Phase 0 / Task 1.1 (etl-core):
  workspace conversion, domain field lists + derive stacks + classifier
  thresholds + physics constants, the three trait signatures, the `Row` contract,
  real test names. NO implementation.
- `src/01-extraction/build-celestrak.md` — Tasks 1.1(TLE)+1.2: etl-sources crate,
  TLE parser interface + column map, `classify_status` mapping, CelestrakSource +
  CelestrakTransform spec, wiremock note, real test names. NO implementation.
- `src/01-extraction/build-jsonl-sink.md` — Task 1.3: append-only JsonlSink spec +
  real test names. NO implementation.
- `src/01-extraction/quiz.md` — intro + `{{#quiz ../../quizzes/01-extraction.toml}}`.
- `quizzes/01-extraction.toml` — 7 questions (2 Tracing, 3 MultipleChoice, 2 ShortAnswer).

## Examples compiled + output captured (scratch cargo project + rustc)

All CONCEPT-page runnable blocks were compiled and run in isolation
(`scratchpad/etl_verify` for dep-bearing code; `rustc --edition 2021` for std-only):

| Example | Block kind | Verified output |
| --- | --- | --- |
| receipt-fetcher E/T/L (concept-source-etl) | ```rust``` | `fetched from corner-shop` / `parsed 2 rows` / `stored 2 rows` / `first: LineItem { item: "coffee", cents: 450 }` |
| tagged `Row` serde (concept-source-etl) | ```rust``` | `{"kind":"launch","name":"Starlink G-99"}` / `true` |
| async `Box<dyn Fetch>` (concept-source-etl) | ```rust,ignore``` (async-trait+tokio) | `corner-shop -> "corner-shop"` — compiles+runs |
| generic-method object-safety (concept-source-etl) | ```rust,compile_fail``` | `error[E0038]: the trait ... is not dyn compatible` |
| fixed-width `cols()` toy (concept-tle) | ```rust``` | `station: "KWJ"` / `range: "412"` / `elevation: 0.123456` |
| off-by-one slice trap (concept-tle) | ```rust``` | `correct REC[0..3]: "KWJ"` / `naive REC[1..3]: "WJ"` |
| ISS checksum (concept-tle) | ```rust``` | both lines computed 7 / claim 7 / after 1-digit edit: 8 |
| epoch pivot loop (concept-tle) | ```rust``` | `57->1957 99->1999 00->2000 08->2008 26->2026 56->2056` |
| implied eccentricity (concept-tle) | ```rust``` | `eccentricity = 0.0006703` |

`ignore`/`compile_fail` blocks carry a dep-list note per the playground rule
(`reqwest`/`wiremock`/`chrono`/`async-trait` are not playground-safe).

## Tracing quiz programs (compiled by rustc, std-only)

- Q1 checksum arithmetic: `doesCompile = true`, `stdout = "8"` — verified
  (`checksum("2 8-B7")` = 2+0+8+1+0+7 = 18, 18%10 = 8).
- Q2 generic-method object-safety: `doesCompile = false` — verified (E0038).

## UUIDs (all unique, uuidgen-generated, TOML validated with tomllib)

- 05f81b55-f77a-4aae-a2bf-02fed921a9d6 — Tracing, checksum arithmetic
- 9af58549-44b6-4dee-8cc6-5a239daea69f — Tracing, object-safety compile-fail
- 728cfc1c-a25f-474d-95c7-b031b6bc3d10 — MC, why `Row` is tagged
- 0b6798d6-6384-42b4-9a41-86c80a687e0b — ShortAnswer, which stage owns parsing (Transform)
- 8840b6a7-4973-4b6c-bf7a-eb99d7058b8b — MC, dyn-Source object-safety (async_trait)
- e8c39181-927b-4e7a-adb2-275e5cbd2005 — MC, checksum minus-sign rule
- 9241337e-3ceb-4db3-a228-cfe16e61b6b0 — ShortAnswer, epoch pivot (08 -> 2008)

## Assumptions / deviations from the plan (plan diverged from the shipped code)

1. **Quiz path.** Task said `quizzes/01-extraction.toml`; the plan text says
   `02-extraction.toml`. Used `01-extraction.toml` (matches the task and the
   existing SUMMARY.md links). The `{{#quiz}}` directive resolves relative to
   the book root as `../../quizzes/01-extraction.toml`.
2. **Real test names over plan names.** The shipped code is the source of truth
   ("everything must match this code"), so build pages list the *actual* verified
   test names, which differ from the plan in places:
   - TLE: `checksum_matches_known_lines` (not `_line`), `decodes_2008_epoch`
     (not `_2026_`), plus `reads_norad_and_designator`, `rejects_misaligned_line_pair`.
   - CelesTrak error taxonomy: `server_error_is_transient_network`,
     `rate_limit_is_retryable_with_backoff`, `not_found_is_permanent_not_retryable`
     (the plan's generic `http_error_is_network_error` is not what shipped).
   - JSONL: `writes_one_row_per_line_and_roundtrips`, `append_accumulates_across_loads`
     (the plan's three-way split is folded into these two shipped tests).
3. **`parse_tle` signature.** Plan showed `parse_tle(name, line1, line2)`; shipped
   code is `parse_tle(line1, line2) -> OrbitalElements` (the name is handled by
   `CelestrakTransform`). Taught the shipped signature.
4. **Error taxonomy.** Shipped `EtlError` has `Network`/`RateLimited`/`Auth`/
   `Permanent`/`Parse`/`Io`, and a `classify_status` helper in `etl-sources/http.rs`.
   Taught these (the plan's Task 0.2 listed a slightly different set). Added a
   short Part B in build-celestrak for `http.rs` since `CelestrakSource::extract`
   depends on `classify_status` — keeps "no chapter relies on un-introduced machinery."
5. **build-workspace-domain scope.** Task maps it to "Phase 0 / Task 1.1" and lists
   domain + traits + workspace as its givens, so it builds the `etl-core` crate
   (domain.rs + pipeline.rs + workspace), treating `EtlError`/`NoradId` as Part I
   prerequisites that get their home here. The etl-sources crate + TLE parser
   (plan Task 1.1) is built in build-celestrak, where the transform needs it.
6. **No `mdbook build` run** (per instructions — you run it centrally). Verified
   only that examples and Tracing programs compile/run in isolation.

## Not done / for the central build

- `mdbook build` (validates Tracing via real rustc, resolves `{{#quiz}}` once).
- Answer-key anchor `../appendix-answer-key.html#task-...` links were NOT added
  (the appendix task anchors for Part II are not yet finalized); build pages point
  readers to the concept pages and the spec instead.
