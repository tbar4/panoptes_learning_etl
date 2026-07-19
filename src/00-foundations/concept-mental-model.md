# Concept: The ETL Mental Model & the Four Seams

> **Kind:** Concept. No new crate — this chapter installs the architectural vocabulary the whole course leans on. Read it slowly; every later arc refers back to it by name.

## The idea

**ETL** is three verbs: **Extract** data from somewhere you do not control, **Transform** it into a shape you do, and **Load** it into a place you can read later. That is the entire job description of `panoptes_etl`: pull orbital data from CelesTrak, Space-Track, and TheSpaceDevs; normalize it into typed rows; write the JSONL files the Panoptes eval harness reads.

You could build that as a pile of one-off scripts — a `fetch_celestrak.rs`, a `fetch_spacetrack.rs`, each with its own hard-coded URL and its own ad-hoc parsing. It would work today and it would never grow. The moment you want retries, or a schedule, or "only fetch what changed since yesterday," you are rewriting.

You could also swing the other way and build a full orchestrator on day one — a scheduler, a daemon, a DAG engine, a run database — and never ship anything.

This course takes the third path, and it is the whole thesis:

> **Build the simplest pipeline that already has the joints an orchestrator needs.**

Simple now. But with the *seams* pre-cut, so that "add a scheduler," "add backfill," "add retries," and "add a fourth source" each become a later chapter that **adds a node**, never a rewrite. The seams are the point. Everything below is a description of the four we cut, why each one earns its place, which arc installs it, and where it finally gets *spent* in Parts V–VI.

## Why "seam" is the right word

A seam is a place where the design lets you change one thing without disturbing the others. A tailor leaves seam allowance so a garment can be let out later; you are leaving *design* allowance so the pipeline can be let out later. A seam costs you a little discipline up front — a trait instead of a free function, a typed boundary instead of passing raw strings around — and pays you back the first time a requirement lands that would otherwise have been a rewrite.

The four seams below are not arbitrary. Each one corresponds to a specific real-world complication every ETL eventually hits: *new source*, *stage failure isolation*, *knowing what already ran*, and *safe re-runs*. Install the seam before the complication arrives and the complication becomes cheap.

## Seam 1 — the `Source` trait: one shape for every source

The first complication is simply *there is more than one source, and there will be more later*. If every source has a bespoke signature, the orchestrator has to know each one by name. So we make every source wear the same uniform: an object-safe async trait.

```rust,ignore
// from crates/etl-core/src/pipeline.rs
#[async_trait]
pub trait Source: Send + Sync {
    fn name(&self) -> &str;
    async fn extract(&self) -> Result<Vec<RawRecord>, EtlError>;
}
```

> This sketch needs `async_trait`, `chrono`, `thiserror`, and `serde`, so it is `rust,ignore` — the real thing lives in `etl-core` and is built in Part II. To run a version yourself, `cargo new` a lib crate and `cargo add async-trait chrono thiserror serde --features serde/derive`.

Two design choices carry the seam. `Send + Sync` means a `Source` can be moved to and shared across tokio tasks — the orchestrator will need that. And because the trait is **object-safe**, you can hold a `Box<dyn Source>`: the orchestrator can keep a list of sources and call `extract()` on each without knowing whether it is talking to CelesTrak or TheSpaceDevs. A new source is a new `impl Source` and nothing else in the system changes.

**Installed:** Part II (CelesTrak is the first `impl Source`).
**Spent:** Part V, where the DAG executor schedules a `Vec<Box<dyn Source>>` and depends on *none* of them by name.

## Seam 2 — E / T / L as three separable stages

The second complication is *partial failure*. A fetch can succeed while parsing fails; parsing can succeed while the disk write fails. If extract, transform, and load are tangled into one function, you cannot retry the fetch without re-parsing, or swap the sink without touching the parser. So we split them into three traits with a **typed boundary** between each:

```rust,ignore
// Extract produces RawRecord (bytes off the wire, unparsed).
// Transform turns &RawRecord into Vec<Row> (normalized, typed). Pure & sync.
// Load takes &[Row] and returns how many it wrote.

pub struct RawRecord {
    pub source: String,
    pub payload: String,
    pub fetched_at: DateTime<Utc>,
}

pub trait Transform: Send + Sync {
    fn transform(&self, raw: &RawRecord) -> Result<Vec<Row>, EtlError>;
}

#[async_trait]
pub trait Sink: Send + Sync {
    async fn load(&self, rows: &[Row]) -> Result<usize, EtlError>;
}
```

The boundary types are the seam. `RawRecord` is the contract between E and T: everything upstream deals in raw bytes, everything downstream deals in a `Row`. Notice that **`Transform` is pure and synchronous** — no network, no clock, no disk. That is deliberate: parsing is the part you most want to unit-test exhaustively, and a pure function is trivial to test. `Source` and `Sink` are async because they touch the outside world; `Transform` is not because it must not.

