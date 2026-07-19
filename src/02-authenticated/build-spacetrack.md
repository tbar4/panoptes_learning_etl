# Build: The Space-Track Source

> **Maps to:** Tasks 2.1 (token-bucket limiter) + 2.2 (session auth + the Space-Track source). **Kind:** Build — spec and test names only. No implementation.

## Objective

Build the first **authenticated, rate-limited** source. Two files:

1. **`limiter.rs`** — the hand-rolled `TokenBucket` from the rate-limiting concept: burst up to `capacity`, refill at a fixed rate, `acquire` sleeps until a token is free.
2. **`spacetrack.rs`** — `Credentials` (from `new` or the environment), `SpaceTrackSource` (E — log in, keep the cookie, run the CDM query, rate-limited), and `SpaceTrackTransform` (T — Space-Track CDM JSON into `Row::Conjunction`).

By the end, `SpaceTrackSource` implements `Source` and `SpaceTrackTransform` implements `Transform` — the same two traits CelesTrak implemented, now with a login handshake and a limiter in front. Every test hits a `wiremock` server or runs under paused tokio time; nothing touches the live network or a real Space-Track account.

## Scaffold

**Create:** `crates/etl-sources/src/limiter.rs` and `crates/etl-sources/src/spacetrack.rs`; declare both modules in `lib.rs`.

**Dependencies:** none new — `reqwest`, `async-trait`, `chrono`, `serde`/`serde_json`, `tokio`, and the dev-only `wiremock` + `pretty_assertions` were all declared when you created `etl-sources` for CelesTrak. The paused-clock tests need tokio's `test-util` feature, which the crate's `[dev-dependencies]` already enables (`tokio = { workspace = true, features = ["test-util"] }`).

> `reqwest`, `wiremock`, `chrono`, and `tokio` time are **not** on the Rust playground. Build and test in the workspace with `cargo test -p etl-sources`.

**Expected result:** `cargo test -p etl-sources` → all limiter and spacetrack tests green (alongside the CelesTrak tests from Part II).

## Part A — the token bucket (`limiter.rs`)

The rate-limiting concept page is the reference; this is the interface to build.

**`TokenBucket`** — a `#[derive(Clone)]` struct with fields `capacity: f64`, `refill_per_sec: f64`, and `state: Arc<Mutex<State>>`, where `State { tokens: f64, last: Instant }`. The `Arc<Mutex<…>>` is what lets a cloned bucket, shared across tasks, refer to one shared pool of tokens.

- `new(capacity: u32, refill_per_sec: f64) -> Self` — start the bucket **full** (`tokens = capacity`) with `last = Instant::now()`. (The `u32` capacity is stored as `f64`.)
- `async fn acquire(&self)` — the loop from the concept page: lock the state, refill by `elapsed × refill_per_sec` capped at `capacity`, and if at least one token is available, spend it and **return**; otherwise compute the wait for one token to refill, **drop the lock**, `tokio::time::sleep(wait).await`, and loop.

**The rule that matters (from the concept page):** compute the wait *inside* the lock, then release the guard *before* the `.await`. Holding a `MutexGuard` across `tokio::time::sleep` would block every other caller for the whole sleep.

**Test names to hit** (both `#[tokio::test(start_paused = true)]` so virtual time fast-forwards through the waits):

- `bucket_allows_burst_up_to_capacity` — `TokenBucket::new(3, 0.5)`; `acquire().await` three times; assert `start.elapsed()` is under ~50 ms. A full bucket serves the burst with no real waiting.
- `bucket_blocks_when_empty_then_refills` — `TokenBucket::new(1, 1.0)`; `acquire().await` twice. The second call finds the bucket empty and must wait ~1 s for a refill; assert `start.elapsed() >= 900 ms`. Under the paused clock this passes in microseconds of wall time.

