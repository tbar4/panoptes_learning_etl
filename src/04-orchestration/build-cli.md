# Build: The `panoptes-etl` CLI

> **Maps to:** Task 4.3. **Kind:** Build — spec and test names only. No implementation.

## Objective

Build the `panoptes-etl` binary: the front door. This is the **only crate in the workspace that names a concrete source** — it imports `CelestrakSource`, `SpaceTrackSource`, `SpaceDevsSource` from `etl-sources`, wires each into a `Pipeline`, hands the pipelines to the `Executor`, and exposes it all as a `clap` subcommand CLI: `run`, `backfill`, `embed`, `status`. Everything below it — the orchestrator, the traits — stays source-agnostic; the binary is where the abstract machinery meets the concrete world.

## Scaffold

**Create the crate:** `crates/panoptes-etl/Cargo.toml`, `src/main.rs`, `src/cli.rs`. Add `"crates/panoptes-etl"` to the workspace `members`.

**`Cargo.toml`** — this crate depends on *all three* layers plus the CLI machinery:

```toml
[dependencies]
etl-core = { path = "../etl-core" }
etl-orchestrate = { path = "../etl-orchestrate" }
etl-sources = { path = "../etl-sources" }   # the ONLY crate that may name these
clap = { workspace = true, features = ["derive"] }
tokio = { workspace = true }
anyhow = { workspace = true }
chrono = { workspace = true }
serde_json = { workspace = true }
```

Split the crate in two: `cli.rs` holds the **testable** parts (the `clap` types and the pure status-report renderer); `main.rs` holds the `#[tokio::main]` wiring that names concrete sources and cannot be unit-tested without the network. Test the seam, not the socket.

**Expected result:** `cargo test -p panoptes-etl` → the CLI-parse and status tests green; `panoptes-etl --help` lists the four subcommands.

## The spec (givens)

**`cli.rs` — the parseable surface:**

- `Cli` (derive `Parser`): a `#[command(subcommand)] command: Command`, plus two global flags — `--out-dir: PathBuf` (default `"data"`) and `--ledger: PathBuf` (default `"data/runs.jsonl"`).
- `Command` (derive `Subcommand, Debug, PartialEq, Eq`): four variants —
  - `Run` — run every pipeline once, skipping sources still fresh.
  - `Backfill` — re-run every pipeline, ignoring freshness.
  - `Embed` — embed prose fields of already-loaded launches (Part VI).
  - `Status` — show the last successful watermark per source.
- `pub const SOURCES: &[&str] = &["celestrak", "spacetrack", "spacedevs"];` — the known sources, in DAG order.
- `status_report(ledger: &Ledger, sources: &[&str]) -> Result<String, EtlError>` — for each source, look up `ledger.last_success(source)`; render either `"{source:<12} last success {watermark} ({rows} rows)"` or `"{source:<12} never run"`; join with newlines. Pure over its ledger argument, so it is trivially testable.

**`main.rs` — the wiring (given, not built here):**

- `#[tokio::main] async fn main() -> anyhow::Result<()>`: parse the `Cli`, create the out-dir and `Ledger`, then `match cli.command`.
- `Run` / `Backfill` both build the executor and call `run()`; the difference is a single boolean — `incremental = cli.command == Command::Run` — which sets a `Duration::hours(6)` refresh window (or `None` for backfill). Print each record, each skipped, each blocked node.
- `build_executor` is where the three `CelestrakSource` / `SpaceTrackSource` / `SpaceDevsSource` are constructed with their base URLs and wired into `Pipeline`s with an idempotent JSONL sink. Space-Track is added *only when its credentials are present* in the environment.

## Concepts exercised

- `clap` derive with a `#[command(subcommand)]` enum and global flags that may appear before the subcommand.
- Separating the testable CLI surface (`cli.rs`) from the un-testable I/O wiring (`main.rs`).
- The single-boolean-toggle pattern: `run` vs `backfill` differ only in the freshness window.
- The binary as the sole owner of concrete-source knowledge — the top of the dependency-inversion story.

## The build loop (you drive)

1. **`cli_parses_run_subcommand`.** `Cli::try_parse_from(["panoptes-etl", "run"])`; assert `cli.command == Command::Run`. Define `Cli` / `Command` to pass it.
2. **`cli_parses_flags_before_subcommand`.** `["panoptes-etl", "--out-dir", "/tmp/x", "status"]`; assert the subcommand is `Status` and `out_dir` parsed. **Predict:** does `clap` accept a global flag *before* the subcommand by default?
3. **`status_reports_last_watermark`.** Seed a temp ledger with one success for `celestrak` (42 rows, a known watermark); call `status_report(&ledger, SOURCES)`; assert the string contains `celestrak`, the watermark date, `"42 rows"`, and that an unseeded source (`spacedevs`) reads `"never run"`.
4. **Wire `main.rs`** — construct the three sources in `build_executor`, dispatch each subcommand. Run `panoptes-etl --help` and confirm all four verbs appear.
5. **Run green, commit.**

<div class="callout">
<span class="callout-label">Why `status` is the easiest verb to test</span>
<code>status</code> touches only the ledger — no network, no sources. That is exactly why <code>status_report</code> is a pure function taking <code>&amp;Ledger</code> and a source list: you can seed a temp ledger, call it, and assert on the returned <code>String</code> with no mock server at all. The verbs that <em>do</em> hit the network (<code>run</code>, <code>backfill</code>) get their coverage one layer down, in the executor's stub-source tests. Test each behavior at the layer where it is cheapest to isolate.
</div>

## Done when

`cargo test -p panoptes-etl` is green, `panoptes-etl --help` lists `run`, `backfill`, `embed`, and `status`, and `panoptes-etl status` against a fresh ledger prints `never run` for all three sources. The pipeline is now a single command with discoverable verbs — and the capstone will prove it absorbs a fourth source without the orchestrator noticing.
