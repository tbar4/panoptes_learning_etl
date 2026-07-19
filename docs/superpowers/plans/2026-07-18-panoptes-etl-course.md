# Panoptes ETL Course — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `panoptes_etl` — a space-data ETL that feeds the Panoptes eval harness — test by test, and generate a compiler-verified mdBook course from the verified build.

**Architecture:** A Cargo workspace that grows one crate per arc: `etl-core` (seams: domain, traits, ledger, content-addressing, retriever), `etl-sources` (four sources + TLE parser + limiter + retry + embedder), `etl-orchestrate` (DAG executor over `etl-core` traits only), and the `panoptes-etl` binary (CLI wiring). Every arc builds a working pipeline and installs one seam; the last two arcs spend the seams (DAG executor, then RAG).

**Tech Stack:** `reqwest` + `tokio`, `serde`/`serde_json` (JSONL), `clap`, `chrono`, `thiserror` (+ `anyhow` in bin), `sha2`, `async-trait`; `wiremock` + `pretty_assertions` for tests. Hand-rolled where it is the lesson: TLE parser, token-bucket limiter, retry/backoff, DAG executor, cosine retriever.

## Global Constraints

- **Rust edition 2024**, toolchain ≥ 1.96 (verified: cargo 1.96.0).
- **No test hits a live network** — every external API is mocked with `wiremock`, exactly as the first course mocked the Anthropic API.
- **Dependency direction (must hold):** `etl-core` depends on nothing in-workspace; `etl-sources` and `etl-orchestrate` depend only on `etl-core`; **`etl-orchestrate` must NOT depend on `etl-sources`** (the dependency-inversion proof); only `panoptes-etl` (bin) names concrete sources.
- **Load contract:** normalized rows are written as JSONL that `panoptes-gen` reads. `Row` is a `#[serde(tag = "kind", rename_all = "snake_case")]` enum.
- **Course methodology:** the course is generated from THIS verified plan. One seam per arc; no build chapter uses machinery no earlier chapter demonstrated; concept-before-build; one quiz per arc; every example and Tracing quiz compiled by rustc before publish.
- **Audience:** sequel to the first Panoptes course — ownership, serde, traits, async/tokio, wiremock, clap, and the TDD loop are assumed.

## Verification status legend

- ✅ **verified** — code built and tests run green in `panoptes_etl` (output captured).
- ◻︎ **planned** — files, interfaces, test names, and specs fixed; code verified when its arc is built (the methodology forbids shipping un-run code into chapters).

---

## Phase 0 — Foundations crate (`etl-core`) ✅ verified

Maps to course **Part I (Foundations)** + the shared types Parts II–VI build on. Built and green (13 tests).

### Task 0.1: Workspace + `etl-core` skeleton ✅
**Files:** Create `Cargo.toml` (virtual workspace, `workspace.dependencies`), `crates/etl-core/Cargo.toml`, `crates/etl-core/src/lib.rs`.
**Produces:** the `etl-core` crate; `workspace.dependencies` table every crate inherits from.

### Task 0.2: The error taxonomy ✅
**Files:** Create `crates/etl-core/src/error.rs`.
**Produces:** `EtlError` (`Network`, `RateLimited{retry_after_secs}`, `Auth`, `Parse`, `Io(#[from] io::Error)`), `EtlError::is_transient() -> bool`, `EtlError::retry_after() -> Option<Duration>`.
**Tests (green):** `network_and_rate_limit_are_transient`, `parse_and_auth_are_permanent`, `retry_after_only_on_rate_limit`.
**Given (spec for the build chapter):** transient = `Network | RateLimited`; everything else permanent. This classification is what the Part IV retry loop consumes.

### Task 0.3: Id newtypes ✅
**Files:** Create `crates/etl-core/src/ids.rs`.
**Produces:** `NoradId(pub u32)` with `Display` (zero-pad to 5), `FromStr` (→ `EtlError::Parse`), `#[serde(transparent)]`.
**Tests (green):** `display_is_zero_padded_to_five`, `parses_from_trimmed_digits`, `rejects_non_numeric`, `serializes_as_bare_integer`.

