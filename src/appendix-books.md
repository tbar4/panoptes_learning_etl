# Appendix: Reference Books

Where to go deeper on the ideas this course leans on. None is required reading —
the course is self-contained — but each is the canonical source for a mechanism
we used.

## Rust language & libraries

- **The Rust Programming Language** ("the book") — ownership, traits, error
  handling. The baseline this sequel assumes.
- **Rust for Rustaceans** (Jon Gjengset) — trait objects and object safety (why
  `dyn Source` needs `async-trait`), and API design for library crates like
  `etl-core`.
- **Rust Atomics and Locks** (Mara Bos) — the concurrency model behind the
  `TokenBucket`'s `Mutex` and the shared-state reasoning in the executor.
- **Asynchronous Programming in Rust** (the async book) — futures, `await`, and
  the runtime that `tokio` provides.
- **The `serde` and `tokio` docs** — the two libraries most load-bearing here;
  the `serde` data model explains the `Row` tagged-enum wire format.

## Domain: space situational awareness

- **CelesTrak** documentation (Dr. T.S. Kelso) — the authoritative reference for
  the TLE format, the GP element API, and orbit classification.
- **Space-Track.org API docs** — the CDM (conjunction data message) schema and
  the query language, plus the rate-limit etiquette the `TokenBucket` respects.
- **The Launch Library 2 API** (TheSpaceDevs) — the paginated launch schema.
- **Revisiting Spacetrack Report #3** (Vallado et al.) — the definitive treatment
  of TLE mean elements and SGP4, if you take the propagation graduation step.

## Data engineering & orchestration

- **The Dagster and Apache Airflow docs** — the systems this ETL's four seams are
  designed to grow toward. Read them to see what "add a scheduler" actually
  entails once the ledger and DAG are in place.
