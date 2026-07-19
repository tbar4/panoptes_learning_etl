# Build: Retry Middleware + the Idempotent Sink

> **Maps to:** Tasks 3.1 + 3.2. **Kind:** Build — spec and test names only. No implementation.

## Objective

Build the two `etl-core` seams the paginated source depends on: `with_retry` + `RetryPolicy` (retry transient failures with capped exponential backoff, honoring `Retry-After`) and the content-addressing machinery — `content_id`, `Row::content_key`, and the `IdempotentSink` decorator that makes re-running the pipeline write each row at most once.

Both live in `etl-core` (not `etl-sources`): they are seams every other crate consumes, so they belong in the crate the dependency arrow points *toward*.

## Scaffold

**Create:** `crates/etl-core/src/retry.rs` and `crates/etl-core/src/content.rs`; add `pub mod retry;` / `pub mod content;` and re-exports to `crates/etl-core/src/lib.rs`:

```text
pub use content::{content_id, IdempotentSink};
pub use retry::{with_retry, RetryPolicy};
```

**Dependencies (etl-core):** add `sha2` (content ids) and ensure `tokio` (with `time` — for `sleep` and, in tests, `test-util`) and `async-trait` are present. `serde_json` is already there for `Row`'s canonical form.

> `sha2` and `tokio` are **not** playground-safe; build and test in the workspace with `cargo test -p etl-core retry` and `cargo test -p etl-core content`.

**Expected result:** `cargo test -p etl-core retry` and `cargo test -p etl-core content` → all tests green.

## Part A — `with_retry` + `RetryPolicy` (`retry.rs`)

**`RetryPolicy`** — `#[derive(Clone, Copy, Debug)]` with fields `max_attempts: u32`, `base_delay: Duration`, `max_delay: Duration`. `Default` = `{ max_attempts: 4, base_delay: 200ms, max_delay: 10s }`.

**`with_retry`** — the signature and behavior:

```text
pub async fn with_retry<F, Fut, T>(policy: RetryPolicy, op: F) -> Result<T, EtlError>
where F: FnMut() -> Fut, Fut: Future<Output = Result<T, EtlError>>
```

- Loop, calling `op().await` each iteration. On `Ok`, return it.
- On `Err`, increment `attempt`. If `attempt >= policy.max_attempts` **or** `!err.is_transient()`, return the error immediately.
- Otherwise compute the backoff — `err.retry_after()` if the error carries one (a `429`'s `Retry-After`), else capped exponential `(base_delay * 2^(attempt-1)).min(max_delay)` — `sleep` it, and loop.

`op` is `FnMut` returning a fresh `Future` each call because retrying means re-running it.

**Test names to hit** (all `#[tokio::test(start_paused = true)]`, using an `AtomicU32` to count `op` calls):

- `retries_transient_then_succeeds` — `op` returns `Network` on the first two calls, then `Ok(42)`; assert the result is `42` and `op` ran exactly **3** times.
- `gives_up_on_permanent_immediately` — `op` always returns `Parse`; assert the error is `Parse` and `op` ran exactly **1** time (no retry on a permanent error).
- `honors_retry_after` — `op` returns `RateLimited { retry_after_secs: 5 }` once, then `Ok`; capture `tokio::time::Instant::now()` before and assert `start.elapsed() >= 5s` (the server's `Retry-After` beats the small base backoff — provable in microseconds because the clock is paused).

## Part B — content addressing + `IdempotentSink` (`content.rs`)

**`content_id`** — `pub fn content_id(kind: &str, canonical: &str) -> String`: SHA-256 of `kind` bytes, then a single `0x00` byte, then `canonical` bytes, formatted as lowercase hex. The `kind` tag prevents cross-entity id collisions; the `0x00` separator removes boundary ambiguity between `kind` and `canonical`.

**`Row::content_key`** — `pub fn content_key(&self) -> Result<String, EtlError>`: serialize `self` to JSON (`serde_json::to_string`, error → `EtlError::Parse`), then `content_id("row", &canonical)`. **Fallible on purpose** — never `unwrap_or_default`, which would collapse distinct un-serializable rows onto one shared key and silently drop them as duplicates.

**`IdempotentSink<S: Sink>`** — fields `inner: S` and `seen: Mutex<HashSet<String>>`. Two constructors: `new(inner)` (empty seen-set) and `with_seen(inner, keys: impl IntoIterator<Item = String>)` (pre-loaded, for cross-process re-runs). It implements `Sink`: under the lock, `insert` each row's `content_key()?` into `seen`; the rows whose key was newly inserted are "fresh." If none are fresh, return `Ok(0)`; otherwise forward only the fresh rows to `inner.load` and return its count.

**Test names to hit:**

- `same_content_hashes_stable_distinct_content_differs` — `content_id("row","x") == content_id("row","x")`; `!= content_id("row","y")`; and `content_id("a","x") != content_id("b","x")` (the `kind` tag matters).
- `content_key_is_stable_and_distinguishes_rows` — two equal `Row::Launch`es have equal keys; a row with a changed field has a different key.
- `rerun_writes_zero_new_rows` — wrap a counting sink; `load` two rows returns `2`; loading the identical two again returns `0`.
- `distinct_rows_all_written` — three distinct rows through the sink return `3`.

## Concepts exercised

- Capped exponential backoff, transient-only retry, `Retry-After` precedence (*Retry with Backoff*).
- Deterministic timing tests via tokio's paused clock.
- Content-addressed ids and the decorator-`Sink` idempotency pattern (*Idempotent, Content-Addressed Writes*).

## The build loop (you drive)

1. **`content_id` first — pure and tiny.** Write `same_content_hashes_stable_distinct_content_differs`, implement, run green.
2. **`content_key`.** Write `content_key_is_stable_and_distinguishes_rows`. Predict: why must the return type be `Result`, not `String`?
3. **`IdempotentSink`.** Write `rerun_writes_zero_new_rows` against a counting stub sink; implement the decorator; then `distinct_rows_all_written`.
4. **`with_retry`.** Write `gives_up_on_permanent_immediately` (proves the `is_transient` gate) before `retries_transient_then_succeeds`. Add `honors_retry_after` last, under `start_paused = true`.
5. **Run green, commit.**

## Done when

`cargo test -p etl-core retry content` is green. `with_retry` and `IdempotentSink` are now available to every source — `SpaceDevsSource` already wraps each page fetch in `with_retry`, and any pipeline can slip an `IdempotentSink` in front of its `JsonlSink` to make re-runs safe. These are the seams the orchestration and retrieval arcs will spend.