<div class="callout predict">
<span class="callout-label">Predict before you run</span>
In <code>bucket_blocks_when_empty_then_refills</code>, the bucket has capacity 1 and refills 1 token/sec. Before running, say what <code>start.elapsed()</code> will report in <em>virtual</em> time, and what it would cost in <em>real</em> wall-clock time. If your mental model says "about a second of real time," re-read the paused-clock section — the whole point is that it does not.
</div>

## Part B — `Credentials` + `SpaceTrackSource` (E) (`spacetrack.rs`)

**`Credentials`** — a struct with public `user: String` and `pass: String`.

- `new(user: impl Into<String>, pass: impl Into<String>) -> Self`.
- `from_env() -> Result<Self, EtlError>` — read `SPACETRACK_USER` and `SPACETRACK_PASS`; if *either* is missing, return `EtlError::Auth("… not set")`. (Missing credentials are permanent — the retry loop must not retry them.)

**`SpaceTrackSource`** — fields `base_url: String`, `creds: Credentials`, `limiter: TokenBucket`; constructor `new(base_url, creds, limiter)`. It implements `Source`:

- `name(&self)` returns `"spacetrack"`.
- `extract` performs the handshake from the concept page, spending a token before **each** request:
  1. `self.limiter.acquire().await`, then **POST** `{base}/ajaxauth/login` with the credentials as **form fields** `identity` and `password` (trim a trailing `/` off `base_url`). A transport error maps to `EtlError::Network`.
  2. On a non-success login status, return `classify_status(status, headers)` — so `401`/`403` become `EtlError::Auth` (permanent) while `5xx`/`429` stay transient.
  3. Read the `SET_COOKIE` header, keep only the first `;`-separated segment; if absent, `EtlError::Auth("login returned no session cookie")`.
  4. `self.limiter.acquire().await`, then **GET** `{base}/basicspacedata/query/class/cdm_public/orderby/TCA/format/json` with that string in the `COOKIE` header. Non-success status → `classify_status`.
  5. On success, read the body text into **one** `RawRecord { source: "spacetrack", payload, fetched_at: Utc::now() }`; return a one-element `Vec`.

**Test names to hit** (wiremock):

- `login_then_query_uses_cookie` — the core E test. Mount two mocks: a `POST /ajaxauth/login` returning `200` with `set-cookie: chocolatechip=abc123; Path=/`, and a `GET` on the CDM query path guarded by `header_exists("cookie")`, returning `200` with a CDM JSON body, marked `.expect(1)`. Point `SpaceTrackSource::new(server.uri(), Credentials::new("u", "p"), bucket())` at it; assert one `RawRecord`, and that its payload contains a known NORAD id. The `.expect(1)` verifies on drop that the *cookie-guarded* query was actually hit — proof the cookie was forwarded.
- `login_failure_is_auth_error` — mount only a `POST /ajaxauth/login` returning `401`; assert `extract().await` is `Err(EtlError::Auth(_))`.

<div class="callout trap">
<span class="callout-label">The silent-404 trap, now with a cookie twist</span>
The query mock is guarded by <code>header_exists("cookie")</code>. If your <code>extract</code> forgets to attach the <code>COOKIE</code> header — or attaches it under the wrong name — the query mock matches <em>nothing</em>, wiremock answers <code>404</code>, and <code>classify_status</code> turns that into <code>EtlError::Permanent</code>. The test then fails with a "permanent 404" that looks like the server rejected you, when really the <em>mock</em> was never hit because the cookie never arrived. When <code>login_then_query_uses_cookie</code> fails on an unexpected 404, suspect the forwarded cookie before you suspect the source.
</div>

## Part C — `SpaceTrackTransform` (T) (`spacetrack.rs`)

**`SpaceTrackTransform`** — a unit struct implementing `Transform`. Space-Track serializes a **CDM** (Conjunction Data Message) as JSON where **every number arrives as a string**. Deserialize into a private `RawCdm` with `#[serde(rename = "…")]` on each field:

