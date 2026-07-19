# Design: The Panoptes ETL Course

**Date:** 2026-07-18
**Status:** Approved (brainstorm complete) — ready for implementation planning
**Deliverable:** A self-paced, compiler-verified mdBook course that teaches Rust by building
`panoptes_etl`, a space-data ETL tool, test by test. Same methodology and voice as the
first course (*Building Panoptes: A Rust Harness, Test by Test*, live at
https://eval.rustycloud.org).

This document specifies **both** the software project the course builds **and** the course
that teaches it. The course is *generated from* a fully worked, compiling task plan; this
spec is the input to writing that plan.

---

## 1. Purpose & context

### 1.1 What the ETL is for

The Panoptes eval harness (`panoptes-core` + `panoptes-gen`) generates decision-scenario
*vignettes* from a parameter grid and measures how an LLM makes each call. The one live
family, `ca_geo`, has params `attribution_confidence`, `time_pressure` (Hours/Days),
`reversibility`, and `info_request` — i.e. **collision-avoidance in geostationary orbit**:
an operator deciding whether to maneuver, under time pressure, with uncertain attribution
of who caused the conjunction, where the maneuver may or may not be reversible.

`panoptes_etl` supplies the **real-world ground truth** those vignettes are built on — real
satellites, real orbital elements, and above all **real conjunctions** — so scenarios are
factual and regenerable rather than hand-waved. A real Conjunction Data Message *is* a
collision-avoidance decision waiting to be posed to a model.

### 1.2 The connection to Panoptes (the file contract)

Decision: **local files, Panoptes reads them.** The ETL writes normalized JSONL to disk;
`panoptes-gen` reads those files when building vignettes. This keeps the two crates
decoupled by a file contract, exactly like Panoptes' existing `scenarios/` directory. No
shared database and no in-process crate dependency. (DuckDB-over-Parquet is a named
graduation, not part of the taught course.)

### 1.3 The keystone entity

`Conjunction` is the keystone. It carries TCA (time of closest approach), miss distance,
collision probability, and the two object ids — a raw collision-avoidance scenario. The
other entities (`SpaceObject`, `OrbitalElements`, `Launch`) exist to enrich and attribute
conjunctions.

---

## 2. Goals & non-goals

### 2.1 The scope ceiling

The course climbs to **"pipeline + seams + a real mini-orchestrator"** (option B of three
considered). It builds working pipelines for real sources, installs the seams an
orchestrator needs, and then — in the payoff arc — builds a real DAG executor that reuses
those seams. It does **not** build a scheduler/daemon, queryable run-history UI, or Airflow
full clone (option C); those are sketched design-only in the wrap-up as the graduation path.

Rationale: the discipline-then-vindication structure is the best teaching arc, and a
background scheduler is genuinely painful to TDD (time, concurrency, flakiness) in a way
that would undercut the clean test-per-chapter rhythm the methodology depends on. The
scheduler is a natural sequel course.

### 2.2 The four seams (non-negotiable; the whole point)

The craft being taught is **building the simplest thing that already has the joints an
orchestrator needs**. Four seams cover ~90% of the "grows into Airflow" ambition with no
scheduler code:

1. **A `Source` trait** — every source is a uniform `async fn fetch → RawRecord`. New
   source = new impl; nothing else changes.
2. **Extract / Transform / Load as three separable stages** with a typed boundary between
   each (raw bytes → normalized domain rows → sink). This is a small DAG per source;
   making it a bigger DAG later is additive.
3. **A run ledger** — append-only records of *what ran, when, at what watermark,
   success/failure*. This one artifact is what later becomes lineage, incremental pulls,
   retries, and backfill. Airflow's value proposition is essentially a good ledger plus a
   scheduler reading it.
4. **Idempotent, content-addressed writes** (sha256 of the normalized row → stable id) so
   re-running is safe — the precondition for any retry/scheduling story.

Build those four and "add a scheduler", "add a DAG of dependent tasks", "add backfill",
"add retries with backoff" all become later chapters that add a node, never a rewrite.

### 2.3 Non-goals (deliberate)

- **No `sgp4` / orbit propagation in the taught course.** Propagation is a rabbit hole;
  parsing + normalizing the elements is the ETL lesson, and real Conjunctions arrive
  pre-computed from Space-Track / SOCRATES. `sgp4` is named on the graduation shelf.
- **No vector database.** The corpus is thousands of objects; brute-force cosine is
  correct, and pretending otherwise would teach the wrong instinct. DuckDB VSS is the named
  graduation.
- **No scheduler/daemon** (see 2.1).

---

## 3. Audience & prerequisites

**Audience: sequel to the Panoptes course** (option A of three). The reader is assumed to
have finished the first course and therefore already owns: ownership/borrowing, `serde` +
derive, traits, `async`/`await`/`tokio`, `wiremock`, `clap`, and the
write-failing-test → predict → run → implement → green → commit loop.

Consequence: **Foundations stays lean.** It re-orients to the ETL mental model and teaches
only what is genuinely new to *this* project — deep error handling (partial failure and
retries), newtypes for id-safety, and the seam architecture. A single "if you skipped the
first course, here's the assumed toolkit" pointer chapter keeps a cold reader from being
stranded.

---

## 4. Architecture

### 4.1 Crate layout — a workspace that grows one crate per arc

The repo starts as the single `panoptes_etl` package (edition 2024); Part II converts it to
a workspace, exactly like the first course's opening build converted to a workspace.

- **`etl-core`** — the seams live here. Domain model (`SpaceObject`, `OrbitalElements`,
  `Conjunction`, `Launch`), the `Source` + `Sink` traits, the `Embedder` trait, the error
  taxonomy, id newtypes, the run ledger, content-addressing, and the generic cosine
  retriever. **Depends on nothing else in the workspace.**
- **`etl-sources`** — concrete impls: the four sources, the hand-rolled TLE parser, the
  token-bucket limiter, retry middleware, the concrete embedder. **Depends on `etl-core`.**
- **`etl-orchestrate`** — the DAG executor. **Depends on `etl-core` only — never on
  `etl-sources`.** That constraint *is* the dependency-inversion lesson: the executor
  schedules `dyn Source` and has no idea NASA exists.
- **`panoptes-etl`** (bin) — the CLI (`run` / `backfill` / `embed` / `status`). **The only
  crate that names concrete sources** — the wiring happens here.

**The load-bearing teaching move:** `etl-orchestrate` compiling *without* a dependency on
`etl-sources` is the proof the seam is real. It is a chapter beat, not a footnote.

Dependency direction (must hold): `etl-core` ← `etl-sources` ← `panoptes-etl`;
`etl-core` ← `etl-orchestrate` ← `panoptes-etl`. `etl-orchestrate` must not depend on
`etl-sources`.

### 4.2 Domain model (the file contract)

Normalized entities written as JSONL, read by `panoptes-gen`:

- **`SpaceObject`** — `norad_cat_id`, name, international designator, object type, operator,
  orbit class (GEO / LEO / MEO / HEO), plus current `OrbitalElements`.
- **`OrbitalElements`** — epoch, inclination, RAAN, eccentricity, argument of perigee, mean
  anomaly, mean motion, and derived apogee / perigee / period.
- **`Conjunction`** (keystone) — TCA, miss distance, collision probability, the two object
  ids.
- **`Launch`** — id, name, date, provider, payloads.

Retrieval is split by data shape: **structured retrieval** (SQL/filter) over the numeric
entities; **semantic retrieval** (embeddings) only over the *prose* fields (launch/mission
descriptions, agency blurbs, advisories). The ETL produces both artifacts. Never
semantic-search the numbers.

### 4.3 The sources (order is pedagogical)

Each source introduces exactly one new real-world complication and installs one seam.

| # | Source | Auth | New complication | Seam installed |
|---|--------|------|------------------|----------------|
| 1 | **CelesTrak** | none (open GET) | Parsing the **TLE** fixed-width text format — columns, checksum digit, epoch decode | `Source` trait + E/T/L split |
| 2 | **Space-Track** | login → session cookie | **Auth + rate limiting** (token bucket; Space-Track bans impolite clients) + secrets from env | The run ledger |
| 3 | **TheSpaceDevs** (Launch Library 2) | none, throttled | **Pagination** (follow `next`) + **429 / Retry-After backoff** + deeply nested JSON | Idempotent content-addressed writes |
| 4 | **NASA** `api.nasa.gov` | API key (query param) | *Nothing new* — different shape, same machinery | **Capstone: reader adds it unassisted**, proving the `Source` trait held |

All live APIs are mocked with `wiremock` in tests, exactly as the first course mocked the
Anthropic API. Nothing in the test suite hits a live network.

### 4.4 The orchestrator (Part V payoff)

- A **typed DAG**: nodes are tasks (extract/transform/load steps, or whole source
  pipelines), edges are dependencies; execution is topological.
- The executor schedules `dyn Source` / trait objects and depends only on `etl-core` — it
  cannot see concrete sources (dependency inversion, enforced by crate boundaries).
- **Retries with backoff** reuse the Part IV retry policy.
- **Watermark-driven incremental runs** read the ledger to fetch only what is new.
- The **NASA capstone** is a "you drive, unassisted" exercise: by this point the reader has
  seen auth, rate limiting, pagination, and the trait, so a fresh source with no new
  concepts proves the abstraction earned its keep.

### 4.5 The RAG arc (Part VI)

RAG is split by stage across the boundaries already drawn:

- **Index-building (chunk → embed → write vectors) is ETL** — the "load" target is a vector
  artifact. It reuses all four seams; embeddings are the definitive content-addressed,
  idempotent, ledger-tracked workload (expensive calls you must never repeat), so the
  sha256 seam becomes load-bearing.
- **The retrieval read-interface** (embed a query, top-k cosine) is a thin read over the
  file contract — a generic helper in `etl-core`.
- **Augment + generate** is *Panoptes'* job, out of scope here — it decides whether
  retrieved facts ground a vignette or become an eval treatment arm.

An **`Embedder` trait** mirrors the first course's `ModelClient`, with a **Voyage AI** impl
as the concrete real example, mocked with `wiremock` in tests. Start **brute-force cosine**
over a JSONL of `{id, text, vector}`; DuckDB VSS is the named graduation. The orchestrator
**schedules the embedding node** — the perfect expensive-idempotent DAG node, and the
strongest proof the seams generalize beyond "download JSON".

---

## 5. The stack

Mirrors the Panoptes workspace deps wherever they overlap; hand-rolls exactly the things
that are the lesson.

| Concern | Choice | Why |
|---|---|---|
| HTTP + async | `reqwest` + `tokio` | Same as first course; assumed knowledge |
| Serialize | `serde` / `serde_json` (JSONL) | The file contract |
| CLI | `clap` (derive) | Same as first course |
| Time | `chrono` | Epochs, TCA, watermarks |
| Errors | `thiserror` (libs) + `anyhow` (bin) | The taxonomy is a Foundations chapter |
| Hashing | `sha2` | Content-addressing — already in the stack |
| Test | `wiremock` + `pretty_assertions` | Mock every live API |
| **TLE parser** | **hand-rolled** | Parsing *is* the Part II lesson |
| **Token-bucket limiter** | **hand-rolled** | Rate-limiting *is* the Part III lesson |
| **Retry / backoff** | **hand-rolled** | Backoff policy *is* the Part IV lesson |
| **DAG executor** | **hand-rolled** | The whole Part V payoff |
| **Cosine retriever** | **hand-rolled** | Brute-force-first is the honest Part VI lesson |

**Graduation shelf** (named in Part VII, not built): `sgp4` (propagation), `governor`
(production rate-limiting), `duckdb` + VSS (storage + real vector search), and a `tokio`
scheduler/daemon (the option-C ceiling). Each is framed as "here's the exact seam it snaps
onto."

---

## 6. Course structure

6 build arcs, ~34 chapters — comparable in size to the first course. Concept-before-build
throughout; one quiz per arc; `book.toml` uses the mdbook-quiz preprocessor with
`cache-answers = true`, theme `navy`, and `additional-css = ["theme/quiz-extra.css"]`.

**Intro** — what we're building, the Panoptes connection, the simple-with-seams thesis.

**Part I — Foundations** *(lean; sequel-aware)*
- How to Use This Course (+ "if you skipped the first course" pointer)
- Concept: The ETL Mental Model & the Four Seams
- Concept: Error Handling for Fallible I/O (transient vs permanent, `thiserror` enums, retry classification)
- Concept: Newtypes & Id-Safety
- Concept-Check: Foundations

**Part II — The Extraction Arc (CelesTrak)** — installs `Source` trait + E/T/L split
- Concept: The `Source` Trait & the E/T/L Boundary (toy: a "receipt fetcher")
- Concept: Parsing the TLE Format (toy: a fixed-width record mirroring TLE)
- Build: Workspace + Domain Model + the `Source` Trait
- Build: The CelesTrak Source — Extract + Transform
- Build: The JSONL Sink — Load
- Concept-Check: Extraction

**Part III — The Authenticated Arc (Space-Track)** — installs the run ledger + rate limiting
- Concept: Session Auth & Secrets from the Environment
- Concept: Rate Limiting with a Token Bucket (toy mirror)
- Concept: The Run Ledger
- Build: The Space-Track Source (login → cookie, rate-limited, wiremock-tested)
- Build: The Ledger + the `Conjunction` Keystone Entity
- Concept-Check: Authentication

**Part IV — The Paginated Arc (TheSpaceDevs)** — installs idempotent content-addressed writes + retry/backoff
- Concept: Pagination & Following `next`
- Concept: Retry with Backoff & 429 / Retry-After
- Concept: Idempotent, Content-Addressed Writes
- Build: The TheSpaceDevs Source — Pagination (the `Launch` entity)
- Build: Retry Middleware + the Idempotent Sink
- Concept-Check: Pagination

**Part V — The Orchestration Arc** *(the payoff)* — the DAG executor
- Concept: Modeling a Pipeline as a Typed DAG
- Concept: Dependency Inversion — the Executor Knows Only the Trait
- Build: The Task Graph + Topological Execution
- Build: Retries, Watermarks & Incremental Runs
- Build: The `panoptes-etl` CLI — `run` / `backfill` / `status`
- **Capstone (you drive, unassisted): Add the NASA Source**
- Concept-Check: Orchestration

**Part VI — The Retrieval Arc (RAG)** — embed load target + retriever; orchestrator schedules it
- Concept: Structured vs Semantic Retrieval
- Concept: The `Embedder` Trait & Content-Addressed Embeddings (mocked in tests)
- Build: The Embed Load Target (chunk prose → embed → vector JSONL; ledger-tracked)
- Build: The Brute-Force Retriever + Wiring it as a DAG Node
- Concept-Check: Retrieval

**Part VII — Wrap-Up**
- Where This Plugs Into Panoptes (the file contract; vignette grounding; the retrieval eval arm)
- The Graduation Path (design-only: DuckDB + VSS, `governor`, `sgp4`, the real scheduler)

**Appendices** — Task-to-Chapter Map · The Answer Key (full task plan) · Workspace Scaffold · Reference Books

### 6.1 Invariants (the course methodology)

- **One seam per arc**; no build chapter uses machinery no earlier chapter demonstrated. If
  a build needs a crate/API with no lead-in, a concept chapter precedes it.
- **Concept pages are verbose with verified examples**, use a toy domain whose shape mirrors
  the paired build one-for-one, demonstrate the trap the build will hit with real
  compiler/library error text, and end with "Questions to lock" + a quiz.
- **Build pages give the spec, not the implementation** — objective, scaffold with exact
  paths and expected test counts/names, the givens (field lists, derive stacks, wire
  formats, constants, CLI flags, error formats) each deep-linked to the answer key, concepts
  exercised, the build loop, and "Done when".
- The course is generated from a **fully worked task plan whose code compiles** — verified
  by execution before any chapter is derived (the first course shipped three real bugs
  caught only by running the plan).

---

## 7. Testing & verification

- Every code example is compiled and run in a scratch cargo project — including
  expected-failure examples (assert the exact error code) and captured warning/help text.
- All live APIs mocked with `wiremock`; no test hits a live network.
- `mdbook build` must exit 0 (this validates Tracing quizzes + quiz schema; Tracing
  questions are compiled and executed by real rustc at build time).
- Answer-key anchor links must match generated heading ids.
- Deploy pins tools (`mdbook 0.4.52`, `mdbook-quiz 0.4.0`), caches by version, and is
  watched to green before curling live pages for a distinctive string.

---

## 8. Open questions / deferred

None blocking. Deferred by design to the graduation shelf (Part VII, not built): DuckDB +
VSS storage, `governor` rate-limiting, `sgp4` propagation, and the scheduler/daemon
(option-C ceiling) — each a potential sequel.
