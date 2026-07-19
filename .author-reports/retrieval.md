# Author Report — Part VI: The Retrieval Arc (RAG)

Authored 2026-07-19. Source of truth: the verified `panoptes_etl` crates
(`etl-core`: `embed.rs`, `vector.rs`, `content.rs`; `etl-sources`: `voyage.rs`;
`panoptes-etl`: `main.rs`, `cli.rs`). Everything below was checked against that
code, which compiles and passes tests. Teaches the shipped reality only.

## Files written (overwrote stubs)

- `src/05-retrieval/concept-structured-vs-semantic.md` — the sharp split: numbers →
  filter/sort (structured), prose → embed + cosine (semantic); three reasons not to
  embed numbers; `Row::prose()` as the single point of truth; the two halves of
  semantic search (embed corpus + embed query, same model); brute-force cosine is
  *correct* at this scale, not a stopgap. Toy mirror: a bookstore (numeric price
  filter beside a cosine ranking over prose blurbs).
- `src/05-retrieval/concept-embedder.md` — the `Embedder` trait (mirrors the first
  course's `ModelClient`); embeddings as the definitive expensive/idempotent/
  content-addressed workload; mock it in tests; seeding the `seen` set from disk
  via `content_id`. Toy mirror: a sync `Embed` trait + `MockEmbedder` + a
  content-id `HashSet` cache.
- `src/05-retrieval/build-embed-sink.md` — spec + test names, NO impl: `VoyageEmbedder`
  (bearer auth, empty-input no-op, `classify_status`) and `EmbedSink` (prose-only,
  content-addressed, lock-scope trap, length guard).
- `src/05-retrieval/build-retriever-node.md` — spec + test names, NO impl:
  `cosine`/`VectorIndex::top_k` (`total_cmp` callout) + wiring the embed step as the
  `embed` subcommand the orchestrator schedules (with the DAG-node promotion noted).
- `src/05-retrieval/quiz.md` — intro + `{{#quiz ../../quizzes/05-retrieval.toml}}`.
- `quizzes/05-retrieval.toml` — 7 questions (3 Tracing, 2 MultipleChoice, 2 ShortAnswer).

Convention followed: concept pages END with "Questions to lock" + a pointer to
`./quiz.md`; the `{{#quiz}}` include appears ONLY on `quiz.md` (referenced once).

## Examples compiled + output captured (scratch cargo project + rustc)

All CONCEPT-page runnable blocks compiled and run in isolation
(`scratchpad/rag_examples` cargo project; `rustc --edition 2021` for the rest):

| Example | Block kind | Verified output |
| --- | --- | --- |
| bookstore structured-vs-semantic (concept-structured-vs-semantic) | ```rust``` (std) | `under $20:` / `The Quiet Garden ($15.00)` / `Rockets for Kids ($12.00)` / `closest in meaning:` / `Orbital Mechanics (0.998)` / `Rockets for Kids (0.938)` |
| text-is-not-a-vector (concept-structured-vs-semantic) | ```rust,compile_fail``` | `error[E0308]: arguments to this function are incorrect ... expected &[f32], found &[&str; 1]` |
| `Embedder` trait (concept-embedder) | ```rust,ignore``` (async-trait) | dep note only — not playground-safe |
| mock-embedder + content-id cache (concept-embedder) | ```rust``` (std) | `this load embedded 2 fresh text(s)` / `this load embedded 1 fresh text(s)` / `total texts actually embedded: 3` |
| `Row::prose` / `EmbedSink::new`/`with_seen` (concept-embedder) | ```rust,ignore``` (etl-core) | dep note only — not playground-safe |

Playground rule applied: pure `cosine`/`top_k`/std examples → ```rust```; anything
needing async-trait / reqwest / etl-core → ```rust,ignore``` with a dep-list note;
the type-error demo → ```rust,compile_fail``` asserting `E0308`.

## Tracing quiz programs (compiled by rustc, std-only, clean under `-D warnings`)

- Q1 cosine zero-norm + orthogonal: `doesCompile = true`, `stdout = "0 0"` — verified
  (zero-vector guard → 0.0; orthogonal dot → 0.0; `f32` Display prints `0`).
- Q2 `top_k` descending rank: `doesCompile = true`, `stdout = ["a", "c"]` — verified
  (a=1.0, c≈0.707, b=−1.0; descending `total_cmp` + `truncate(2)`).
- Q3 content-id cache dedupe: `doesCompile = true`, `stdout = "2"` — verified
  (4 texts, 2 distinct; `HashSet::insert` returns false on repeats).

TOML escaping note: Q3's program contains `\u{0}` written as `\\u{0}` in the TOML
triple-quoted string; after TOML unescaping it matches the rustc-verified source.
`answer.stdout` for Q2 is `"[\"a\", \"c\"]"` → `["a", "c"]`.

## UUIDs (all unique, uuidgen-generated, lowercased)

- 272bd730-3de0-4816-aff2-ddd45e12def2 — Tracing, cosine zero-norm + orthogonal
- 90a421ea-efb0-4ac7-8a5c-8ee86817e247 — Tracing, top_k descending rank
- 772442a7-c6b0-4012-a132-c386d583752f — Tracing, content-id cache dedupe
- 6bdedb1f-8d48-4a8e-9d7e-b6dc5703b2ef — MC, why numbers stay out of the index
- ff4e2a3e-bc10-4a71-8bd8-ef3da769b5e3 — MC, brute-force is correct at this scale
- 849f1461-db24-4b30-aa70-35247b4c7ae7 — ShortAnswer, empty-input request count (0)
- 7b50ea1c-79ad-405d-9f71-cddb60900c8d — ShortAnswer, content_id bridges the cache

## Assumptions / deviations

1. **Quiz path.** Plan Task 5.3 text says `quizzes/06-retrieval.toml`; the task brief
   and the existing `SUMMARY.md` link both say `quizzes/05-retrieval.toml`. Used
   `05-retrieval.toml` (matches SUMMARY.md, which resolves `../../quizzes/05-retrieval.toml`).
2. **Real test names over plan names.** Source of truth is the shipped code, so build
   pages list the *actual* verified test names:
   - Voyage: `embeds_batch_of_texts_with_bearer_auth` (plan said `embeds_batch_of_texts`),
     plus `empty_input_makes_no_request` (a real shipped test the plan omitted).
   - Index: `cosine_of_identical_is_one_orthogonal_is_zero` (plan said `cosine_of_identical_is_one`),
     `top_k_ranks_by_similarity`, `embed_sink_only_touches_prose`, `cached_text_not_re_embedded`.
   - CLI: taught `cli_parses_embed_subcommand` alongside the plan's `embed_node_runs_after_sources`
     (the shipped CLI test suite parses subcommands; the DAG-node form is presented as the fuller build).
3. **Embed step = subcommand, DAG node noted.** Shipped reality is the `Embed` CLI
   subcommand calling `embed_launches` (reads `launches.jsonl`, seeds from the existing
   `embeddings.jsonl` via `read_embedded_ids` + `EmbedSink::with_seen`). Taught that as
   the truth; the "promote to a `Pipeline` downstream of spacedevs" story is framed
   explicitly as the fuller build, matching the code comment in `main.rs`.
4. **Removed a stray dimension figure.** Early draft said "1024-dimensional"; softened to
   "many-dimensional" since the shipped `voyage-3` default is not asserted to a fixed dim
   anywhere the reader builds against — avoids teaching an unverified number.
5. **No `mdbook build` run** (per instructions). Verified only that examples and Tracing
   programs compile/run in isolation; Tracing programs are clean under `-D warnings`.

## Not done / for the central build

- `mdbook build` (validates Tracing via real rustc, resolves each `{{#quiz}}` once).
- Answer-key anchor links (`appendix-answer-key.html#task-5-...`) were NOT added — the
  Part VI task anchors are not yet finalized in the (stub) answer-key appendix; build
  pages point readers to the concept pages and the spec, matching the Part II convention.
