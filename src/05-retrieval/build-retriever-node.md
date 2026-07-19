# Build: The Brute-Force Retriever + Wiring it as a DAG Node

> **Maps to:** Task 5.2 (the search half) + 5.3. **Kind:** Build — spec and test names only. No implementation.

## Objective

Implement the *read* side of semantic retrieval — `cosine`, `EmbeddedRow`'s index, and `VectorIndex::top_k` — and then wire the embed step into the pipeline as something the orchestrator *schedules*: the `embed` subcommand today, a DAG node downstream of `spacedevs` in a fuller build. This is where the vectors written last chapter become answers to queries.

## Scaffold

**Create/extend:** `crates/etl-core/src/vector.rs` (add `cosine`, `VectorIndex`, `top_k`, `VectorIndex::from_jsonl`). **Modify:** `crates/panoptes-etl/src/cli.rs` (add the `Embed` subcommand) and `crates/panoptes-etl/src/main.rs` (the `embed_launches` wiring).

**Dependencies:** none new. `serde`/`serde_json`, `tokio`, `clap` are all present from earlier arcs.

**Expected result:** `cargo test -p etl-core cosine` / `top_k` → the index tests pass; `cargo test -p panoptes-etl` → the CLI parses the new subcommand.

## The spec (givens) — the index

- **`pub fn cosine(a: &[f32], b: &[f32]) -> f32`** — dot product over the L2 norms; **return `0.0` if either norm is zero** (never divide by zero). Identical vectors score `1.0`; orthogonal vectors score `0.0`.
- **`VectorIndex { pub rows: Vec<EmbeddedRow> }`** — `new(rows)`, `Default`, and `from_jsonl(text)` (one `EmbeddedRow` per non-blank line; a parse failure maps to `EtlError::Parse`). This is the whole "database": a `Vec`, in memory.
- **`top_k(&self, query: &[f32], k: usize) -> Vec<(String, f32)>`** — score every row's vector against `query` with `cosine`, sort **descending** by score (use `f32::total_cmp` — `f32` is not `Ord`), truncate to `k`, return `(id, score)` pairs. Highest similarity first.

<div class="callout">
<span class="callout-label">Why <code>total_cmp</code>, not <code>partial_cmp().unwrap()</code></span>
<code>f32</code> has no total order because of <code>NaN</code>, so it is not <code>Ord</code> — <code>sort_by</code> needs a total comparator. <code>partial_cmp().unwrap()</code> works right up until a <code>NaN</code> score panics the sort mid-run. <code>total_cmp</code> is a genuine total order over all <code>f32</code> bit patterns, so the sort is always well-defined. Reach for it any time you sort floats.
</div>

## The spec (givens) — wiring the step

- **`Embed` subcommand** — add an `Embed` variant to the `clap` `Command` enum with a doc comment ("Embed the prose fields of already-loaded launches into a vector index"), and a match arm in `main` that calls `embed_launches(&cli.out_dir)` and prints the count of new prose fields embedded.
- **`embed_launches(out_dir)`** — read `out_dir/launches.jsonl` into `Vec<Row>` (error out if empty — "run `panoptes-etl run` first"); read the existing `out_dir/embeddings.jsonl` for already-embedded ids; build `VoyageEmbedder::from_env("https://api.voyageai.com")`; construct `EmbedSink::with_seen(embedder, index_path, seen_ids)`; `load` the rows and return the count.
- **`read_embedded_ids(path)`** — parse each line as `EmbeddedRow`, collect the `id`s; a missing file yields an empty `Vec` (a first run has nothing embedded yet).

<div class="callout">
<span class="callout-label">Why this is "a step the orchestrator schedules," even as a subcommand</span>
The embed step <em>consumes the output of a source</em> (launches) and produces a derived artifact (vectors). That is exactly the shape of a downstream DAG node — it depends on <code>spacedevs</code> having run. Shipping it as its own subcommand keeps the dependency explicit and manual for now: you run <code>run</code>, then <code>embed</code>. Promoting it to a real node — a <code>Pipeline</code> whose "source" reads <code>launches.jsonl</code> and whose "sink" is the <code>EmbedSink</code>, added to the executor after <code>spacedevs</code> — changes no seam: the <code>Sink</code> trait already fits. The order (sources first, embed after) is the edge; the executor's topological sort is what would enforce it.
</div>

## Test names to hit

- `cosine_of_identical_is_one_orthogonal_is_zero` — assert `cosine` of a vector with itself is `1.0` (within a small epsilon) and of two orthogonal vectors is `~0.0`. Pins the two anchor points of the metric.
- `top_k_ranks_by_similarity` — build a `VectorIndex` of three rows (one near the query, one opposite, one in between); `top_k(query, 2)`; assert the result has length `2`, the first id is the nearest row, and scores are non-increasing (`hits[0].1 >= hits[1].1`).
- `embed_node_runs_after_sources` (integration-style) / `cli_parses_embed_subcommand` — assert `Cli::try_parse_from(["panoptes-etl", "embed"])` yields `Command::Embed`, and (fuller build) that the executor schedules the embed node only after `spacedevs` succeeds.

<div class="callout predict">
<span class="callout-label">Predict before you run</span>
In <code>top_k_ranks_by_similarity</code>, with query <code>[1.0, 0.0]</code> and rows <code>near=[1.0, 0.1]</code>, <code>far=[-1.0, 0.0]</code>, <code>mid=[0.5, 0.5]</code>: write down the order <code>top_k(query, 2)</code> returns <em>before</em> running. If <code>far</code> appears in your top 2, you sorted ascending — the highest cosine must come first.
</div>

## The build loop (you drive)

1. **Write `cosine_of_identical_is_one_orthogonal_is_zero`**, implement `cosine` with the zero-norm guard.
2. **Write `top_k_ranks_by_similarity`.** Predict the ranking, then implement `top_k` with `total_cmp` + `truncate`.
3. **Add the `Embed` subcommand** and `cli_parses_embed_subcommand`; wire `embed_launches` / `read_embedded_ids` in `main`.
4. **Run green, commit.** Optionally: promote the embed step to a real DAG node downstream of `spacedevs` and prove ordering.

## Done when

`cargo test -p etl-core` covers `cosine`/`top_k` and `cargo test -p panoptes-etl` covers the subcommand. You can now run `panoptes-etl run` to populate `launches.jsonl`, then `panoptes-etl embed` to turn its prose into a deduplicated vector index, and `VectorIndex::top_k` answers "what launches are most like this query vector?" — the semantic half of retrieval, standing beside the structured filters, exactly as the split promised. That completes the four seams and the RAG arc: the pipeline now extracts, transforms, loads, schedules, *and* retrieves.