`Row` itself is a single tagged enum over every entity a source can emit, so one `Sink` can accept the output of any pipeline:

```rust,ignore
#[derive(Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum Row {
    SpaceObject(SpaceObject),
    Conjunction(Conjunction),
    Launch(Launch),
}
```

**Installed:** Part II (the CelesTrak pipeline runs E → T → L end to end).
**Spent:** Part V, where a `Pipeline { source, transform, sink }` becomes a single node the executor runs.

## Seam 3 — the run ledger: knowing what already ran

The third complication is *time and incrementality*. A real ETL does not re-fetch the entire history of the universe on every run; it fetches what changed since last time. To do that it must **remember what it did** — when each source last ran, up to what watermark, and whether it succeeded. That memory is the **run ledger**: an append-only record, one line per run.

```rust,ignore
pub struct RunRecord {
    pub source: String,
    pub started_at: DateTime<Utc>,
    pub finished_at: DateTime<Utc>,
    pub outcome: Outcome,                     // Success | Failed(String)
    pub watermark: Option<DateTime<Utc>>,     // "I am caught up to here"
    pub rows_written: usize,
}
```

Append-only matters: you never mutate history, you only add to it, which means the ledger is itself an audit trail and is safe to read while it is being written. The `watermark` is what makes the *next* run incremental — it answers "where did I leave off?".

**Installed:** Part III (Space-Track, the first source with enough volume and auth cost to make re-fetching everything unacceptable).
**Spent:** Part V, where the executor writes one `RunRecord` per node and reads the last watermark to **skip work that is already up to date**.

## Seam 4 — idempotent, content-addressed writes: safe re-runs

The fourth complication is the ugliest: *runs fail halfway and get retried*. If a run writes 500 rows, dies, and is re-run, a naive sink writes those 500 rows **again** — now you have duplicates polluting the data the eval harness trusts. The fix is to make writes **idempotent**: running twice produces the same result as running once.

We get that by **content-addressing** each row — deriving its id from a hash of its own content rather than from an auto-incrementing counter or the wall clock:

```rust,ignore
// A row's identity is a hash of what it *is*, not when it arrived.
pub fn content_id(kind: &str, canonical: &str) -> String { /* sha256 hex */ }

// A Conjunction's id: content_id("conjunction", "{primary}-{secondary}-{tca}")
```

Because the id is a pure function of the content, the *same* conjunction always hashes to the *same* id. An idempotent sink keeps a set of ids it has already written and skips any it has seen, so a re-run writes **zero new rows**. This is what makes "just run it again" a safe response to any failure — which is exactly the property retries need.

**Installed:** Part IV (TheSpaceDevs, where pagination + retries make duplicate-on-retry a live hazard).
**Spent:** Part IV directly (re-running the paginated fetch is safe), and again in Part VI, where embeddings are keyed by `content_id("embed", text)` so prose is never re-embedded — an expensive API call saved for free by the same seam.

## The payoff, held in view

Here is why the discipline is worth it. When you reach Part V, the orchestrator is almost anticlimactic to write, because every joint it needs is already there:

- it holds sources as `Box<dyn Source>` — **Seam 1**;
- it runs each as a `Pipeline` of separable E/T/L stages — **Seam 2**;
- it records and reads watermarks to run incrementally — **Seam 3**;
- and it retries freely because re-runs are safe — **Seam 4**.

The DAG executor *reuses* all four and depends on none of them by name. `etl-orchestrate` will not even be allowed to mention a concrete source — the dependency graph forbids it, which is the proof that the seams actually decoupled the design rather than just looking like they did. That inversion is the destination. Everything in Parts I–IV is laying seam allowance so that Part V is an assembly, not a rewrite.

<div class="callout">
<span class="callout-label">The one sentence to keep</span>
We are not building an orchestrator. We are building the four seams an orchestrator would need, in the simplest pipeline that can hold them — so that the orchestrator, when it arrives, is a node that snaps onto joints already cut.
</div>

## Questions to lock

1. Name the three verbs of ETL and, for each, say what type crosses the boundary out of it in this pipeline (`RawRecord`, `Row`, a count).
2. What are the four seams? For each, state the one real-world complication it exists to absorb.
3. Why is `Transform` a plain synchronous trait when `Source` and `Sink` are async? What does that buy you at test time?
4. What does "object-safe" let the orchestrator do with a `Source`, and why does it need that?
5. Why does content-addressing (id from a hash of the content) make re-runs safe, when an auto-increment id would not?
6. For each seam, which arc *installs* it and where does it get *spent*?
