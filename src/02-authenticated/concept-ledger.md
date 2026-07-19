# Concept: The Run Ledger

> **Kind:** Concept. **Seam introduced this arc:** the run ledger — an append-only record of every pipeline run, and the single artifact the orchestrator's incrementality, retry, and backfill all read from later.

## The idea

So far a pipeline run is amnesiac: it fetches, transforms, writes, and forgets. Run it again tomorrow and it has no idea it ran today — no idea whether today *succeeded*, how far it got, or whether it should even bother. Every capability the orchestrator adds two arcs from now needs the pipeline to remember its own history:

- **Incrementality** — "only fetch what is new" — needs to know the **watermark** the last successful run reached.
- **Retry / resume** — "did the last run finish?" — needs to know the **outcome** of the last run.
- **Backfill** — "fill the gaps" — needs to know which runs are **missing or failed**.

All three read from one artifact: a **run ledger**. Each time a source runs, you append one record — when it started, when it finished, whether it succeeded, the high-water mark it reached, and how many rows it wrote. The ledger is the pipeline's memory, and this chapter builds it before anything consumes it, because you cannot be incremental until you can remember.

Two properties make the ledger trustworthy, and they are the whole lesson:

1. **It is append-only.** A run *adds* a line; it never rewrites or deletes an earlier one. The file is a complete, ordered history — an audit log, not a mutable "current state" you can corrupt with a bad write.
2. **A failed run does not advance the watermark.** The watermark that matters is the one from the last *successful* run. A run that blew up halfway must not be allowed to claim it made progress, or the next incremental run would skip data the failed run never actually fetched.

## A runnable toy: the backup log

Here is both properties on a domain that is plainly not orbital data — a nightly backup job. Each run records the outcome and the watermark (here, a day-of-year, for legibility). The log is a `Vec` standing in for the append-only file; `last_success` scans it and returns the most recent *successful* run, skipping failures. Pure `std`, runs on the playground:

```rust
/// How a run ended. A failed run carries a reason; a successful one does not.
#[derive(Debug, Clone, PartialEq)]
enum Outcome {
    Success,
    Failed(String),
}

/// One line of the ledger: a single run of a single job. `watermark` is the
/// high-water mark the run reached — here, a day-of-year for legibility.
#[derive(Debug, Clone)]
struct Run {
    job: String,
    outcome: Outcome,
    watermark: u32,
}

/// An append-only log. In the real crate this is JSONL on disk; here it is a
/// `Vec` so the example runs anywhere. The rule is the same: `append` only ever
/// adds; it never rewrites a prior entry.
struct Ledger {
    runs: Vec<Run>,
}

impl Ledger {
    fn new() -> Self {
        Self { runs: vec![] }
    }

    fn append(&mut self, run: Run) {
        self.runs.push(run);
    }

    /// The most recent *successful* run for a job — failures are skipped.
    fn last_success(&self, job: &str) -> Option<&Run> {
        self.runs
            .iter()
            .filter(|r| r.job == job && r.outcome == Outcome::Success)
            .last()
    }
}

fn main() {
    let mut ledger = Ledger::new();

    // A good run advances the watermark to day 200.
    ledger.append(Run { job: "backup".into(), outcome: Outcome::Success, watermark: 200 });

    // The next run fails at day 201. It is recorded — but it must NOT count
    // as the last success, so it must not advance the watermark.
    ledger.append(Run {
        job: "backup".into(),
        outcome: Outcome::Failed("disk full".into()),
        watermark: 201,
    });

    let wm = ledger.last_success("backup").map(|r| r.watermark);
    println!("after a failure, watermark = {wm:?}");

    // A later good run at day 202 does advance it.
    ledger.append(Run { job: "backup".into(), outcome: Outcome::Success, watermark: 202 });
    let wm = ledger.last_success("backup").map(|r| r.watermark);
    println!("after a success, watermark = {wm:?}");

    // A job that never ran has no watermark at all.
    println!("unknown job            = {:?}", ledger.last_success("archive").map(|r| r.watermark));
}
```

```text
after a failure, watermark = Some(200)
after a success, watermark = Some(202)
unknown job            = None
```

The three lines of output are the ledger's whole contract:

- **After the failed run, the watermark is still `200`** — not `201`. The failure *was* appended (the history is complete and honest), but `last_success` filters it out, so the failed run's optimistic watermark never counts. This is the second property in action: a failure records itself without claiming progress.
- **After the next success, the watermark advances to `202`.** Because `last_success` takes the *last* matching record, successive good runs move the watermark forward. Incrementality is "start from `last_success().watermark`."
- **An unknown job returns `None`** — the correct answer for "we have never successfully run this," which the orchestrator reads as "there is no watermark yet; do a full pull."

## The append-only discipline, enforced by the type

Notice the toy's `Ledger` exposes exactly two operations: `append` and read (`last_success`). There is no `update`, no `delete`, no `set`. That is deliberate — the append-only guarantee is baked into the *interface*, not left to a convention you might violate later. The history can only grow.

