# Build: The Ledger + the `Conjunction` Keystone Entity

> **Maps to:** Task 2.3 (the run ledger) + the `Conjunction` domain type (defined in Part I's domain build, first emitted for real this arc). **Kind:** Build — spec and test names only. No implementation.

## Objective

Build the **run ledger** — the append-only record of every pipeline run that Part V's orchestrator reads for incrementality, retry, and backfill — and pin down the **`Conjunction`** keystone entity that the Space-Track transform you just built emits. The ledger is new code (`ledger.rs`); `Conjunction` already exists in `domain.rs` from Part I, but this is the arc where it stops being a definition and starts being data, so we revisit it as the entity it was designed to be.

By the end, `cargo test -p etl-core` covers the ledger's three behaviors: append-and-read-back, failures are ignored by `last_success`, and the watermark advances across successful runs.

## Part A — the `Conjunction` keystone (recap, `domain.rs`)

`Conjunction` is the reason the whole project exists: a predicted close approach between two tracked objects. It is the "keystone" because it is the entity that *references other entities* — a conjunction is meaningless without the two `NoradId`s it points at. Its shipped shape (from Part I):

```text
Conjunction {
    id: String,                    // composite key now; content-addressed in Part IV
    tca: DateTime<Utc>,            // time of closest approach
    miss_distance_km: f64,         // closest separation, in kilometers
    collision_probability: f64,    // 0.0 when the source omits it
    primary: NoradId,              // the two objects, by catalog number...
    secondary: NoradId,            // ...and this is where id-safety earns its keep
}
```

**Why the keystone makes the newtype pay off.** `primary` and `secondary` are both `NoradId`, not bare `u32`. If they were plain integers, swapping the two arguments when constructing a `Conjunction` would compile silently and mislabel a collision-avoidance record — the exact bug the newtype chapter warned about. Because both are `NoradId`, the *values* cannot be confused with any other kind of id (a launch id, a payload count); the ordering of the two `NoradId`s within a conjunction is then a discipline the transform gets right by construction. The keystone entity is where Part I's id-safety seam earns its keep.

You do not write new code in this part — `Conjunction` was built and tested in `domain.rs` already. The point is to recognize what the Space-Track transform produced: the first real `Conjunction` values, keyed by the composite `id` the transform formats.

## Part B — the run ledger (`ledger.rs`)

The ledger concept page is the reference; this is the interface to build.

**Create:** `crates/etl-core/src/ledger.rs`; declare the module and re-export `Ledger`, `RunRecord`, and `Outcome` from `lib.rs`.

**Dependencies:** none new. The append-only file uses `tokio::fs::OpenOptions` and `tokio::io::AsyncWriteExt` (both under tokio's `full` features), and `serde`/`serde_json` for the JSONL encoding — all already declared in `etl-core`.

**`Outcome`** — how a run ended. Derives `Debug, Clone, PartialEq, Serialize, Deserialize`; tagged `#[serde(tag = "outcome", rename_all = "snake_case")]` so each variant is self-describing on the wire (the same tagging idea as `Row`'s `kind`):

```text
Outcome::Success
Outcome::Failed { reason: String }
```

`PartialEq` is not optional here — `last_success` compares `rec.outcome == Outcome::Success`, and without the derive that comparison will not compile (`error[E0369]`).

**`RunRecord`** — one line of the ledger. Derives `Debug, Clone, PartialEq, Serialize, Deserialize`:

```text
RunRecord {
    source: String,                    // which source ran
    started_at: DateTime<Utc>,
    finished_at: DateTime<Utc>,
    outcome: Outcome,
    watermark: Option<DateTime<Utc>>,  // high-water mark reached; None before first success
    rows_written: usize,
}
```

**`Ledger`** — one field, `path: PathBuf`; constructor `new(path: impl Into<PathBuf>)`. Two async methods:

- `append(&self, rec: &RunRecord) -> Result<(), EtlError>` — open `path` with **create + append** (`OpenOptions::new().create(true).append(true)`), serialize the record with `serde_json::to_string` (failure → `EtlError::Parse`), and write the line followed by a `'\n'`. One record per line — JSONL. The `?` on the file ops relies on `EtlError`'s `#[from] std::io::Error` from Part I.
- `last_success(&self, source: &str) -> Result<Option<RunRecord>, EtlError>` — read the file to a string; if it does **not exist yet**, return `Ok(None)` (match on `ErrorKind::NotFound`), not an error. Parse each non-empty line into a `RunRecord`; keep the **last** one whose `source` matches *and* whose `outcome == Outcome::Success`. Return that, or `None`.

## Test names to hit

- `append_then_last_success_reads_back` — append one successful `RunRecord` for `"celestrak"`; assert `last_success("celestrak")` returns `Some(rec)` equal to what you wrote, and `last_success("spacetrack")` returns `None`. Proves the round-trip and the source filter. (Use a temp path — `std::env::temp_dir()` plus the process id — and `remove_file` first for a clean slate, exactly as the JSONL sink tests do.)
- `last_success_ignores_failures` — append a `Success` at watermark `2026-07-18…`, then a `Failed { reason }` at watermark `2026-07-19…`. Assert `last_success` returns the run whose watermark is still `2026-07-18…`. **This is the load-bearing test:** the failed run was recorded but must not advance the watermark.
- `watermark_advances_across_successful_runs` — append two `Success` runs at `2026-07-18…` then `2026-07-19…`; assert `last_success` returns the `2026-07-19…` watermark. Successive successes move the mark forward.

<div class="callout predict">
<span class="callout-label">Predict before you run</span>
Before writing <code>last_success_ignores_failures</code>, say which watermark you expect back — the failed run's (<code>07-19</code>) or the last success's (<code>07-18</code>). If your <code>last_success</code> filtered on <code>source</code> but forgot the <code>outcome == Success</code> clause, which one would it return? That one missing clause is the difference between a correct incremental pull and one that skips data a failed run never fetched.
</div>

<div class="callout trap">
<span class="callout-label">Create + append, never truncate</span>
The whole value of the ledger is its accumulated history. Open the file with <code>.create(true).append(true)</code>. If you reach for <code>.write(true)</code> without <code>.append(true)</code>, or <code>.truncate(true)</code>, each <code>append</code> overwrites from the start of the file and the ledger silently forgets every prior run — no error, no panic, just an amnesiac pipeline. This is the same append-only discipline the JSONL sink follows for rows, applied to run history.
</div>

## The build loop (you drive)

1. **`Outcome` and `RunRecord` first.** Define the types with their derives and serde tags. If a comparison later fails to compile, you dropped `PartialEq` on `Outcome`.
2. **`append` + `append_then_last_success_reads_back`.** Write the round-trip test, implement `append` (create+append, one line) and a first `last_success`. **Predict** the returned record and the `None` for the absent source before running.
3. **`last_success_ignores_failures`.** Predict the watermark (see the callout), then confirm your filter includes the `outcome == Success` clause.
4. **`watermark_advances_across_successful_runs`.** Two successes; assert the later watermark wins because `last_success` takes the *last* match.
5. **Run green, commit.**

## Done when

`cargo test -p etl-core` is green across the three ledger tests (alongside the domain, pipeline, and sink tests from earlier arcs). The pipeline can now *remember its own runs* — the memory that Part IV's retry loop and Part V's incremental executor will both read from. Every seam the orchestrator needs is now in place: the E/T/L traits, the file contract, the error taxonomy, and — as of this chapter — the run ledger.
