# Author Report — Part IV: The Paginated Arc (TheSpaceDevs)

Authored 2026-07-19. Source of truth: the verified `panoptes_etl` crates
(`etl-sources/src/spacedevs.rs`, `etl-sources/src/http.rs`;
`etl-core/src/{retry.rs,content.rs,error.rs,domain.rs,lib.rs}`). Everything below
was checked against that shipped code, which compiles and passes tests.

## Files written (overwrote stubs)

- `src/03-paginated/concept-pagination.md` — cursor pagination, the follow-`next`
  loop, the visited-set cycle guard. Runnable toy (in-memory URL→Page map) +
  verbatim `ignore` excerpt of the shipped extract loop. Cycle = `EtlError::Parse`
  (permanent). One-`RawRecord`-per-page rationale.
- `src/03-paginated/concept-retry-backoff.md` — capped exponential backoff,
  transient-only via `is_transient()`, `Retry-After` wins. Runnable toy that
  records the delay each attempt would sleep; verbatim `ignore` of `with_retry`
  and the `honors_retry_after` paused-clock test. Chain: status → classify_status
  → EtlError → with_retry.
- `src/03-paginated/concept-content-addressing.md` — `content_id(kind ‖ 0x00 ‖
  canonical)`, `Row::content_key` (fallible), `IdempotentSink` decorator. Two
  runnable toys: (1) content ids + idempotent dedup; (2) the `unwrap_or_default`
  trap collapsing distinct rows. Verbatim `ignore` of content_id / content_key /
  IdempotentSink::load.
- `src/03-paginated/build-spacedevs.md` — Task 3.3 spec: SpaceDevsSource
  (base_url + policy, cycle guard, with_retry per page) + SpaceDevsTransform
  (nested LL2 → Row::Launch). Real test names. NO implementation.
- `src/03-paginated/build-retry-idempotent.md` — Tasks 3.1+3.2 spec: with_retry /
  RetryPolicy + content_id / content_key / IdempotentSink. Real test names. NO impl.
- `src/03-paginated/quiz.md` — intro + `{{#quiz ../../quizzes/03-paginated.toml}}`.
- `quizzes/03-paginated.toml` — 7 questions (3 Tracing, 2 MultipleChoice, 2 ShortAnswer).

## Examples compiled + output captured (rustc --edition 2021, std-only)

All CONCEPT-page ```rust``` blocks compiled and run in isolation
(scratchpad `etl4/`). Captured output pasted verbatim into the ```text``` fences:

| Example | Block kind | Verified output |
| --- | --- | --- |
| pagination drain + cycle (concept-pagination) | ```rust``` | `Ok(["a", "b", "c"])` / `Err("pagination cycle at /p1")` |
| backoff decision watcher (concept-retry-backoff) | ```rust``` | `transient x3 -> Ok("ok"), delays = [200ms, 400ms, 800ms]` / `permanent    -> Err("Permanent"), delays = []` / `rate-limited -> Ok("ok"), delays = [5s]` |
| content ids + idempotent dedup (concept-content-addressing) | ```rust``` | `iss/iss again` identical (`471231c61412846c`), `hst` differs, `as launch` differs; `wrote 2 of 3 rows` |
| unwrap_or_default collapse trap (concept-content-addressing) | ```rust``` | `bad key A = ""` / `bad key B = ""` / `wrote 1 of 2 distinct rows (should be 2)` |

`ignore` blocks are lifted **verbatim** from the shipped crates (with_retry,
honors_retry_after, extract loop, content_id, content_key, IdempotentSink::load)
and carry a dep-list note (reqwest/async-trait/chrono/tokio/sha2 are not
playground-safe). No non-verbatim `ignore` blocks were written. No
```rust,compile_fail``` used this part (no error-code assertion was pedagogically
needed).

## Tracing quiz programs (compiled by rustc, std-only)

- Q1 capped exponential backoff: `doesCompile = true`, `stdout = "[200, 400, 800, 1000, 1000]"` — verified.
- Q2 pagination cycle guard: `doesCompile = true`, `stdout = "cycle after 2 fetches"` — verified.
- Q3 content dedup w/ kind tag: `doesCompile = true`, `stdout = "3"` — verified.

