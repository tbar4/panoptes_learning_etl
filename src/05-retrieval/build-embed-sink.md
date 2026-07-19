# Build: The Embed Load Target

> **Maps to:** Task 5.1 + 5.2 (the load half). **Kind:** Build — spec and test names only. No implementation.

## Objective

Implement two things that form the *write* side of semantic retrieval: `VoyageEmbedder`, a concrete `Embedder` backed by a real HTTP API but tested entirely against a mock; and `EmbedSink`, a `Sink` that embeds **prose only**, **content-addressed** so the same text is never embedded twice. Together they are how launch descriptions become durable, deduplicated vectors on disk.

## Scaffold

**Create:** `crates/etl-core/src/vector.rs` (the `EmbedSink`, plus `Row::prose`); `crates/etl-sources/src/voyage.rs` (the `VoyageEmbedder`). The `Embedder` trait itself lives in `crates/etl-core/src/embed.rs` (Task 5.1). Declare the modules and re-export `EmbedSink`, `EmbeddedRow`, `VectorIndex`, `cosine` from `etl-core`'s `lib.rs`, and `VoyageEmbedder` from `etl-sources`'s `lib.rs`.

**Dependencies:** none new. `reqwest`, `serde`/`serde_json`, `tokio`, `async-trait`, `sha2` and the dev-only `wiremock` were all declared in earlier arcs. The embedder reuses the HTTP-error classifier (`crate::http::classify_status`) from the authenticated arc.

**Expected result:** `cargo test -p etl-sources voyage` → the Voyage tests pass; `cargo test -p etl-core embed` (or `vector`) → the sink tests pass.

## The spec (givens) — `VoyageEmbedder`

- Fields: `base_url`, `api_key`, `model` (all `String`; `base_url` injectable so tests point at a mock). Constructor `new(base_url, api_key)` defaulting `model` to `"voyage-3"`; plus `from_env(base_url)` that reads `VOYAGE_API_KEY` and maps a missing key to `EtlError::Auth`.
- Implements `Embedder`. `embed(&self, texts: &[String])`:
  - **Empty input is a no-op:** if `texts` is empty, return `Ok(vec![])` *without* making a request. (This is a real correctness rule — an empty batch must not cost a round trip, and some APIs reject an empty `input`.)
  - `POST {base_url}/v1/embeddings` (trim a trailing `/` off `base_url`), **bearer auth** with the key, JSON body `{"input": <texts>, "model": <model>}`.
  - On a non-success status, return `classify_status(status, headers)` — the same transient/permanent split the retry layer reads.
  - Parse `{"data": [{"embedding": [...]}, ...]}` and return the embeddings in order. A parse failure maps to `EtlError::Parse`.

## The spec (givens) — `Row::prose` + `EmbedSink`

- **`Row::prose(&self) -> Option<String>`** — returns `Some(description)` only for a `Row::Launch` whose `description` is non-empty after `trim`; `None` for `SpaceObject`, `Conjunction`, and empty-description launches. This method *is* the structured-vs-semantic split; the sink embeds exactly what it returns.
- **`EmbeddedRow { id: String, text: String, vector: Vec<f32> }`** — one embedded prose field; `Serialize`/`Deserialize`, so it round-trips through JSONL.
- **`EmbedSink<E: Embedder>`** — fields: the `embedder`, a `path: PathBuf`, and a `seen: Mutex<HashSet<String>>`. Two constructors: `new(embedder, path)` (empty `seen`) and `with_seen(embedder, path, ids)` (seed `seen` from ids already on disk).
- **`impl Sink for EmbedSink`** — `load(&self, rows)`:
  1. For each row, take `row.prose()`; skip `None`. Compute `content_id("embed", &text)`. Insert the id into `seen`; keep the text only if the insert was *new* (dedupe within and across loads).
  2. If nothing is fresh, return `Ok(0)` — **no embed call**.
  3. Embed the fresh texts in one batch. **Guard the length:** if the embedder returns a vector count that differs from the text count, return `EtlError::Parse` — never silently misalign texts and vectors.
  4. Append one `EmbeddedRow` per (id, text, vector) as JSONL (create + append, single buffered `write_all`, same file discipline as `JsonlSink`). Return the count written.

<div class="callout trap">
<span class="callout-label">Lock the `seen` set only while selecting fresh work — never across the <code>await</code></span>
The dedupe check touches shared state (<code>seen</code>); the embed call is a slow network <code>await</code>. Hold the lock only long enough to insert ids and collect the fresh texts, then <em>drop it</em> before <code>embed(...).await</code>. Hold it across the await and every concurrent load serializes behind one HTTP call — you have turned a batching sink into a global mutex. Scope the lock in an inner block; the shipped code does exactly this.
</div>

## Test names to hit

**`VoyageEmbedder` (wiremock):**

- `embeds_batch_of_texts_with_bearer_auth` — mount a mock matching `POST /v1/embeddings` **and** `header("authorization", "Bearer test-key")`, returning two embeddings. Call `embed` with two texts; assert the parsed vectors equal the mocked ones. The header matcher is an assertion that your client actually sent bearer auth.
- `empty_input_makes_no_request` — construct an embedder pointing at an unroutable address, mount *no* mock, and assert `embed(&[])` returns an empty `Vec` — proving the empty-input short-circuit never calls out.

**`EmbedSink`:**

- `embed_sink_only_touches_prose` — load a `Launch` (has prose) and a `Conjunction` (no prose) through a `CountingEmbedder`; assert the return is `1` and the embedder was called for exactly `1` text. Proves the split: numbers are never embedded.
- `cached_text_not_re_embedded` — load two launches with *identical* descriptions; assert `1` written and `1` embed call (dedupe within a load). Load the same rows again; assert `0` written and the call count *unchanged* (dedupe across loads). This is the content-addressed cache, made a test.

<div class="callout predict">
<span class="callout-label">Predict before you run</span>
In <code>cached_text_not_re_embedded</code>, before writing the code: what does the *second* <code>load</code> of the same rows return, and how many total embed calls has the <code>CountingEmbedder</code> seen by the end? Write both numbers down. If your prediction for the second load is anything but <code>0</code> new rows and an <em>unchanged</em> call count, your dedupe is keyed on the wrong thing.
</div>

## The build loop (you drive)

1. **Build the `Embedder` trait first** (Task 5.1, `embed.rs`) — one async method; it is three lines but it is the seam everything else names.
2. **Write `embeds_batch_of_texts_with_bearer_auth`**, then implement `VoyageEmbedder` against it. Add `empty_input_makes_no_request` and confirm the short-circuit.
3. **Write `embed_sink_only_touches_prose`** with a `CountingEmbedder` mock (deterministic vectors, an atomic call counter). Implement `Row::prose` and the prose-filtering half of `load`.
4. **Write `cached_text_not_re_embedded`.** Predict both loads' results, then implement the `seen`/`content_id` dedupe. Watch the lock scope (see the trap).
5. **Run green, commit.**

## Done when

`cargo test -p etl-sources voyage` and `cargo test -p etl-core embed` both pass. You now have the write side of retrieval: prose becomes deduplicated `EmbeddedRow` JSONL, at most one embed call per distinct text, and a real HTTP client proven correct without spending a cent. The next build page reads those vectors back — the search side — and wires the whole embed step into the orchestrator.