### Task 0.4: Domain model ✅
**Files:** Create `crates/etl-core/src/domain.rs`.
**Produces:** `OrbitClass` (`#[serde(rename_all = "SCREAMING_SNAKE_CASE")]`: LEO/MEO/GEO/HEO); `OrbitalElements` with derived `period_minutes`, `semi_major_axis_km`, `apogee_km`, `perigee_km`, `orbit_class`; `SpaceObject`, `Conjunction` (keystone), `Launch`.
**Given (constants):** `EARTH_MU_KM3_S2 = 398_600.4418`, `EARTH_RADIUS_KM = 6378.137`. Classifier: `ecc > 0.25 → HEO`; else `period < 128 → LEO`; else `period ∈ [1430,1450] → GEO`; else `MEO`.
**Tests (green):** `iss_is_low_earth_orbit` (period ≈ 92.90, class LEO, perigee ∈ 300..500), `geostationary_is_classified_geo` (period ≈ 1436.1, class GEO), `orbit_class_serializes_screaming`, `space_object_roundtrips_through_json`.

### Task 0.5: The E/T/L trait boundary ✅
**Files:** Create `crates/etl-core/src/pipeline.rs`; re-export from `lib.rs`.
**Produces:** `RawRecord{source, payload, fetched_at}`; `Row` enum (`#[serde(tag="kind", rename_all="snake_case")]`); `#[async_trait] Source: Send+Sync { fn name(&self)->&str; async fn extract(&self)->Result<Vec<RawRecord>,EtlError>; }`; `Transform: Send+Sync { fn transform(&self,&RawRecord)->Result<Vec<Row>,EtlError>; }`; `#[async_trait] Sink: Send+Sync { async fn load(&self,&[Row])->Result<usize,EtlError>; }`.
**Tests (green):** `row_tags_by_kind`, `a_source_can_be_boxed_as_dyn` (proves `Box<dyn Source>` works — the object-safety the orchestrator needs).

**Captured output (Phase 0):** `cargo test -p etl-core` → `13 passed; 0 failed`.

---

## Phase 1 — The Extraction Arc: CelesTrak ◻︎ planned  → course **Part II**

Installs the `Source` trait usage + the E/T/L split end to end. Adds `crates/etl-sources`.

### Task 1.1: `etl-sources` crate + TLE parser
**Files:** Create `crates/etl-sources/Cargo.toml` (dep: `etl-core`, `chrono`, `serde`; dev: `pretty_assertions`), `crates/etl-sources/src/lib.rs`, `crates/etl-sources/src/tle.rs`.
**Interfaces — Produces:** `pub fn parse_tle(name: &str, line1: &str, line2: &str) -> Result<OrbitalElements, EtlError>`; `pub fn tle_checksum(line: &str) -> u8`; `pub fn decode_epoch(yyddd: &str, frac: &str) -> Result<DateTime<Utc>, EtlError>`.
**Given (TLE wire format):** line-1 cols 19–20 = 2-digit epoch year (57–99 ⇒ 19xx, 00–56 ⇒ 20xx), cols 21–32 = day-of-year with fractional day; line-2 cols 9–16 inclination, 18–25 RAAN, 27–33 eccentricity (leading `.` implied), 35–42 arg-perigee, 44–51 mean anomaly, 53–63 mean motion; last col of each line = mod-10 checksum (digits sum, minus signs = 1, all else = 0).
**Test names:** `checksum_matches_known_line`, `rejects_bad_checksum` (→ `EtlError::Parse`), `decodes_2026_epoch`, `parses_iss_tle_to_elements` (ISS TLE ⇒ inclination ≈ 51.6, class LEO).
**Loop:** write failing checksum test → predict → implement `tle_checksum` → green → repeat for epoch, then full `parse_tle` → commit.

