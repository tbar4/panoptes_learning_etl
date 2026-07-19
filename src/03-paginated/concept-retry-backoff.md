# Concept: Retry with Backoff & 429 / Retry-After

> **Kind:** Concept. **Seam spent this arc:** the transient/permanent classification from Part I becomes a real retry loop.

## The idea

A paginated extract makes *many* requests, and over many requests a transient failure is not a possibility, it is a certainty: a connection resets, a load balancer hiccups, the API returns `429 Too Many Requests` because you are pulling a hundred pages in a row. The wrong response is to let the whole run die on the first blip. The right response is to **retry** — but only the failures that retrying could actually fix, and only after **waiting**, so you do not amplify the problem by hammering a struggling server.

Two rules make a retry loop correct rather than dangerous:

1. **Retry only transient errors.** Retrying a `404` or a checksum failure will produce the identical failure every time — it just wastes attempts and delays the inevitable error. This is precisely the `is_transient()` distinction you built in Part I: `Network` and `RateLimited` are worth retrying; `Parse`, `Auth`, and `Permanent` are not.
2. **Back off between attempts, with a cap.** Wait longer after each successive failure so a momentarily overloaded server gets breathing room — but cap the wait so you do not sleep for an hour. This is **capped exponential backoff**: `base`, `2×base`, `4×base`, … clamped to `max_delay`. And when the server *tells* you how long to wait (a `Retry-After` header on a `429`), that number wins over anything you would compute.

## A runnable toy: watching the backoff decision

The real `with_retry` needs `tokio` to actually sleep, so it is not playground-safe. But the *decision* it makes on each failure is pure arithmetic, and we can watch it. This toy mirrors the shipped logic exactly — retry only transient, `Retry-After` wins, otherwise capped exponential — and instead of sleeping, it records the delay each attempt *would* wait:

```rust
use std::time::Duration;

// A tiny error taxonomy mirroring EtlError's transient/permanent split.
#[derive(Debug)]
enum Err_ {
    Network,                    // transient
    RateLimited { after: u64 }, // transient, carries a server backoff
    Permanent,                  // never worth retrying
}

impl Err_ {
    fn is_transient(&self) -> bool {
        matches!(self, Err_::Network | Err_::RateLimited { .. })
    }
    fn retry_after(&self) -> Option<Duration> {
        match self {
            Err_::RateLimited { after } => Some(Duration::from_secs(*after)),
            _ => None,
        }
    }
}

struct Policy {
    max_attempts: u32,
    base: Duration,
    cap: Duration,
}

// The exact backoff decision `with_retry` makes, pulled out so we can watch it:
// a server-supplied Retry-After wins; otherwise capped exponential.
fn backoff(policy: &Policy, err: &Err_, attempt: u32) -> Duration {
    err.retry_after().unwrap_or_else(|| {
        let factor = 2u32.saturating_pow(attempt - 1);
        (policy.base * factor).min(policy.cap)
    })
}

// Run `op`, collecting the delays a real loop would sleep between attempts.
fn run(
    policy: &Policy,
    mut op: impl FnMut(u32) -> Result<&'static str, Err_>,
) -> (Result<&'static str, String>, Vec<Duration>) {
    let mut delays = Vec::new();
    let mut attempt = 0u32;
    loop {
        match op(attempt) {
            Ok(v) => return (Ok(v), delays),
            Err(e) => {
                attempt += 1;
                if attempt >= policy.max_attempts || !e.is_transient() {
                    return (Err(format!("{e:?}")), delays);
                }
                delays.push(backoff(policy, &e, attempt));
            }
        }
    }
}

fn main() {
    let policy = Policy {
        max_attempts: 5,
        base: Duration::from_millis(200),
        cap: Duration::from_secs(10),
    };

    // Transient until the 4th call: watch the delays double.
    let (r, d) = run(&policy, |n| if n < 3 { Err(Err_::Network) } else { Ok("ok") });
    println!("transient x3 -> {r:?}, delays = {d:?}");

    // A permanent error stops immediately — no delays at all.
    let (r, d) = run(&policy, |_| Err::<&str, _>(Err_::Permanent));
    println!("permanent    -> {r:?}, delays = {d:?}");

    // Retry-After (5s) wins over the computed 200ms backoff.
    let (r, d) = run(&policy, |n| if n == 0 { Err(Err_::RateLimited { after: 5 }) } else { Ok("ok") });
    println!("rate-limited -> {r:?}, delays = {d:?}");
}
```

```text
transient x3 -> Ok("ok"), delays = [200ms, 400ms, 800ms]
permanent    -> Err("Permanent"), delays = []
rate-limited -> Ok("ok"), delays = [5s]
```

Read the three lines against the three rules. The transient run doubles the wait each time — `200ms`, `400ms`, `800ms` — classic exponential backoff (and had it kept failing, the fourth delay would have been `1600ms`, then it would clamp to the `10s` cap). The permanent run does **zero** waiting and **zero** extra attempts: `is_transient()` returned false, so the loop returned the error on the first failure. And the rate-limited run waited `5s` — the server's `Retry-After`, not the `200ms` the exponential formula would have produced.

