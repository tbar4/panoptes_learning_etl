# Build: The CelesTrak Source — Extract + Transform

> **Maps to:** Tasks 1.1 (TLE parser) + 1.2 (CelesTrak source). **Kind:** Build — spec and test names only. No implementation.

## Objective

Create the `etl-sources` crate and make the first real pipeline stages: the **TLE parser** (the heart of T), a small **HTTP status classifier**, the **`CelestrakSource`** (E — an open GET returning TLE text), and the **`CelestrakTransform`** (T — 3-line TLE blocks into `Row::SpaceObject`). Every test hits a `wiremock` server, never the live network.

By the end, `CelestrakSource` implements `Source` and `CelestrakTransform` implements `Transform` — the two traits you defined in `etl-core`, now with a real API behind them.

## Scaffold

**Create the crate:** `crates/etl-sources/Cargo.toml`, `src/lib.rs`, `src/tle.rs`, `src/http.rs`, `src/celestrak.rs`. Add `"crates/etl-sources"` to the workspace `members`.

**`crates/etl-sources/Cargo.toml`** — depends only on `etl-core` plus the machinery a real HTTP source forces on us:

```toml
[dependencies]
etl-core = { path = "../etl-core" }
serde = { workspace = true }
serde_json = { workspace = true }
chrono = { workspace = true }        # DateTime for the epoch decode + fetched_at
async-trait = { workspace = true }   # Source::extract is async
reqwest = { workspace = true }       # the actual HTTP GET
tokio = { workspace = true }

[dev-dependencies]
pretty_assertions = { workspace = true }
tokio = { workspace = true, features = ["test-util"] }
wiremock = { workspace = true }      # the mock server every source test hits
```

> `reqwest`, `wiremock`, and `chrono` are **not** on the Rust playground; build and test this in your workspace with `cargo test -p etl-sources`, exactly as the async arc of the first course did.

**Expected result:** `cargo test -p etl-sources` → all TLE, http, and celestrak tests green.

## Part A — the TLE parser (`tle.rs`)

The concept page (*Parsing the TLE Format*) is the reference; this is the interface to build. **Public functions:**

```text
pub fn tle_checksum(line: &str) -> u8
pub fn decode_epoch(year2: &str, day_frac: &str) -> Result<DateTime<Utc>, EtlError>
pub fn tle_norad(line1: &str) -> Result<u32, EtlError>
pub fn tle_intl_designator(line1: &str) -> Result<String, EtlError>
pub fn parse_tle(line1: &str, line2: &str) -> Result<OrbitalElements, EtlError>
```

Note `parse_tle` takes **two** lines (the name is handled by the transform, not here) and returns bare `OrbitalElements`.

**The column map (given — the TLE wire format).** These are already converted from the spec's 1-based inclusive columns to Rust's 0-based half-open slices, so you write them once and never re-derive:

| Field | Line | 1-based cols | Rust slice | Notes |
| --- | --- | --- | --- | --- |
| NORAD id | 1 or 2 | 3–7 | `[2..7]` | both lines must agree |
| intl designator | 1 | 10–17 | `[9..17]` | raw, e.g. `"98067A"` |
| epoch year | 1 | 19–20 | `[18..20]` | 2 digits, pivoted |
| epoch day+frac | 1 | 21–32 | `[20..32]` | day-of-year, 1-based |
| inclination | 2 | 9–16 | `[8..16]` | degrees |
| RAAN | 2 | 18–25 | `[17..25]` | degrees |
| eccentricity | 2 | 27–33 | `[26..33]` | **implied** leading `0.` |
| arg of perigee | 2 | 35–42 | `[34..42]` | degrees |
| mean anomaly | 2 | 44–51 | `[43..51]` | degrees |
| mean motion | 2 | 53–63 | `[52..63]` | rev/day |
| checksum digit | both | 69 | `.nth(68)` | mod-10 |

**Rules to encode (all covered on the concept page):**

- `tle_checksum`: sum over the first **68** chars; digit → its value, `'-'` → 1, everything else → 0; result mod 10.
- Epoch year pivot: `yy < 57` ⇒ `2000 + yy`, else `1900 + yy`. Day-of-year is 1-based; the fractional part is a fraction of a day.
- Eccentricity: parse `format!("0.{}", cols_26_33.trim())`.
- `parse_tle` order: verify **both** checksums first → confirm both lines carry the **same NORAD id** (a mismatch is a misaligned block that checksums cannot catch) → then slice and parse each field.
- Slice with `line.get(a..b)` (returns `Option`), never `line[a..b]` (panics): a short line must become `EtlError::Parse`, not a crash.

**Test names to hit** (use the canonical ISS block as the fixture — both its checksums are `7`):

```text
L1 = "1 25544U 98067A   08264.51782528 -.00002182  00000-0 -11606-4 0  2927"
L2 = "2 25544  51.6416 247.4627 0006703 130.5360 325.0288 15.72125391563537"
```

- `checksum_matches_known_lines` — `tle_checksum(L1) == 7` and `tle_checksum(L2) == 7`.
- `rejects_bad_checksum` — flip the last digit of `L2`; `parse_tle(L1, &bad)` returns `Err(EtlError::Parse(_))`.
- `decodes_2008_epoch` — `decode_epoch("08", "264.51782528")` decodes to year 2008, month 9, day 20, hour 12.
- `parses_iss_tle_to_elements` — `parse_tle(L1, L2)`: inclination ≈ 51.6416, eccentricity ≈ 0.0006703, mean motion ≈ 15.72125391, `orbit_class()` == `Leo`.
- `reads_norad_and_designator` — `tle_norad(L1) == 25544`, `tle_intl_designator(L1) == "98067A"`.
- `rejects_misaligned_line_pair` — build a line 2 for a *different* catalog number **with a corrected checksum**, so only the cross-line id mismatch (not a checksum) can reject it; assert `Err(EtlError::Parse(_))`.

