# The Graduation Path

> **Kind:** Wrap-up.

Everything in this chapter is **design, not code**. Nothing here was built in the course — and that is the entire point. This course climbed to "pipeline + seams + a real mini-orchestrator" and stopped there on purpose. A production data platform has four more things this one does not: a real storage-and-search engine, a production rate limiter, orbit propagation, and a scheduler that runs on its own. This chapter names each one, points at the *exact* seam it snaps onto, and explains why leaving it out was a design decision rather than an omission.

The claim the whole course has been making is testable right here: if the four seams are real, each of these four additions is a *node you add*, not a rewrite you suffer. Let's check.

## First, the four seams — because they are the argument

Every graduation below snaps onto one of these. Keep them in view:

1. **The `Source` trait** — extraction behind one `async fn`. New source, new impl.
2. **The E/T/L split** — extract, transform, load as three swappable stages joined by typed boundaries (`RawRecord` → `Row` → `Sink`).
3. **The run ledger** — an append-only `RunRecord` per run, carrying the watermark.
4. **Idempotent, content-addressed writes** — a `sha256` of the row's canonical form gives a stable id, so re-running is always safe.

A seam is only real if you can replace what sits behind it without touching what sits in front. That is what "additive, not a rewrite" means, and it is what each of the following tests.

## DuckDB + VSS over Parquet — snaps onto the `Sink` / `VectorIndex` boundary

**What it replaces.** Two things at once: the `JsonlSink` load target, and the brute-force `VectorIndex`. Today the ETL writes newline-delimited JSON and searches vectors by scanning every row with `cosine`. The graduation writes columnar **Parquet** and searches with DuckDB's **VSS** (vector similarity search) extension — indexed nearest-neighbour instead of a linear scan, plus real SQL over the numeric entities.

**The seam it snaps onto.** Loading was never `println!`-to-a-file; it was always a trait:

```rust,ignore
#[async_trait]
pub trait Sink: Send + Sync {
    async fn load(&self, rows: &[Row]) -> Result<usize, EtlError>;
}
```

A `DuckDbSink` is just another `impl Sink`. Nothing upstream of the sink — no source, no transform, no orchestrator — knows or cares whether the bytes land as JSONL or Parquet. On the retrieval side, `VectorIndex` already exposes a narrow read interface — `from_jsonl` to build, `top_k(query, k)` to search. A DuckDB-backed index that keeps the same `top_k` shape drops in behind that boundary the same way. The `EmbedSink` that pays for embeddings stays exactly as it is; only where the vectors *land* and how they are *searched* changes.

**Why it was deferred.** The corpus is a few thousand prose fields, not millions. At that scale brute-force cosine is not a compromise — it is the *correct* tool, and reaching for a vector database would teach the wrong instinct: that you need heavy machinery before you have the data volume that justifies it. Building the honest version first, behind a trait, is what makes the upgrade cheap when the volume finally arrives.

## `governor` — snaps onto the `TokenBucket` constructor argument

**What it replaces.** The hand-rolled `TokenBucket` limiter — `TokenBucket::new(capacity, refill_per_sec)`, with an `acquire()` that sleeps until a token is free. `governor` is the production-grade equivalent: battle-tested, more limiting algorithms, no rate-limiter bugs of your own to maintain.

**The seam it snaps onto.** The limiter was never hard-wired into the source. It is a constructor parameter:

```rust,ignore
SpaceTrackSource::new(SPACETRACK_BASE, creds, TokenBucket::new(30, 0.5))
```

Anything that gates a call before the fetch can take that slot. Wrap `governor` to satisfy the same "await before you fetch" shape and pass it where `TokenBucket` goes. The source's `extract` logic does not move a line. This is the smallest graduation of the four precisely because the injection point was designed in from the start.

**Why it was deferred.** Rate limiting *is* the Part III lesson. You cannot understand what `governor` does for you until you have felt the problem it solves — Space-Track banning an impolite client — and built the token bucket that fixes it by hand. Hand-rolling the thing that is the lesson, and reaching for the library for the thing that is not, is the rule the whole stack follows.

## `sgp4` orbit propagation — snaps onto the parsed `OrbitalElements`

