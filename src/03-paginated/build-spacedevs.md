# Build: The TheSpaceDevs Source — Pagination

> **Maps to:** Task 3.3. **Kind:** Build — spec and test names only. No implementation.

## Objective

Build `SpaceDevsSource` (E — a paginated GET that follows the `next` cursor to the end) and `SpaceDevsTransform` (T — deeply nested Launch Library 2 JSON into `Row::Launch`, including the prose `description` that the RAG arc will later embed). Every test hits a `wiremock` server, never the live network.

This is the third concrete `Source`/`Transform` pair. It is also the first source whose Extract is a *loop* of requests rather than a single one, and the first to lean on the `with_retry` middleware built in the companion page.

## Scaffold

**Create:** `crates/etl-sources/src/spacedevs.rs`; add `pub mod spacedevs;` and re-exports to `crates/etl-sources/src/lib.rs`.

**Dependencies:** no manifest changes — `reqwest`, `async-trait`, `chrono`, `serde`, `serde_json`, `tokio`, and (dev) `wiremock` + `pretty_assertions` were all declared when you created `etl-sources` in Part II.

> `reqwest`, `wiremock`, and `chrono` are **not** on the Rust playground; build and test this in your workspace with `cargo test -p etl-sources spacedevs`.

**Expected result:** `cargo test -p etl-sources spacedevs` → all four tests green.

## The spec (givens)

**`SpaceDevsSource`** — fields `base_url: String` and `policy: RetryPolicy`; constructor `new(base_url: impl Into<String>)` that stores the base URL and `RetryPolicy::default()`. It implements `Source`:

- `name(&self)` returns `"spacedevs"`.
- `extract`:
  - Trim a trailing `/` off `base_url`. Seed the cursor with `{base}/2.2.0/launch/?limit=100&mode=detailed`.
  - Keep a `HashSet<String>` of visited URLs. Before fetching each URL, `insert` it; if `insert` returns `false` (already seen), return `EtlError::Parse("pagination cycle at {url}")`.
  - Fetch each page **through `with_retry(self.policy, …)`**: inside the closure, `reqwest::Client::get(&url).send()` (transport error → `EtlError::Network`); on a non-success status return `classify_status(status, headers)`; on success read the body text (`resp.text()` error → `EtlError::Network`).
  - Deserialize the body into a `Page` (parse error → `EtlError::Parse`). Set the cursor to `page.next`. Push one `RawRecord { source: "spacedevs", payload: <body>, fetched_at: Utc::now() }` per page.
  - Loop until the cursor is `None`; return the `Vec<RawRecord>`.

**`SpaceDevsTransform`** — a unit struct implementing `Transform`:

- Deserialize `raw.payload` into a `Page` (parse error → `EtlError::Parse`).
- For each launch in `page.results`: parse `net` as `DateTime<Utc>` (parse error → `EtlError::Parse` naming the bad value); derive `payloads` and `description` from the optional `mission` (a missing mission ⇒ empty `payloads`, empty `description`; a mission's absent `name` ⇒ empty `payloads`, absent `description` ⇒ `String::new()`); take `provider` from `launch_service_provider.name` (absent ⇒ `String::default()`). Push a `Row::Launch(Launch { id, name, net, provider, payloads, description })`.

**The deserialization structs (given — the LL2 shape you model, ignoring the dozens of fields you do not need):**

```text
Page   { next: Option<String>, results: Vec<RawLaunch> }
RawLaunch { id: String, name: String, net: String,
            launch_service_provider: Option<Provider>, mission: Option<Mission> }
Provider  { name: String }
Mission   { name: Option<String>, description: Option<String> }
```

Note `next` is `Option<String>` — serde maps JSON `null` to `None`, which is exactly the loop's termination signal. And serde ignores every LL2 field you did not declare, so you model only the five you use.

## Concepts exercised

- Cursor pagination: the follow-`next` loop and the visited-set cycle guard (*Pagination & Following `next`*).
- `with_retry` wrapping each page fetch (*Retry with Backoff*).
- `classify_status` mapping a non-2xx response onto the error taxonomy (Part II's `http.rs`).
- Deserializing deeply nested, mostly-ignored JSON into a flat `Row::Launch`.

## Test names to hit

Use a helper that emits one page of JSON with a parameterized `next` (a `null` when you pass `None`), carrying one launch with a nested `mission` and `launch_service_provider`:

- `follows_next_until_null` — the core pagination test. Mount two mocks: `path("/2.2.0/launch/")` returns a page whose `next` points at `{server.uri()}/page2`; `path("/page2")` returns a page with `next: null`. Point `SpaceDevsSource::new(server.uri())` at it; assert `extract().await` yields **2** `RawRecord`s.
- `rejects_pagination_cycle` — the first page's `next` points **back at itself** (`{server.uri()}/2.2.0/launch/`). Assert `extract().await` is `Err(EtlError::Parse(_))` — the cycle guard, not an infinite loop.
- `http_error_is_network_error` — the mock returns `500`; assert `extract().await` is `Err(EtlError::Network(_))` (5xx is transient).
- `parses_nested_launch_json` — the core T test, no server. Hand `SpaceDevsTransform` a `RawRecord` whose payload is one page; assert one `Row::Launch` with `provider == "SpaceX"`, `payloads == ["Starlink G-99"]`, and a `description` containing the mission prose.

<div class="callout trap">
<span class="callout-label">The silent-404 trap (still with us)</span>
If your matcher chain is wrong — a typo in <code>path("/2.2.0/launch/")</code>, a missing trailing slash — wiremock matches nothing and answers <code>404</code>. <code>classify_status</code> maps that to <code>EtlError::Permanent</code>, and <code>follows_next_until_null</code> fails looking like a broken source when the <em>mock</em> was never hit. When a pagination test fails on an unexpected permanent error, suspect the matcher before the loop.
</div>

## The build loop (you drive)

1. **Transform first — it needs no server.** Write `parses_nested_launch_json` against a hand-written page JSON. Predict: what should `payloads` and `description` be when `mission` is `null`? Decide before implementing.
2. **`follows_next_until_null`.** Stand up two mocks; watch the loop consume both pages and stop on the `null`. Predict what happens if you forget to set the cursor from `page.next` (it stops after one page).
3. **`rejects_pagination_cycle`.** Point the first page's `next` at itself. Confirm the guard fires as `EtlError::Parse` rather than hanging the test.
4. **`http_error_is_network_error`.** A `500` mock; confirm the 5xx maps to transient `Network`.
5. **Run green, commit.**

## Done when

`cargo test -p etl-sources spacedevs` is green. `SpaceDevsSource` and `SpaceDevsTransform` now satisfy `Source` and `Transform` from `etl-core`, with pagination, retry, and cycle-safety all exercised against a mock. Three sources now feed one uniform `Row` enum; the orchestrator in Part V will schedule all three without knowing which is which.