```text
TCA       -> tca:        String   (RFC-3339-ish, e.g. "2026-07-20T12:00:00.000000")
MIN_RNG   -> min_rng_m:  String   (miss distance in METERS, as a string)
PC        -> pc:         Option<String>   (collision probability; may be absent/empty)
SAT_1_ID  -> sat1:       String
SAT_2_ID  -> sat2:       String
```

`transform` parses the payload as `Vec<RawCdm>` (a parse failure → `EtlError::Parse`), then for each CDM builds a `Conjunction`:

- `tca`: parse with `NaiveDateTime::parse_from_str(&c.tca, "%Y-%m-%dT%H:%M:%S%.f")` then `.and_utc()`; a bad value → `EtlError::Parse`.
- `miss_distance_km`: parse `min_rng_m` as `f64` and **divide by 1000.0** (meters → kilometers); unparseable → `EtlError::Parse`.
- `collision_probability`: if `pc` is present and non-empty, parse it as `f64`; otherwise default to `0.0`.
- `primary` / `secondary`: `c.sat1.parse()` and `c.sat2.parse()` into `NoradId` — the newtype's `FromStr` returns `EtlError::Parse` on junk, so `?` lifts it straight into the taxonomy.
- `id`: `format!("{}_{}_{}", primary.0, secondary.0, tca.format("%Y%m%dT%H%M%S"))`. (This is a readable composite key; Part IV replaces it with a content-addressed hash.)

Push each as `Row::Conjunction(...)`.

**Test name to hit:**

- `transform_parses_cdm_to_conjunction` — hand `SpaceTrackTransform` a `RawRecord` whose payload is a one-element CDM array (`MIN_RNG` `"550.5"`, `PC` `"0.0001"`, sats `"25544"` / `"48274"`). Assert one row, a `Row::Conjunction` with `primary == NoradId(25544)`, `secondary == NoradId(48274)`, `miss_distance_km ≈ 0.5505` (note the ÷1000), and `collision_probability ≈ 0.0001`.

<div class="callout trap">
<span class="callout-label">Numbers arrive as strings — and meters, not kilometers</span>
Two decoding traps hide in the CDM. First, every numeric field is JSON <em>string</em>, so <code>MIN_RNG</code> is <code>"550.5"</code>, not <code>550.5</code> — deserialize into <code>String</code> and parse yourself, or serde rejects the document. Second, <code>MIN_RNG</code> is in <strong>meters</strong> while <code>Conjunction.miss_distance_km</code> is kilometers; forget the <code>/ 1000.0</code> and every miss distance is off by a factor of a thousand — a test that <code>miss_distance_km ≈ 0.5505</code> (not <code>550.5</code>) is what catches it.
</div>

## The build loop (you drive)

1. **Token bucket first.** Write `bucket_allows_burst_up_to_capacity`, predict "no wait," implement `new` + the `acquire` loop, run green.
2. **`bucket_blocks_when_empty_then_refills`.** Predict the virtual elapsed time before running; confirm the paused clock makes it instant in real time. If it actually hangs, you are almost certainly holding the lock across the `.await` — release the guard before sleeping.
3. **`Credentials::from_env`.** Small and pure; a missing var is `EtlError::Auth`.
4. **`login_then_query_uses_cookie`.** Stand up both mocks; **predict** what happens if you drop the `COOKIE` header (silent 404 via the `header_exists` guard), then confirm. Get the happy path green.
5. **`login_failure_is_auth_error`.** One mock returning `401`; assert `EtlError::Auth`.
6. **`transform_parses_cdm_to_conjunction`.** Build the `RawCdm` structs, wire the meters→km and string→number conversions. If the miss distance is 1000× too big, you skipped the divide.
7. **Run green, commit.**

## Done when

`cargo test -p etl-sources` is green. You now have a second concrete `Source`/`Transform` pair — authenticated, rate-limited, emitting the `Conjunction` keystone — behind the exact same traits as CelesTrak. The orchestrator will not be able to tell them apart, which is the entire point. Next you build the ledger that lets a run remember it happened.