## UUIDs (all unique, uuidgen-generated; TOML validated with tomllib)

- bdbe85d2-00ac-4f0c-a695-b565ffae22ce — Tracing, backoff sequence
- 06173184-1ccf-4d04-ba6f-3769a010ddf9 — Tracing, cycle guard fetch count
- 936c05ee-867e-415b-a447-f527d5cec9d9 — Tracing, content dedup + kind tag
- e5d0415c-c165-4aaa-8d9d-37f42a5e7cb7 — MC, Retry-After wins over backoff
- fd014a78-a764-4cb9-a16b-c0973aa48d0f — MC, pagination termination + cycle guard
- 0ca1a899-c43b-4058-aced-58c0c8847c54 — ShortAnswer, fallible content_key (duplicates)
- d162e106-e882-4158-b3f4-390efc257837 — ShortAnswer, is_transient decides retry

Verified: no id collisions across all `quizzes/*.toml`; each quiz toml referenced
by exactly one `{{#quiz}}`; `{{#quiz}}` appears only on `quiz.md` (concept pages
end with "Questions to lock" + a pointer to `quiz.md`).

## Taught the shipped code where the plan diverged

The plan is stale in three places; I taught the shipped reality (source of truth):

1. **`retry.rs` lives in `etl-core`, not `etl-sources`.** Plan Task 3.1 said
   `crates/etl-sources/src/retry.rs`; shipped code has it in `etl-core`
   (`pub use retry::{with_retry, RetryPolicy}`). Both build pages and concepts
   place it in `etl-core` and explain why (seam every crate consumes).
2. **`content_key` is fallible and there is no `Conjunction` id example.** Plan
   Task 3.2 described `IdempotentSink { inner: JsonlSink, seen: HashSet }` and a
   Conjunction-specific id. Shipped: `IdempotentSink<S: Sink>` generic over any
   sink, `seen: Mutex<HashSet<String>>`, with `new` **and** `with_seen`; the key
   is `Row::content_key() -> Result<String, EtlError>` (never unwrap_or_default).
   Taught the generic decorator + fallible key + the swallow trap.
3. **Real test names over plan names.** Shipped names differ:
   - retry.rs: `retries_transient_then_succeeds`, `gives_up_on_permanent_immediately`,
     `honors_retry_after` (match plan).
   - content.rs: `same_content_hashes_stable_distinct_content_differs`,
     `content_key_is_stable_and_distinguishes_rows`, `rerun_writes_zero_new_rows`,
     `distinct_rows_all_written` (plan listed `same_row_hashes_stable`).
   - spacedevs.rs: `follows_next_until_null`, `rejects_pagination_cycle`,
     `http_error_is_network_error`, `parses_nested_launch_json` (the plan's
     `429_triggers_backoff_then_succeeds` is NOT what shipped; the shipped source
     has a `500`→Network test and a self-referential-cycle test instead).

## Assumptions / deviations

- **Quiz path.** Task says `quizzes/03-paginated.toml` and
  `{{#quiz ../../quizzes/03-paginated.toml}}`; the plan text says
  `04-pagination.toml`. Used `03-paginated.toml` — matches the task and the
  existing `src/03-paginated/quiz.md` chapter path and SUMMARY.md ordering.
- **SpaceDevsSource also uses `with_retry` internally.** The shipped extract wraps
  each page fetch in `with_retry(self.policy, …)`. build-spacedevs notes this and
  cross-references build-retry-idempotent so no chapter relies on un-introduced
  machinery (concept-retry-backoff introduces `with_retry` before both builds).
- **No `mdbook build` run** (per instructions). Verified only that all ```rust```
  toys and all three Tracing programs compile and run in isolation.

## Not done / for the central build

- `mdbook build` (validates Tracing via real rustc, resolves `{{#quiz}}` once).
- Answer-key anchor links not added (Part IV appendix anchors not finalized);
  build pages point readers to the concept pages and the spec instead.
