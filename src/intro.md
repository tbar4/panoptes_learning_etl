# Introduction

This is a course, not a manual — and it is a **sequel**. By the end you will have built `panoptes_etl`, a real space-data ETL that pulls from CelesTrak, Space-Track, and TheSpaceDevs, normalizes the results, and writes the files the Panoptes eval harness reads. More importantly, you will understand *why* every piece is shaped the way it is — and specifically, how to build the simplest thing that already has the joints an orchestrator needs.

## What we are building, and why

Panoptes measures how large language models make hard decisions. Its live scenario family, `ca_geo`, is **collision avoidance in geostationary orbit**: an operator deciding whether to maneuver, under time pressure, with uncertain attribution of who caused the conjunction. Those scenarios are only defensible if they are grounded in *real* orbital data — real satellites, real orbital elements, and above all real **conjunctions**. Supplying that ground truth is this ETL's job.

So the keystone is the **conjunction**: a real Conjunction Data Message *is* a collision-avoidance decision waiting to be posed to a model. Everything else — space objects, orbital elements, launches — exists to enrich and attribute conjunctions.

## The tension this course is about

You could build this as a pile of one-off scripts. It would work, and it would never grow. You could instead build a full orchestrator — a scheduler, a daemon, a DAG engine — on day one, and never finish. This course takes the third path, the one worth teaching: **build the simplest pipeline that already has the four seams an orchestrator needs**, so that "add a scheduler," "add backfill," "add retries" all become later chapters that *add a node*, never a rewrite.

The four seams, installed one per arc:

1. A **`Source` trait** — every source is a uniform `async fn`. A new source is a new impl; nothing else changes.
2. **Extract / Transform / Load as three separable stages**, with a typed boundary between each.
3. A **run ledger** — an append-only record of what ran, when, at what watermark, success or failure.
4. **Idempotent, content-addressed writes** — so re-running is always safe.

Build those four, and the final arc's payoff writes itself: a DAG executor that drops the sources you already wrote into a graph and runs them — reusing every seam, and depending on none of them by name.

## How the arcs are sequenced

Each source-arc introduces exactly one new real-world complication and installs one seam:

- **Part I, Foundations** — the ETL mental model, error handling for fallible I/O, and id-safety. Short, because this is a sequel: ownership, `serde`, traits, `async`/`tokio`, `wiremock`, and `clap` are assumed.
- **Part II, Extraction (CelesTrak)** — an open GET and the TLE text format. Installs the `Source` trait + the E/T/L split.
- **Part III, Authenticated (Space-Track)** — login, cookies, secrets, and rate limiting. Installs the run ledger.
- **Part IV, Paginated (TheSpaceDevs)** — cursor pagination and retry/backoff. Installs idempotent content-addressed writes.
- **Part V, Orchestration** — the payoff: a typed DAG executor over the seams, plus a capstone where you add a fourth source unassisted.
- **Part VI, Retrieval (RAG)** — embed the prose fields into a vector index; the orchestrator schedules the embedding node.
- **Part VII, Wrap-Up** — how this plugs into Panoptes, and the graduation path (DuckDB, a real scheduler) sketched against the seams.

## A note on where you are starting from

If you finished the first Panoptes course, you already think in ownership, traits, and `async`. What is new here is not the language — it is the *shape of ETL*: partial failure, retries, idempotency, and the discipline of leaving seams for a future you cannot yet see. That discipline is the whole subject.

<div class="callout">
<span class="callout-label">The rhythm of each build chapter</span>
Before we run any code, I will ask you to <em>predict</em> what the compiler or the test runner will say. The prediction is where the understanding forms. A wrong prediction followed by the real message is the single most efficient way to learn what the types are actually enforcing. Build chapters give you the spec and the test names — never the implementation. You drive.
</div>

## What stays out (on purpose)

We stop at "pipeline + seams + a real mini-orchestrator." We do **not** build a long-running scheduler daemon, a queryable run-history UI, or orbit propagation (`sgp4`), and we do **not** reach for a vector database. Each of those is named in the graduation path with the exact seam it snaps onto — because the point of the seams is that they make those additions cheap later, not that we spend this course on them.
