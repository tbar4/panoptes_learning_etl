# Concept: Rate Limiting with a Token Bucket

> **Kind:** Concept. **Seam introduced this arc:** a client-side rate limiter — a token bucket every request must pass through, and the paused-clock testing technique that makes it deterministic.

## The idea

Space-Track will ban a client that hammers it. The etiquette is explicit: keep your request rate low, and do not burst without limit. So before *every* request — login included — the source must ask permission from a **rate limiter** that only lets requests through at a controlled pace.

The classic algorithm for this is a **token bucket**, and it is worth building by hand because the whole idea fits in a paragraph:

- The bucket holds up to `capacity` tokens and starts full.
- It **refills continuously** at `refill_per_sec` tokens per second, never exceeding `capacity`.
- To make a request you must **spend one token**. If the bucket has one, you take it and go immediately. If it is empty, you **wait** exactly long enough for one token to refill, then go.

Two behaviors fall straight out of that. A **burst** is fine up to `capacity`: a full bucket serves that many requests back-to-back with no waiting. After the burst, you are throttled to the steady refill rate — one request every `1 / refill_per_sec` seconds. The bucket smooths a bursty caller down to a rate the server tolerates, while still allowing short spikes.

## Why hand-rolled, not `governor`

There is a perfectly good crate (`governor`) that does this. We build it by hand anyway, for the same reason we hand-rolled the TLE parser: the token bucket *is* the lesson. Writing the refill arithmetic yourself — `tokens = min(capacity, tokens + elapsed × rate)` — is how you actually understand what a rate limiter guarantees and what it does not. It is thirty lines. Reach for `governor` in production if you like; here, the thirty lines are the point.

## A runnable toy: the bucket with a logical clock

The real limiter sleeps on `tokio::time`, which does not run on the playground. So here is the *same arithmetic* with the clock made explicit — we pass in "the current logical time" instead of sleeping, which also makes every step visible. `capacity` tokens, refilling at `refill_per_sec`:

```rust
/// A token bucket driven by a *logical* clock we pass in — no real sleeping, so
/// the arithmetic is visible and deterministic. `capacity` tokens sit in the
/// bucket; it refills continuously at `refill_per_sec`, never past `capacity`.
struct Bucket {
    tokens: f64,
    capacity: f64,
    refill_per_sec: f64,
    last: f64, // logical seconds at which we last refilled
}

impl Bucket {
    fn new(capacity: f64, refill_per_sec: f64) -> Self {
        Self { tokens: capacity, capacity, refill_per_sec, last: 0.0 }
    }

    /// Try to take one token at logical time `now`.
    /// `Ok(())` if one was free (and consumed); `Err(wait)` = seconds until one is.
    fn acquire(&mut self, now: f64) -> Result<(), f64> {
        let elapsed = now - self.last;
        self.tokens = (self.tokens + elapsed * self.refill_per_sec).min(self.capacity);
        self.last = now;
        if self.tokens >= 1.0 {
            self.tokens -= 1.0;
            Ok(())
        } else {
            Err((1.0 - self.tokens) / self.refill_per_sec)
        }
    }
}

fn main() {
    // capacity 3, refills half a token per second (one every 2s).
    let mut b = Bucket::new(3.0, 0.5);

    // A full bucket serves a burst of 3 at t=0 with no waiting.
    for i in 1..=3 {
        println!("t=0  req {i}: {:?}", b.acquire(0.0));
    }
    // The 4th request at t=0 finds the bucket empty; it must wait.
    println!("t=0  req 4: {:?}", b.acquire(0.0));
    // After 2 logical seconds exactly one token has refilled.
    println!("t=2  req 5: {:?}", b.acquire(2.0));
}
```

```text
t=0  req 1: Ok(())
t=0  req 2: Ok(())
t=0  req 3: Ok(())
t=0  req 4: Err(2.0)
t=2  req 5: Ok(())
```

Every line of that output is the algorithm showing its work:

- **Requests 1–3 at `t=0` all pass.** The bucket started with 3 tokens; the burst spends them with no wait. That is the `capacity` guarantee.
- **Request 4 at `t=0` fails with `Err(2.0)`.** The bucket is empty and no time has passed, so zero tokens have refilled. The wait is `(1.0 − 0.0) / 0.5 = 2.0` seconds — the time to refill one token at half a token per second.
- **Request 5 at `t=2` passes.** Two logical seconds elapsed, refilling `2 × 0.5 = 1.0` token — exactly enough. The `.min(self.capacity)` never comes into play here, but it is the guard that stops a long idle period from letting the bucket overflow past 3 and permitting an unbounded burst later.

The real `TokenBucket::acquire` does not return a wait for you to handle — it **sleeps** that duration itself and loops, so from the caller's side `acquire().await` simply returns once a token is yours. Same arithmetic; the loop hides the waiting.

## The shipped limiter, and the paused-clock test

