# Capstone: Add the NASA Source (You Drive)

> **Maps to:** Task 4.4. **Kind:** Capstone — specification only. No walkthrough, no test names handed to you, no build loop. You have built three sources; this is the one you build alone.

## The claim to prove

The entire arc has been an argument that the `Source` trait absorbs a new data source with **zero orchestrator changes**. This capstone is where you cash that claim. You will add a fourth source — NASA — as a new leaf in `etl-sources`, wire it into the CLI exactly like the other three, and watch the DAG schedule it without a single edit to `etl-orchestrate`. If you find yourself opening `executor.rs` or `graph.rs`, stop: the design says you should not have to, and if you do, something is being done at the wrong layer.

## What you are building

A new source that reads from NASA's open API at `api.nasa.gov`, normalizes its responses into `Row`s, and runs as a pipeline in the DAG alongside CelesTrak, Space-Track, and TheSpaceDevs.

**You choose the endpoint and the mapping.** NASA publishes many datasets behind one API key — APOD (astronomy picture of the day), NeoWs (near-Earth objects), DONKI (space-weather notifications), and more. Pick one whose shape appeals to you. The point of the capstone is not a specific endpoint; it is that *any* of them slots in through the same trait.

## Hard requirements (the givens)

- **New file:** `crates/etl-sources/src/nasa.rs`, exported from that crate's `lib.rs`. This is the *only* new source code the capstone requires.
- **API key from the environment, passed as a query parameter.** Read `NASA_API_KEY` from the environment (as the other credentialed sources read theirs) and attach it to the request as a query param — e.g. `?api_key=…`. It is a query parameter, **not** a header, for this API. Never hard-code the key; never commit it.
- **A `NasaSource` implementing `Source`.** `name()` returns a stable string like `"nasa"`; `extract()` does the HTTP GET, maps transport errors and non-success statuses through the same `classify_status` helper the other sources use, and returns `RawRecord`(s).
- **A transform implementing `Transform`.** Parse the JSON payload into one or more of the existing `Row` variants, or add a new `Row` variant in `etl-core` if your chosen endpoint genuinely needs one. *You* decide the `Row` mapping — that decision is the substance of the capstone.
- **Tested against `wiremock`, never the live API.** Follow the pattern from the CelesTrak build: a mock server, a `query_param("api_key", …)` matcher, a canned body, assertions on the parsed `RawRecord` / `Row`. Watch for the silent-404 trap when your matcher chain is wrong.
- **Wired into the CLI like the other three.** Add a `NasaSource` pipeline in `build_executor` in `panoptes-etl/src/main.rs`, with an idempotent JSONL sink and the same freshness window.

## The extension point

`build_executor` in `crates/panoptes-etl/src/main.rs` already marks exactly where your pipeline goes. The comment is there waiting for you:

```text
// CAPSTONE (you drive, unassisted): add your NASA pipeline here.
// A `NasaSource` reads its key from `NASA_API_KEY` and passes it as a query
// parameter; wire it with a transform and a `JsonlSink` exactly like the
// three above. The Source trait is all the orchestrator needs — nothing in
// this file's structure has to change to admit a fourth source.
```

Adding your pipeline is three or four lines mirroring the CelesTrak block right above it: construct the source, wrap it in a `Pipeline` with a transform and an idempotent sink, set `refresh_after`, and `add_pipeline("nasa", …)`.

## The property to observe (this is the grade)

When you are done, take stock of what you did **not** touch:

- `etl-orchestrate/src/graph.rs` — unchanged.
- `etl-orchestrate/src/executor.rs` — unchanged.
- `etl-orchestrate/Cargo.toml` — still depends on `etl-core` only. Re-run `cargo tree -p etl-orchestrate` and confirm `etl-sources` is *still* absent, even though a fourth source now runs through the executor.

The orchestrator scheduled a source that did not exist when the orchestrator was written, because it only ever knew the trait. That is dependency inversion delivering — not as a slogan, but as a diff you can measure by its emptiness. A new capability arrived by *adding a leaf*, not by *editing the core*.

## Self-checks (you write the tests)

No test names are given — deciding what to assert is part of the exercise. But you are done when, at minimum:

1. `cargo test -p etl-sources nasa` is green: the extract test hits a mock (asserting your `api_key` query param is sent), and the transform test turns a canned payload into the `Row`(s) you designed.
2. `panoptes-etl status` lists your new source (extend `SOURCES` if you want it in the status report), and `panoptes-etl run` executes it in DAG order with the others.
3. `git diff --stat` shows changes in `etl-sources` and `panoptes-etl` — and nothing in `etl-orchestrate`.

<div class="callout">
<span class="callout-label">This is the graduation moment</span>
Every seam the course installed — the E/T/L traits, the run ledger, the retry middleware, the DAG, the dependency-inverted executor — exists so that <em>this</em> is a small, local, unassisted change. If adding the fourth source felt like filling in a template you already understood, that is the whole course working as designed. The next arc (Retrieval) adds one more kind of node — an embedding step — and it, too, slots in the same way.
</div>