### Task 1.2: The CelesTrak source (E) + transform (T)
**Files:** Create `crates/etl-sources/src/celestrak.rs`.
**Interfaces — Consumes:** `Source`, `Transform`, `RawRecord`, `Row`, `parse_tle`. **Produces:** `CelestrakSource { base_url: String, group: String }` impl `Source` (GETs `{base}/NORAD/elements/gp.php?GROUP={group}&FORMAT=tle`); `CelestrakTransform` impl `Transform` (3-line TLE chunks ⇒ `Row::SpaceObject`).
**Given:** `Source::name` = `"celestrak"`; group default `"geo"`; a 404/5xx ⇒ `EtlError::Network`.
**Test names (wiremock):** `extract_returns_raw_tle_text`, `transform_parses_tle_blocks_to_space_objects`, `http_error_is_network_error`.

### Task 1.3: The JSONL sink (L) — the file contract
**Files:** Create `crates/etl-core/src/sink_jsonl.rs`; re-export.
**Interfaces — Produces:** `JsonlSink { path: PathBuf }` impl `Sink` (appends one JSON object per `Row` per line; returns count).
**Test names:** `writes_one_row_per_line`, `roundtrips_via_serde_jsonl`, `load_returns_count`.
**Quiz:** `quizzes/02-extraction.toml` — Tracing (TLE checksum arithmetic), MultipleChoice (why `Row` is tagged), ShortAnswer (which stage owns parsing).

---

## Phase 2 — The Authenticated Arc: Space-Track ◻︎ planned  → course **Part III**

Installs the run ledger + rate limiting.

### Task 2.1: Token-bucket rate limiter
**Files:** Create `crates/etl-sources/src/limiter.rs`.
**Interfaces — Produces:** `TokenBucket { capacity: u32, refill_per_sec: f64 }` with `async fn acquire(&self)` (awaits until a token is free). Uses `tokio::time`.
**Test names:** `bucket_allows_burst_to_capacity`, `bucket_blocks_when_empty_then_refills` (uses `tokio::time::pause`/`advance` — deterministic, no real sleep).
**Given:** Space-Track etiquette ⇒ default `capacity = 30`, `refill_per_sec = 0.5`.

### Task 2.2: Session auth + the Space-Track source
**Files:** Create `crates/etl-sources/src/spacetrack.rs`.
**Interfaces — Produces:** `SpaceTrackSource { base_url, creds: Credentials, limiter: TokenBucket }`; `Credentials::from_env() -> Result<Self,EtlError>` (reads `SPACETRACK_USER`/`SPACETRACK_PASS`; missing ⇒ `EtlError::Auth`). `Source::extract` POSTs `/ajaxauth/login`, keeps the cookie, GETs the CDM query, rate-limited.
**Given:** login failure ⇒ `EtlError::Auth`; emits `Row::Conjunction`.
**Test names (wiremock):** `login_then_query_uses_cookie`, `missing_credentials_is_auth_error`, `respects_rate_limit` (mock counts requests under paused time).