The real thing (in `crates/etl-sources/src/limiter.rs`) is the same math wrapped for concurrency and real time: the state lives behind an `Arc<Mutex<…>>` so a cloned bucket shared across tasks stays consistent, and the wait is a real `tokio::time::sleep`. It is `rust,ignore` — `tokio` async is not playground-safe:

```rust,ignore
// deps: tokio = { version = "1", features = ["full"] }
pub async fn acquire(&self) {
    loop {
        let wait = {
            let mut s = self.state.lock().await;
            let now = Instant::now();
            let elapsed = now.duration_since(s.last).as_secs_f64();
            s.tokens = (s.tokens + elapsed * self.refill_per_sec).min(self.capacity);
            s.last = now;
            if s.tokens >= 1.0 {
                s.tokens -= 1.0;
                return; // a token was free — go now
            }
            // Not enough yet: compute how long one token takes to refill.
            Duration::from_secs_f64((1.0 - s.tokens) / self.refill_per_sec)
        }; // lock released here, BEFORE we sleep
        tokio::time::sleep(wait).await;
    }
}
```

Now the problem that technique solves. This limiter is *about waiting*. A test for "the bucket blocks when empty, then refills after one second" would seem to require the test to actually sleep a second — which makes the suite slow and, worse, **flaky**: on a loaded CI box the real elapsed time drifts, and an assertion like "took at least 900 ms" fails at random.

The fix is `#[tokio::test(start_paused = true)]`. Under a paused clock, tokio's timer is **virtual**. When every task in the test is parked on a `sleep`, tokio does not sit and wait for the wall clock — it **auto-advances** the virtual clock straight to the next timer's deadline. A one-second `sleep` therefore completes *instantly* in real time, yet `Instant::now()` inside the test reports that a full virtual second elapsed. The waiting is real to the code and free to the clock:

```rust,ignore
#[tokio::test(start_paused = true)]
async fn bucket_blocks_when_empty_then_refills() {
    let bucket = TokenBucket::new(1, 1.0);
    let start = Instant::now();
    bucket.acquire().await; // consumes the one starting token
    bucket.acquire().await; // empty: must wait ~1s for a refill
    // Virtual time advanced ~1s, but the test ran in microseconds.
    assert!(start.elapsed() >= Duration::from_millis(900));
}
```

<div class="callout">
<span class="callout-label">Why paused time is not cheating</span>
The code under test is completely unchanged — it calls the same <code>tokio::time::sleep</code> it uses in production. Only the <em>clock</em> is swapped for a virtual one that fast-forwards through idle waits. So the test exercises the real blocking-and-refilling logic and the real timing arithmetic; it just does not spend real seconds doing it. You get a deterministic assertion on timing behavior with a test that finishes in microseconds.
</div>

<div class="callout trap">
<span class="callout-label">Release the lock before you sleep</span>
Look at where the closure ends in the shipped <code>acquire</code>: the <code>MutexGuard</code> is dropped at the closing brace, <em>before</em> <code>tokio::time::sleep(wait).await</code>. If you instead held the lock across the <code>.await</code>, a task that has to wait would keep every <em>other</em> task blocked on the mutex for the entire sleep — serializing all callers behind the slowest one and defeating the point of a shared bucket. Compute the wait under the lock; sleep without it. Holding a lock across an <code>.await</code> is one of the most common async performance bugs, and here it would quietly turn a rate limiter into a global stop-the-world.
</div>

## The mapping onto the shipped code

- **`Bucket` fields** ↔ **`TokenBucket`**: `capacity` and `refill_per_sec` are the same; the toy's `tokens` + `last` live inside the real bucket's `Arc<Mutex<State>>` so clones share one bucket.
- **`Bucket::new(3.0, 0.5)`** ↔ **`TokenBucket::new(30, 0.5)`** — the Space-Track default: 30 tokens of burst, refilling one every two seconds. The real constructor takes `capacity: u32` and converts to `f64` internally.
- **the toy's `Err(wait)` you handle** ↔ the real `acquire` **sleeps `wait` itself and loops**, so the caller just `.await`s.
- **passing `now` explicitly** ↔ `Instant::now()` under a **paused** clock in tests, real wall-clock time in production.

Every request in `SpaceTrackSource::extract` — the login POST *and* the CDM GET — begins with `self.limiter.acquire().await`. A token is spent per request, login included, which is exactly the etiquette Space-Track asks for.

## Questions to lock

1. State the token-bucket rule in three parts (capacity, refill, spend-or-wait). What two behaviors — one about bursts, one about steady state — does it produce?
2. In the toy, request 4 returned `Err(2.0)`. Where does the `2.0` come from, arithmetically?
3. What does `.min(self.capacity)` prevent? Give a scenario where dropping it would let a caller exceed the intended rate.
4. What does `#[tokio::test(start_paused = true)]` change about time, and why does that make a timing test both fast and deterministic rather than flaky?
5. Why must the shipped `acquire` drop the mutex guard *before* it sleeps? What goes wrong for other tasks if it does not?

The graded quiz for this arc is at the end of the part, on the [Concept-Check: Authentication](./quiz.md) page.