The `Outcome` comparison inside `last_success` — `r.outcome == Outcome::Success` — is why `Outcome` derives `PartialEq`. Drop that derive and the filter stops compiling, which is a useful thing to have seen once:

```rust,compile_fail
// last_success must ask each record: was this a success?
enum Outcome { Success, Failed }

fn is_success(o: &Outcome) -> bool {
    *o == Outcome::Success // compares two Outcomes — needs PartialEq
}

fn main() {
    println!("{}", is_success(&Outcome::Success));
}
```

That fails to compile with **`error[E0369]: binary operation == cannot be applied to type Outcome`** — the compiler even suggests `#[derive(PartialEq)]`. The lesson is small but concrete: the ledger's failure-filtering rule depends on being able to *compare* outcomes, so `Outcome` must be `PartialEq`, exactly as it is in the shipped type.

## The mapping onto the shipped `Ledger`

The toy is `crates/etl-core/src/ledger.rs` with the storage made durable and the fields fleshed out. The shipped code is `rust,ignore` — it uses `chrono`, `serde`, and `tokio::fs`:

```rust,ignore
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(tag = "outcome", rename_all = "snake_case")]
pub enum Outcome {
    Success,
    Failed { reason: String },
}

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct RunRecord {
    pub source: String,
    pub started_at: DateTime<Utc>,
    pub finished_at: DateTime<Utc>,
    pub outcome: Outcome,
    pub watermark: Option<DateTime<Utc>>,
    pub rows_written: usize,
}

impl Ledger {
    pub async fn append(&self, rec: &RunRecord) -> Result<(), EtlError> {
        let mut file = OpenOptions::new()
            .create(true)
            .append(true) // <- append, never truncate: history only grows
            .open(&self.path)
            .await?;
        let line = serde_json::to_string(rec).map_err(|e| EtlError::Parse(e.to_string()))?;
        file.write_all(line.as_bytes()).await?;
        file.write_all(b"\n").await?; // one record per line — JSONL
        Ok(())
    }

    pub async fn last_success(&self, source: &str) -> Result<Option<RunRecord>, EtlError> {
        // ... read the file, or Ok(None) if it does not exist yet ...
        let mut last = None;
        for line in text.lines().filter(|l| !l.trim().is_empty()) {
            let rec: RunRecord = serde_json::from_str(line)?;
            if rec.source == source && rec.outcome == Outcome::Success {
                last = Some(rec); // last write for this source wins
            }
        }
        Ok(last)
    }
}
```

Field for field: the toy's `job` ↔ `source`; `outcome: Outcome` is the same idea, now `#[serde(tag = "outcome")]` so each JSONL line is self-describing (just like `Row` is tagged by `kind`); `watermark: u32` ↔ `watermark: Option<DateTime<Utc>>` (a real timestamp, `None` before the first success); and the shipped record adds `started_at`, `finished_at`, and `rows_written` for the audit trail. The `Vec` becomes an **append-only JSONL file** opened with `.create(true).append(true)` — the same file contract the JSONL sink uses, applied to run history instead of rows. And `last_success` is identical in spirit: scan every line, keep the last one whose source matches *and* whose outcome is `Success`.

<div class="callout">
<span class="callout-label">The payoff — this one file seeds three features</span>
The ledger looks modest — append a line, scan for the last success — but it is the seed of everything the orchestrator does in Part V. Incremental runs read <code>last_success(source).watermark</code> to know where to resume. The executor writes a <code>RunRecord</code> per node so a later run can see what already succeeded. Backfill reads the gaps between successful runs. You are not building those features yet; you are building the memory they all depend on, and getting the "a failure never advances the watermark" rule right <em>now</em> is what makes them correct later.
</div>

<div class="callout trap">
<span class="callout-label">Append, not overwrite — the same rule as the sink</span>
If <code>append</code> ever opened the file with <code>.truncate(true)</code> instead of <code>.append(true)</code>, each run would erase the entire run history and the ledger would only ever "remember" the current run — silently breaking incrementality and backfill, with no error to warn you. Append is the primitive; the ledger's value <em>is</em> its accumulated history. This is the exact discipline the JSONL sink follows for rows; the ledger applies it to runs.
</div>

## Questions to lock

1. Name the three orchestrator features the ledger seeds, and say which field of `RunRecord` each one reads.
2. Why must a *failed* run be recorded but *not* advance the watermark? What breaks in the next run if a failure were allowed to advance it?
3. What does `last_success` return for a source that has never succeeded, and how should the orchestrator interpret that?
4. Why is the ledger append-only? What would `.truncate(true)` instead of `.append(true)` silently destroy?
5. Why does `Outcome` need to derive `PartialEq`? Which line of `last_success` depends on it, and what error do you get without it?

The graded quiz for this arc is at the end of the part, on the [Concept-Check: Authentication](./quiz.md) page.
