# Author Report — Appendix: The Answer Key (Full Task Plan)

Authored 2026-07-19. Overwrote the stub at `src/appendix-answer-key.md`.

Source of truth: the verified `panoptes_etl` workspace. Every file under
`crates/*/src/`, plus the workspace `Cargo.toml` and each crate `Cargo.toml`,
was read and reproduced **verbatim** (comments, `#[cfg(test)]` modules, and all).
No code was invented or altered. Verified live before writing:

- `cargo test` → 65 pass (etl-core 29, etl-sources 26, etl-orchestrate 7, panoptes-etl 3).
- `cargo tree -p etl-orchestrate` → among workspace crates only `etl-core` appears
  (plus `chrono` and external transitive deps); `etl-sources`/`panoptes-etl` absent.

## Structure

Organized by course arc in dependency order: Foundations → Extraction →
Authenticated → Paginated → Orchestration → Retrieval. Each source file (and each
manifest) has a short HTML `<h3 id="task-...">` heading, the complete file in a
fenced ```rust``` (or ```toml```) block, and a one-line list of its test names.
Ends with a "Verifying the whole thing" section (cargo test / clippy / tree +
per-crate counts). No `{{#quiz}}` includes; `mdbook build` was NOT run.

## Anchor ids created (30 total)

These are unique, stable, lowercase-kebab. Build pages can deep-link as
`appendix-answer-key.html#<id>`.

### Part I — Foundations
- `task-workspace-cargo` — Cargo.toml (workspace manifest)
- `task-etl-core-cargo` — crates/etl-core/Cargo.toml
- `task-etl-core-lib` — crates/etl-core/src/lib.rs
- `task-etl-core-error` — crates/etl-core/src/error.rs
- `task-etl-core-ids` — crates/etl-core/src/ids.rs

### Part II — Extraction (CelesTrak)
- `task-etl-core-domain` — crates/etl-core/src/domain.rs
- `task-etl-core-pipeline` — crates/etl-core/src/pipeline.rs
- `task-etl-core-sink-jsonl` — crates/etl-core/src/sink_jsonl.rs
- `task-etl-sources-cargo` — crates/etl-sources/Cargo.toml
- `task-etl-sources-lib` — crates/etl-sources/src/lib.rs
- `task-etl-sources-http` — crates/etl-sources/src/http.rs
- `task-tle-parser` — crates/etl-sources/src/tle.rs
- `task-celestrak-source` — crates/etl-sources/src/celestrak.rs

### Part III — Authenticated (Space-Track)
- `task-etl-core-ledger` — crates/etl-core/src/ledger.rs
- `task-token-bucket` — crates/etl-sources/src/limiter.rs
- `task-spacetrack-source` — crates/etl-sources/src/spacetrack.rs

### Part IV — Paginated (TheSpaceDevs)
- `task-etl-core-retry` — crates/etl-core/src/retry.rs
- `task-etl-core-content` — crates/etl-core/src/content.rs
- `task-spacedevs-source` — crates/etl-sources/src/spacedevs.rs

### Part V — Orchestration
- `task-etl-orchestrate-cargo` — crates/etl-orchestrate/Cargo.toml
- `task-etl-orchestrate-lib` — crates/etl-orchestrate/src/lib.rs
- `task-dag-graph` — crates/etl-orchestrate/src/graph.rs
- `task-executor` — crates/etl-orchestrate/src/executor.rs
- `task-panoptes-etl-cargo` — crates/panoptes-etl/Cargo.toml
- `task-cli` — crates/panoptes-etl/src/cli.rs
- `task-main` — crates/panoptes-etl/src/main.rs

### Part VI — Retrieval (RAG)
- `task-embedder-trait` — crates/etl-core/src/embed.rs
- `task-vector-index` — crates/etl-core/src/vector.rs
- `task-voyage-embedder` — crates/etl-sources/src/voyage.rs

## Notes / decisions

- `task-tle-parser` matches the slug in the task instructions exactly.
- Part I Foundations has no build code of its own in the course, but the two
  prerequisites it introduces (`error.rs`, `ids.rs`) plus the workspace + crate
  scaffolding are placed here; `domain.rs`/`pipeline.rs`/`sink_jsonl.rs` (built in
  the Part II "Build: Workspace + Domain + Source Trait" chapter) are placed under
  Extraction, matching SUMMARY.md and appendix-task-map.md.
- `lib.rs` for both library crates is shown in its FINAL form (all modules listed),
  with a note that the reader grows it arc by arc — the answer key shows the end state.
- The NASA capstone (Part V) has no reference file; the commented block inside
  `main.rs`'s `build_executor` is its entire answer, called out with a blockquote.
- `spacedevs.rs` test `http_error_is_network_error` is the actual shipped name
  (a 500 → Network), reproduced as-is.

## Not done / for the central build
- `mdbook build` was not run (per instructions).
- No `{{#quiz}}` directives were added (this is a reference-listing chapter).