**What it adds.** `sgp4` takes a set of orbital elements at an epoch and propagates them forward to compute where an object *will be* at a future time. With it, you could compute conjunctions yourself — screen every pair of objects for close approaches — instead of ingesting conjunctions that arrive pre-computed.

**The seam it snaps onto.** This one is different: it is **not** the `Source`/`Transform` boundary. Propagation *consumes* the domain model the transform already produces. Your `OrbitalElements` struct is the input `sgp4` wants:

```rust,ignore
pub struct OrbitalElements {
    pub epoch: DateTime<Utc>,
    pub inclination_deg: f64,
    pub raan_deg: f64,
    pub eccentricity: f64,
    pub arg_perigee_deg: f64,
    pub mean_anomaly_deg: f64,
    pub mean_motion_rev_per_day: f64,
}
```

Parsing a TLE into these seven numbers is ETL. Turning these numbers into a future position is *computation on top of* the parsed model — a downstream consumer of the file contract, not a new stage inside a pipeline. That distinction is why it lives on the graduation shelf rather than in the transform arc.

**Why it was deferred.** Propagation is a rabbit hole — reference frames, drag models, numerical stability — and none of it is the ETL lesson. Parsing and normalizing the elements is. And critically, the course did not *need* it: real conjunctions arrive pre-computed from Space-Track / SOCRATES, so the keystone entity was available without propagating anything. Building your own conjunction screening is a genuinely new capability, not a missing piece of this one.

## A real scheduler / daemon — snaps onto the `Executor` and the ledger watermark

**What it adds.** Today the pipeline runs when you type `panoptes-etl run`. The graduation is a long-running process that drives the `Executor` on a cron or interval — waking up, running the DAG, sleeping, repeating — with no human at the keyboard. This is the option-C ceiling: the thing that finally makes the project look like Airflow.

**The seams it snaps onto — two of them.** The executor already exposes the one method a scheduler needs to call:

```rust,ignore
let report = executor.run().await?;   // runs the whole DAG once
```

A daemon is a loop around that call plus a clock. But the loop would be blind without the ledger. Incremental runs already read it: `Ledger::last_success` finds the last good run, and `Pipeline::is_fresh` skips a source whose last success is inside its `refresh_after` window. Every `RunRecord` carries a `watermark`, and the executor stamps it on success:

```rust,ignore
pub struct RunRecord {
    pub source: String,
    pub finished_at: DateTime<Utc>,
    pub outcome: Outcome,
    pub watermark: Option<DateTime<Utc>>,
    pub rows_written: usize,
}
```

The scheduler reads that watermark to decide what to fetch and when to fetch it. Retries with backoff are already in the executor, reused from Part IV. So the daemon adds a *driver* on top of finished machinery — a `tokio` timer that calls `run()` — without reaching inside any pipeline. Airflow's value proposition, stripped to its core, is a good ledger plus a scheduler that reads it; you built the ledger, and the scheduler is the loop it was always waiting for.

**Why it was deferred.** Two reasons, both honest. First, a background scheduler is genuinely painful to test-drive: time, concurrency, and flakiness fight the clean one-test-per-chapter rhythm this course depends on, and a shaky capstone would undercut the discipline the whole book teaches. Second, it earns its own course. The right place for a scheduler is a sequel that can give concurrency and time the careful treatment they deserve — not a rushed final chapter here.

## The discipline was the subject

Look back at the four graduations. Each one replaces or extends something *behind a seam*, and in every case the thing in front of the seam does not move:

- DuckDB slots behind `Sink` and `VectorIndex` — sources and orchestrator untouched.
- `governor` slots into a constructor argument — `extract` untouched.
- `sgp4` consumes `OrbitalElements` — the transform untouched.
- The scheduler loops over `Executor::run` and reads the ledger — the pipelines untouched.

That is the proof the seams were real. You did not build these four things, and *because* you built the simplest pipeline that already had the joints an orchestrator needs, you do not have to rewrite anything to add them later. Leaving a seam for a future you cannot yet see — and having that future arrive as a new node instead of a demolition — was the whole subject of the course. The graduation shelf is not a list of what is missing. It is the evidence that the discipline paid off.