### Task 2.3: The run ledger
**Files:** Create `crates/etl-core/src/ledger.rs`; re-export.
**Interfaces — Produces:** `RunRecord { source: String, started_at, finished_at, outcome: Outcome, watermark: Option<DateTime<Utc>>, rows_written: usize }`; `Outcome { Success, Failed(String) }`; `Ledger { path: PathBuf }` with `append(&RunRecord)` and `last_success(source: &str) -> Option<RunRecord>` (append-only JSONL, mirrors the first course's response log).
**Test names:** `append_then_last_success_reads_back`, `last_success_ignores_failures`, `watermark_advances_across_runs`.
**Quiz:** `quizzes/03-authentication.toml`.

---

## Phase 3 — The Paginated Arc: TheSpaceDevs ◻︎ planned  → course **Part IV**

Installs idempotent content-addressed writes + retry/backoff.

### Task 3.1: Retry with backoff
**Files:** Create `crates/etl-sources/src/retry.rs`.
**Interfaces — Produces:** `async fn with_retry<F, Fut, T>(policy: RetryPolicy, op: F) -> Result<T, EtlError>` retrying only when `EtlError::is_transient()`, honoring `retry_after()`, capped exponential backoff. `RetryPolicy { max_attempts, base_delay, max_delay }`.
**Test names:** `retries_transient_then_succeeds`, `gives_up_on_permanent_immediately`, `honors_retry_after` (paused time).

### Task 3.2: Content-addressed, idempotent writes
**Files:** Create `crates/etl-core/src/content.rs`; re-export.
**Interfaces — Produces:** `pub fn content_id(kind: &str, canonical: &str) -> String` (sha256 hex of `kind` + canonical bytes); `IdempotentSink { inner: JsonlSink, seen: HashSet<String> }` impl `Sink` (skips rows whose `content_id` was already written; returns count of *new* rows).
**Given:** a `Conjunction`'s id = `content_id("conjunction", "{primary}-{secondary}-{tca_rfc3339}")`.
**Test names:** `same_row_hashes_stable`, `rerun_writes_zero_new_rows`, `distinct_rows_all_written`.

### Task 3.3: The TheSpaceDevs source (pagination)
**Files:** Create `crates/etl-sources/src/spacedevs.rs`.
**Interfaces — Produces:** `SpaceDevsSource { base_url }` impl `Source` following `next` cursors until exhausted; nested JSON ⇒ `Row::Launch` (incl. the prose `description`).
**Test names (wiremock):** `follows_next_until_null`, `429_triggers_backoff_then_succeeds`, `parses_nested_launch_json`.
**Quiz:** `quizzes/04-pagination.toml`.

---

## Phase 4 — The Orchestration Arc (payoff) ◻︎ planned  → course **Part V**

Adds `crates/etl-orchestrate` (depends on `etl-core` ONLY) and the `panoptes-etl` bin.

### Task 4.1: Typed task graph + topological execution
**Files:** Create `crates/etl-orchestrate/Cargo.toml` (dep: `etl-core` only), `crates/etl-orchestrate/src/lib.rs`, `.../graph.rs`.
**Interfaces — Produces:** `TaskId(String)`; `Dag { nodes, edges }` with `add_task`, `add_dep`, `topological_order() -> Result<Vec<TaskId>, EtlError>` (cycle ⇒ `EtlError::Parse("cycle: …")`).
**Test names:** `linear_chain_orders_correctly`, `diamond_orders_deps_first`, `cycle_is_rejected`.
**Constraint check:** `grep` the crate's `Cargo.toml` to prove no `etl-sources` dep.

### Task 4.2: The executor (retries + watermark-driven incremental)
**Files:** Create `crates/etl-orchestrate/src/executor.rs`.
**Interfaces — Consumes:** `dyn Source`, `dyn Transform`, `dyn Sink`, `Ledger`, `with_retry`. **Produces:** `Pipeline { source, transform, sink }`; `Executor { dag, ledger }` with `run() -> Result<RunReport, EtlError>` executing pipelines in topological order, retrying transient failures, recording a `RunRecord` per node, and skipping work already past the ledger watermark.
**Test names:** `runs_pipelines_in_dep_order`, `records_a_run_per_node`, `incremental_skips_up_to_date_source` (stub sources; no network).

### Task 4.3: The `panoptes-etl` CLI
**Files:** Create `crates/panoptes-etl/Cargo.toml` (dep: all three + `clap`, `anyhow`, `tokio`), `crates/panoptes-etl/src/main.rs`, `.../cli.rs`.
**Interfaces — Produces:** `clap` subcommands `run` / `backfill` / `status`; the ONLY place `CelestrakSource`/`SpaceTrackSource`/`SpaceDevsSource` are named and wired into the DAG.
**Test names:** `cli_parses_run_subcommand`, `status_reports_last_watermark` (integration test over a temp ledger).

### Task 4.4: Capstone (reader drives) — the NASA source
**Files (reader creates):** `crates/etl-sources/src/nasa.rs`.
**Given only (no implementation shown):** API key from `NASA_API_KEY` as a query param; emits `Row`s; wired into the CLI like the others. Proves the `Source` trait held with zero new concepts.
**Quiz:** `quizzes/05-orchestration.toml`.

---

## Phase 5 — The Retrieval Arc (RAG) ◻︎ planned  → course **Part VI**

### Task 5.1: The `Embedder` trait + content-addressed embeddings
**Files:** Create `crates/etl-core/src/embed.rs`; `crates/etl-sources/src/voyage.rs`.
**Interfaces — Produces:** `#[async_trait] Embedder: Send+Sync { async fn embed(&self, texts: &[String]) -> Result<Vec<Vec<f32>>, EtlError>; }`; `VoyageEmbedder { base_url, api_key }` impl (mocked in tests). Embeddings keyed by `content_id("embed", text)` so they are never recomputed.
**Test names (wiremock):** `embeds_batch_of_texts`, `cached_text_not_re_embedded`.

### Task 5.2: The embed load target + brute-force retriever
**Files:** Create `crates/etl-core/src/vector.rs`.
**Interfaces — Produces:** `EmbedSink<E: Embedder>` writing `{id, text, vector}` JSONL over prose fields only; `pub fn cosine(a: &[f32], b: &[f32]) -> f32`; `VectorIndex { rows }` with `top_k(query: &[f32], k: usize) -> Vec<(String, f32)>`.
**Test names:** `cosine_of_identical_is_one`, `top_k_ranks_by_similarity`, `embed_sink_only_touches_prose`.

### Task 5.3: Wire embedding as a DAG node
**Files:** Modify `crates/panoptes-etl/src/cli.rs` (add `embed` subcommand); wire an embedding `Pipeline` into the DAG downstream of the sources.
**Test names:** `embed_node_runs_after_sources`.
**Quiz:** `quizzes/06-retrieval.toml`.

---

## Phase 6 — Course authoring & verification ◻︎ planned  → course **Parts I–VII + appendices**

For each Part, derive from the verified code: concept pages (verbose, verified toy examples, the trap in real compiler text, quiz) and build pages (spec + test names + the loop, no implementation). Then:

- `book.toml` (mdbook-quiz, `cache-answers`, theme navy, quiz-extra.css); `src/SUMMARY.md`; `.github/workflows/deploy.yml` (pin `mdbook 0.4.52`, `mdbook-quiz 0.4.0`, cache by version); `.gitignore` (`book`, `src/quiz/`); `.claude/launch.json` (`mdbook serve --port 3000`).
- Appendices: Task-to-Chapter Map, The Answer Key (this verified plan as a chapter, anchors checked against built HTML), Workspace Scaffold, Reference Books.
- **Verification gate before push:** every example run in a scratch crate (incl. `compile_fail` with exact error codes); `mdbook build` exit 0 (validates Tracing quizzes via real rustc); every `{{#quiz}}` resolves once; answer-key anchors match generated ids; deploy watched to green (`gh run watch --exit-status`); curl changed live pages for a distinctive string.
- **Push to `panoptes_learning_etl` when complete and green.**

---

## Self-review (against the spec)

- **Spec coverage:** every spec section maps to a phase — four seams (Tasks 0.5, 2.3, 3.2 + arcs), sources in order (1.2, 2.2, 3.3, 4.4), keystone `Conjunction` (0.4, lands via 2.2), orchestrator payoff (Phase 4), RAG (Phase 5), file contract (1.3), graduation shelf (Part VII, design-only). ✓
- **Type consistency:** `Row`, `EtlError`, `RawRecord`, `Source/Transform/Sink`, `NoradId`, `content_id`, `Ledger`/`RunRecord` names are used identically across phases. ✓
- **Dependency-inversion constraint** is an explicit test/grep in Task 4.1. ✓
- **No-live-network** holds: every source task specifies wiremock; time-based tests use paused tokio time. ✓
