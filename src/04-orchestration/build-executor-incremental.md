# Build: Retries, Watermarks & Incremental Runs

> **Maps to:** Task 4.2. **Kind:** Build — spec and test names only. No implementation.

## Objective

Build the **executor**: the piece that owns the DAG, the pipelines, and the ledger, and runs everything in topological order. It adds the three behaviors that turn a bare topological sort into a real scheduler:

1. **Retries** — each pipeline's extract goes through `with_retry`, so a transient failure is retried per the policy before it counts as a failure.
2. **Freshness-skip (incremental)** — a source whose last successful run in the ledger is newer than its refresh window is *skipped*, not re-fetched.
3. **Failure-gating** — a node whose prerequisite failed (or was itself blocked) does not run at all. The DAG edges gate execution, not merely order it.

This is where `etl-orchestrate` finally touches the `etl-core` traits — `dyn Source`, `dyn Transform`, `dyn Sink`, `Ledger`, `with_retry` — and still names no concrete source.

## Scaffold

**Create:** `crates/etl-orchestrate/src/executor.rs`. **Modify:** `src/lib.rs` to `pub mod executor;` and re-export `Executor`, `Pipeline`, `RunReport`.

**Dependencies:** none new — `etl-core` and `chrono` were declared when you created the crate; `async-trait`, `tokio`, and `pretty_assertions` are already dev-dependencies for the stub sources the tests need.

**Expected result:** `cargo test -p etl-orchestrate executor` → the executor tests green (`#[tokio::test]`, stub sources, no network).

## The spec (givens)

**`Pipeline`** — one extract → transform → load chain, over trait objects:

- Fields: `source: Box<dyn Source>`, `transform: Box<dyn Transform>`, `sink: Box<dyn Sink>`, `policy: RetryPolicy`, `refresh_after: Option<Duration>`.
- `Pipeline::new(source, transform, sink)` — defaults `policy` to `RetryPolicy::default()` and `refresh_after` to `None`. Callers set `refresh_after` afterward when they want incremental behavior.
- `run_once(&self) -> Result<usize, EtlError>` — extract **through `with_retry(self.policy, || self.source.extract())`**, transform every raw record (flat-mapping each into rows), then `sink.load(&rows).await` once. Returns the row count the sink reports.
- A private `is_fresh(&self, ledger, id) -> Result<bool, EtlError>` — `false` if `refresh_after` is `None` *or* the ledger has no prior success for this id; otherwise `Utc::now() - last.finished_at < refresh`.

**`RunReport`** — the outcome of a whole DAG run. Derive `Debug, Default`. Three `Vec`s:

- `records: Vec<RunRecord>` — one per node that actually ran (success or failure).
- `skipped: Vec<TaskId>` — nodes skipped because they were still fresh.
- `blocked: Vec<TaskId>` — nodes not run because a prerequisite failed or was itself blocked.

**`Executor`** — owns `dag: Dag`, `pipelines: HashMap<TaskId, Pipeline>`, `ledger: Ledger`.

- `new(ledger)`; `add_pipeline(&mut self, id: &str, pipeline) -> TaskId` (registers the task in the DAG and stores the pipeline); `add_dep(&mut self, task, depends_on)` (delegates to the DAG).
- `run(&self) -> Result<RunReport, EtlError>` — the heart of the arc:
  1. Compute `dag.topological_order()?` (a cycle short-circuits the whole run with `EtlError::Parse`).
  2. Walk the order, carrying a `bad: HashSet<TaskId>` of failed-or-blocked ids.
  3. For each node: if **any** prerequisite is in `bad`, add this node to `bad` and to `report.blocked`, and `continue` — so the block **propagates** transitively down the graph.
  4. Else if `is_fresh`, push to `report.skipped` and `continue`.
  5. Else run `run_once`. On `Ok`, record `Outcome::Success` with a `watermark: Some(finished_at)` and the row count. On `Err`, add the id to `bad` and record `Outcome::Failed { reason }`.
  6. Append the `RunRecord` to the ledger and to `report.records`.

## Concepts exercised

- Scheduling over trait objects (`Box<dyn Source/Transform/Sink>`) — the dependency-inversion payoff in code.
- The retry middleware (`with_retry` + `RetryPolicy`) wired into the extract step.
- Watermark-driven incremental runs: reading `last_success().finished_at` from the ledger.
- Transitive failure propagation via a `bad` set threaded through the topological walk.

## The build loop (you drive)

1. **`runs_pipelines_in_dependency_order`.** Two stub sources `a` and `b`, `add_dep(b, a)`; each stub pushes its name to a shared log on extract. Assert the log is `["a", "b"]` and `report.records.len() == 2`.
2. **`records_a_run_per_node`.** Two independent stubs; assert one `RunRecord` per node and every outcome `Success`.
3. **`failed_upstream_blocks_dependent`.** A `FailingSource` at `a`, a stub at `b`, `add_dep(b, a)`. **Predict first:** how many records, how many blocked, and how many times is `b`'s source extracted? Assert `records.len() == 1`, `report.blocked == [b]`, and `b`'s extract call-count is `0`.
4. **`incremental_skips_up_to_date_source`.** Seed the ledger with a very recent success for the source; set `refresh_after = Some(Duration::hours(6))`. Assert `report.skipped.len() == 1`, `report.records` empty, and the source's extract call-count is `0`.
5. **Run green, commit.**

<div class="callout predict">
<span class="callout-label">Predict first — blocked is not the same as failed</span>
In <code>failed_upstream_blocks_dependent</code>, <code>a</code> <em>fails</em> (it ran and errored — it gets a <code>RunRecord</code> with <code>Outcome::Failed</code>). <code>b</code> is <em>blocked</em> (it never ran — no record, an entry in <code>blocked</code>). Before running, be able to say which list each id lands in and why <code>b</code>'s <code>extract</code> is never called. The counter proving zero extracts is the assertion that the gate actually stopped the work, not just relabeled it.
</div>

<div class="callout trap">
<span class="callout-label">The block must propagate, or a diamond leaks</span>
Step 3 adds a blocked node to <code>bad</code>, not only to <code>report.blocked</code>. That is deliberate: in a diamond, if <code>a</code> fails, both <code>b</code> and <code>c</code> block — and <code>d</code>, which depends on <code>b</code> and <code>c</code>, must block too even though <code>a</code> is not its <em>direct</em> prerequisite. Because <code>b</code> and <code>c</code> were added to <code>bad</code> when they blocked, <code>d</code> sees a bad prerequisite and blocks in turn. Forget to re-add blocked nodes to <code>bad</code> and <code>d</code> would run against missing upstream data.
</div>

## Done when

`cargo test -p etl-orchestrate` is green across graph and executor. You now have a scheduler that retries transient failures, skips fresh sources, and refuses to run a node whose upstream is broken — all over trait objects, with no concrete source in sight. The only thing missing is a front door: the CLI that names the real sources and calls `run()`.