## Part B — the HTTP status classifier (`http.rs`)

Every source funnels non-success responses through one function so the transient/permanent decision is made **once**. **Public functions:**

```text
pub fn retry_after_secs(headers: &HeaderMap) -> u64   // parse Retry-After (secs); absent/unparseable => 1
pub fn classify_status(status: StatusCode, headers: &HeaderMap) -> EtlError
```

**Mapping (given):**

```text
429            -> EtlError::RateLimited { retry_after_secs }   (transient, carries backoff)
401 | 403      -> EtlError::Auth(...)                          (permanent)
other 4xx      -> EtlError::Permanent(...)                     (permanent — retrying won't help)
5xx and else   -> EtlError::Network(...)                       (transient)
```

**Test names to hit:** `maps_429_to_rate_limited_with_backoff`, `maps_401_403_to_permanent_auth`, `maps_404_to_permanent_not_network`, `maps_5xx_to_transient_network`. Each asserts both the variant and (via `is_transient()`) its retry classification.

## Part C — `CelestrakSource` (E) + `CelestrakTransform` (T) (`celestrak.rs`)

**`CelestrakSource`** — fields `base_url: String`, `group: String`; constructor `new(base_url, group)`. It implements `Source`:

- `name(&self)` returns `"celestrak"`.
- `extract` builds the URL `{base}/NORAD/elements/gp.php?GROUP={group}&FORMAT=tle` (trim a trailing `/` off `base_url`), does a `reqwest::get`, maps any transport error to `EtlError::Network`, and on a non-success status returns `classify_status(status, headers)`. On success it reads the body text into one `RawRecord { source: "celestrak", payload, fetched_at: Utc::now() }` and returns a one-element `Vec`.

**`CelestrakTransform`** — a unit struct implementing `Transform`:

- Split `raw.payload` into lines, `trim_end` each, drop empties.
- Walk the lines in `chunks(3)`. A chunk that is not exactly 3 lines is a trailing partial block → `EtlError::Parse`.
- For each `(name, l1, l2)`: `parse_tle(l1, l2)?` for the elements, `tle_norad(l1)?` and `tle_intl_designator(l1)?` for the ids, `operator: None`, `name` trimmed. Push a `Row::SpaceObject(...)`.

**Test names to hit:**

- `extract_returns_raw_tle_text` — the core E test. Start a `MockServer`; mount a mock matching `method("GET")`, `path("/NORAD/elements/gp.php")`, `query_param("GROUP", "geo")`, `query_param("FORMAT", "tle")`, responding `200` with the ISS block as a body string. Point `CelestrakSource::new(server.uri(), "geo")` at it; assert one `RawRecord`, `source == "celestrak"`, payload contains `"25544"`.
- `transform_parses_tle_blocks_to_space_objects` — the core T test. Hand `CelestrakTransform` a `RawRecord` whose payload is the ISS block; assert one row, a `Row::SpaceObject` with `norad_id == NoradId(25544)`, `name == "ISS (ZARYA)"`, `intl_designator == "98067A"`.
- `server_error_is_transient_network` — mock returns `500`; `extract().await` is `Err(EtlError::Network(_))` and `is_transient()`.
- `rate_limit_is_retryable_with_backoff` — mock returns `429` with header `retry-after: 42`; the error is `EtlError::RateLimited { retry_after_secs: 42 }` and `is_transient()`.
- `not_found_is_permanent_not_retryable` — mock returns `404`; the error is `EtlError::Permanent(_)` and `!is_transient()`.

<div class="callout trap">
<span class="callout-label">The silent-404 trap (carried over from the async arc)</span>
If your matcher chain is wrong — a typo in the path, a missing <code>query_param</code> — wiremock matches nothing and answers <code>404</code>. Your <code>extract</code> then returns <code>EtlError::Permanent</code>, and <code>extract_returns_raw_tle_text</code> fails with a message that looks like your source is broken when really the <em>mock</em> was never hit. When an extract test fails on a 404 you did not expect, suspect the matcher before the source.
</div>

## The build loop (you drive)

1. **TLE checksum first.** Write `checksum_matches_known_lines`, predict both values are `7`, implement `tle_checksum`, run green.
2. **`rejects_bad_checksum`, then `decode_epoch`.** Predict `decodes_2008_epoch`'s year before running — is `08` → 1908 or 2008? The pivot answers it.
3. **`parse_tle`.** With checksum + epoch green, assemble the field slices. Run `parses_iss_tle_to_elements`; if a number is off by a factor of a thousand, look at the eccentricity's implied decimal; if off by one column, look at the slice bounds.
4. **`classify_status`.** Small and pure — four tests, four branches.
5. **`extract_returns_raw_tle_text`.** Stand up the mock, point the source at `server.uri()`, assert on the `RawRecord`. **Predict** what happens if you drop the `FORMAT=tle` matcher (silent 404), then confirm.
6. **`transform_parses_tle_blocks_to_space_objects`**, then the three error-taxonomy tests.
7. **Run green, commit.**

## Done when

`cargo test -p etl-sources` is green. `CelestrakSource` and `CelestrakTransform` now satisfy the `Source` and `Transform` traits from `etl-core` — the first concrete E and T. All that is missing is L: a place to write the rows.
