# Build: The JSONL Sink — Load

> **Maps to:** Task 1.3. **Kind:** Build — spec and test names only. No implementation.

## Objective

Implement `JsonlSink` — the **L** stage, and the file contract the rest of the world reads. It appends normalized `Row` values to a file as **JSON Lines**: one JSON object per line, appended (never truncated) so a run only ever adds to what came before. This is the exact format `panoptes-gen` consumes downstream, which is why "append-only, one row per line" is a contract and not an implementation detail.

## Scaffold

**Create:** `crates/etl-core/src/sink_jsonl.rs`; declare the module and re-export `JsonlSink` from `lib.rs`.

**Dependencies:** none new. `serde`/`serde_json` and `tokio` were declared when you built `etl-core`. The async I/O uses `tokio::fs` and `tokio::io::AsyncWriteExt`, both under tokio's `full` features.

**Expected result:** `cargo test -p etl-core sink` → the sink tests pass.

## The spec (givens)

**`JsonlSink`** — one field, `path: PathBuf`; constructor `new(path: impl Into<PathBuf>)`. It implements the `Sink` trait from `pipeline.rs`:

```text
#[async_trait]
impl Sink for JsonlSink {
    async fn load(&self, rows: &[Row]) -> Result<usize, EtlError> { ... }
}
```

**Behavior `load` must have:**

- Open `path` with **create + append** (`tokio::fs::OpenOptions::new().create(true).append(true)`). Append is the whole point: a second `load` must not erase the first.
- Serialize each `Row` with `serde_json::to_string`. A serialization failure maps to `EtlError::Parse(...)`. (In practice `Row` always serializes, but the boundary stays honest.)
- Write **one line per row** — each JSON object followed by a `'\n'`. Build the whole batch into one buffer and do a single `write_all`, so a batch is one write, not N.
- Return the **count** of rows written (`rows.len()`).
- The `?` on file operations relies on `EtlError`'s `#[from] std::io::Error` (Part I) — an I/O failure surfaces as `EtlError::Io` for free.

## Test names to hit

- `writes_one_row_per_line_and_roundtrips` — write a temp path (use `std::env::temp_dir()` plus the process id so parallel test runs do not collide, and `remove_file` first for a clean slate). `load` two rows; assert the returned count is `2`; read the file back and assert it has exactly 2 lines; `serde_json::from_str` the first line and assert it equals the first row you wrote. This proves both halves of the contract at once: one row per line, and a clean round-trip through serde.
- `append_accumulates_across_loads` — call `load` twice on the same path with the same 2 rows; read the file and assert it now has **4** lines. This is the append guarantee: the second call added to the file rather than replacing it.

<div class="callout">
<span class="callout-label">Why append, not overwrite</span>
An ETL run is incremental: today's fetch adds to the historical record, it does not replace it. If <code>load</code> truncated, a re-run — or a second source writing to the same file — would silently destroy earlier rows, and the "idempotent, content-addressed writes" seam in Part IV would have nothing to build on. Append is the primitive; de-duplication comes later, as a wrapper around this sink, never as a change to it.
</div>

## The build loop (you drive)

1. **Write `writes_one_row_per_line_and_roundtrips` first.** You need sample rows — reuse a couple of `Row::Launch` values. **Predict** the returned count and the line count before you run.
2. **Implement `load`** with create+append and the single buffered `write_all`.
3. **Run**, check the round-trip assertion — if the deserialized row differs, your serialize/deserialize derives on the domain types disagree (revisit the `roundtrips` test from the domain build).
4. **Write `append_accumulates_across_loads`.** **Predict:** if you accidentally used `.truncate(true)` or `.write(true)` without `.append(true)`, how many lines would this test see? Then confirm append gives 4.
5. **Run green, commit.**

## Done when

`cargo test -p etl-core sink` is green. You now have all three stages — E (`CelestrakSource`), T (`CelestrakTransform`), L (`JsonlSink`) — each satisfying its trait from `etl-core`. A complete pipeline is now just "call `extract`, feed each `RawRecord` through `transform`, hand the rows to `load`." The next arcs make that wiring incremental, authenticated, retried, and eventually scheduled — but the seams are all in place as of this chapter.