## The real thing, verbatim

The shipped `with_retry` lives in `etl-core` (`retry.rs`) and is the exact same loop, made async so it can actually `sleep`:

```rust,ignore
// crates/etl-core/src/retry.rs  (needs tokio; not playground-safe)
pub async fn with_retry<F, Fut, T>(policy: RetryPolicy, mut op: F) -> Result<T, EtlError>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T, EtlError>>,
{
    let mut attempt: u32 = 0;
    loop {
        match op().await {
            Ok(value) => return Ok(value),
            Err(err) => {
                attempt += 1;
                if attempt >= policy.max_attempts || !err.is_transient() {
                    return Err(err);
                }
                // A server-requested backoff always wins; otherwise exponential.
                let backoff = err.retry_after().unwrap_or_else(|| {
                    let factor = 2u32.saturating_pow(attempt - 1);
                    (policy.base_delay * factor).min(policy.max_delay)
                });
                sleep(backoff).await;
            }
        }
    }
}
```

Line for line, it is the toy: increment `attempt`, bail on `attempt >= max_attempts || !is_transient()`, compute `retry_after().unwrap_or_else(exponential)`, `sleep`. `op` is an `FnMut` returning a fresh `Future` each call — it must be re-runnable, because retrying means calling it again. `RetryPolicy` defaults to `max_attempts: 4`, `base_delay: 200ms`, `max_delay: 10s`.

## Deterministic tests via paused time

There is an obvious problem testing this: a test for "backs off 5 seconds on `Retry-After`" cannot actually sleep 5 real seconds — CI would crawl and the assertion would be flaky. The fix is tokio's **paused clock**. Under `#[tokio::test(start_paused = true)]`, time does not advance on its own; a `sleep` completes *instantly* as far as the wall clock is concerned, but `tokio::time::Instant::now()` still advances by the slept duration. So you get to assert on elapsed *virtual* time without waiting:

```rust,ignore
// crates/etl-core/src/retry.rs — the shipped test, verbatim
#[tokio::test(start_paused = true)]
async fn honors_retry_after() {
    let calls = Arc::new(AtomicU32::new(0));
    let c = calls.clone();
    let start = tokio::time::Instant::now();
    let out = with_retry(policy(), || {
        let c = c.clone();
        async move {
            let n = c.fetch_add(1, Ordering::SeqCst);
            if n == 0 {
                Err(EtlError::RateLimited { retry_after_secs: 5 })
            } else {
                Ok(())
            }
        }
    })
    .await;
    assert!(out.is_ok());
    // The 5s Retry-After beats the 100ms base backoff.
    assert!(start.elapsed() >= Duration::from_secs(5));
}
```

The test runs in microseconds yet proves the loop waited five (virtual) seconds. That is the whole point of paused time: a timing-dependent behavior becomes a deterministic, fast assertion. The sibling tests `retries_transient_then_succeeds` (fails twice, then succeeds — three calls) and `gives_up_on_permanent_immediately` (one call, no retry) use the same paused-clock harness with an `AtomicU32` counting how many times `op` actually ran.

<div class="callout trap">
<span class="callout-label">Retry a permanent error and you have built a slower way to fail</span>
The single most common retry bug is retrying everything. A loop that retries a <code>404</code> or a parse error does not fix it — it just does the same doomed request three more times, adds the full backoff delay to every hard failure, and buries the real cause under a pile of identical log lines. Retrying must be gated on <code>is_transient()</code>; that gate is exactly why Part I made the transient/permanent split a first-class method on the error type instead of an afterthought.
</div>

## Where the classification comes from

`with_retry` never inspects an HTTP status; it only reads `is_transient()` and `retry_after()` off the `EtlError` it is handed. The mapping from status code to error variant happens *before* the retry loop ever sees it, in `classify_status` (from Part II's `http.rs`): `429 → RateLimited { retry_after_secs }`, `401|403 → Auth`, other `4xx → Permanent`, `5xx → Network`. And `RateLimited` is where `retry_after_secs` is populated — parsed from the `Retry-After` header. So the chain is: response status → `classify_status` → typed `EtlError` → `with_retry` reads `is_transient()`/`retry_after()`. Each piece decides one thing, once.

## Questions to lock

1. What are the two conditions that make `with_retry` stop and return the error instead of retrying?
2. Trace the backoff sequence for `base = 200ms`, `cap = 1000ms` across attempts 1–5. Where does the cap first bite?
3. When a `429` carries `Retry-After: 5`, what does `with_retry` sleep — the header value or the computed exponential backoff — and why is that the right call?
4. Why can a test for "waits 5 seconds" run in microseconds under `#[tokio::test(start_paused = true)]`? What does the paused clock do to `sleep` versus to `Instant::now()`?
5. `with_retry` never looks at an HTTP status code. What does it look at instead, and which earlier function is responsible for turning a status into that?

The graded quiz for this arc is at the end of the part, on the [Concept-Check: Pagination](./quiz.md) page.
