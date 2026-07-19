# Concept: Idempotent, Content-Addressed Writes

> **Kind:** Concept. **Seam installed this arc:** `content_id` / `content_key` and the `IdempotentSink` — the precondition for every retry, backfill, and scheduled re-run to come.

## The idea

The retry loop from the last concept has a consequence you have to design for: **the same work can run more than once.** A request that times out after the server already processed it gets retried; a scheduled backfill re-fetches a range you already have; a crashed run gets restarted from the top. If your Load stage blindly appends everything it is handed, every one of those produces **duplicate rows** in your output. Re-running the pipeline should be *safe* — running it twice should leave the same data as running it once. That property has a name: **idempotency**.

The way you get it is **content addressing**: give every row a stable id derived from *its own content*, then refuse to write a row whose id you have already written. Two rows with identical content get identical ids and collapse to one; any change to the content yields a different id and a distinct row. The id is not a database sequence or a random UUID — it is a **hash of the row's canonical bytes**, so it is reproducible across runs, across processes, across machines.

The real code uses SHA-256:

```rust,ignore
// crates/etl-core/src/content.rs  (needs the `sha2` crate; not playground-safe)
/// A deterministic id for a value: `sha256(kind ‖ 0x00 ‖ canonical)`, hex.
pub fn content_id(kind: &str, canonical: &str) -> String {
    let mut hasher = Sha256::new();
    hasher.update(kind.as_bytes());
    hasher.update([0u8]);
    hasher.update(canonical.as_bytes());
    format!("{:x}", hasher.finalize())
}
```

Two design choices in three lines. The **`kind` tag** ("row", "conjunction", "embed", …) is hashed in first, so two different entity types can never collide on an id even if their canonical strings happen to coincide. The **`0x00` separator** between `kind` and `canonical` stops the boundary from being ambiguous — without it, `kind="ab"`,`canonical="c"` and `kind="a"`,`canonical="bc"` would hash the same bytes. Small, deliberate, and load-bearing.

## A runnable toy: content ids and idempotent load

SHA-256 needs the `sha2` crate, which is not playground-safe, so this toy substitutes a tiny hand-rolled digest (FNV-1a). The hash function is not the lesson — the *properties* are: same content in, same id out; different content, different id; and a seen-set that makes a second load of the same rows a no-op.

```rust
use std::collections::HashSet;

// A toy stand-in for sha256: FNV-1a as a short hex digest. The real code uses
// sha2::Sha256; the only property that matters here is the one both share —
// same bytes in, same digest out; different bytes, a different digest.
fn digest(bytes: &[u8]) -> String {
    let mut h: u64 = 0xcbf29ce484222325;
    for &b in bytes {
        h ^= b as u64;
        h = h.wrapping_mul(0x100000001b3);
    }
    format!("{h:016x}")
}

// content_id(kind, canonical) = digest(kind ‖ 0x00 ‖ canonical). The kind tag
// plus a separator keep two entity types from colliding on equal canonical text.
fn content_id(kind: &str, canonical: &str) -> String {
    let mut buf = Vec::new();
    buf.extend_from_slice(kind.as_bytes());
    buf.push(0);
    buf.extend_from_slice(canonical.as_bytes());
    digest(&buf)
}

// A row whose canonical form CAN fail (mirrors serde_json refusing a NaN float).
// content_key is therefore fallible — a Result, never a silent default.
struct Row {
    name: String,
    alt_km: f64,
}

fn canonical(row: &Row) -> Result<String, String> {
    if row.alt_km.is_nan() {
        return Err(format!("cannot canonicalize {}: NaN altitude", row.name));
    }
    Ok(format!("{}|{}", row.name, row.alt_km))
}

fn content_key(row: &Row) -> Result<String, String> {
    Ok(content_id("row", &canonical(row)?))
}

fn main() {
    // Content-addressing: equal content -> equal id; any change -> a new id.
    println!("iss       = {}", content_id("row", "ISS|420"));
    println!("iss again = {}", content_id("row", "ISS|420")); // identical
    println!("hst       = {}", content_id("row", "HST|540")); // different content
    // The kind tag matters: same canonical text, different entity type -> new id.
    println!("as launch = {}", content_id("launch", "ISS|420"));

    // Idempotent load: forward only keys not seen this run.
    let rows = vec![
        Row { name: "ISS".into(), alt_km: 420.0 },
        Row { name: "HST".into(), alt_km: 540.0 },
        Row { name: "ISS".into(), alt_km: 420.0 }, // a duplicate of the first
    ];
    let mut seen = HashSet::new();
    let mut written = 0;
    for r in &rows {
        if seen.insert(content_key(r).unwrap()) {
            written += 1;
        }
    }
    println!("wrote {written} of {} rows", rows.len());
}
```

```text
iss       = 471231c61412846c
iss again = 471231c61412846c
hst       = 895a0f4eeaadb2eb
as launch = 14c1e913350a5b5d
wrote 2 of 3 rows
```

The first two lines are byte-identical: same content, same id — that is what makes dedup possible. `hst` differs because its content differs, and `as launch` differs from `iss` even though the canonical string `"ISS|420"` is the same, because the `kind` tag went into the hash. And three input rows, one of them a duplicate, produced **two** writes. Feed those same rows in again and the seen-set already holds every key: zero new writes. That is idempotency.

## The real seam: `Row::content_key` and `IdempotentSink`

On the real `Row`, the canonical form is its JSON, and the dedupe key is the content id of that JSON:

```rust,ignore
// crates/etl-core/src/content.rs
impl Row {
    pub fn content_key(&self) -> Result<String, EtlError> {
        // serde_json is deterministic for our structs (fields in declaration
        // order), so equal rows hash equal.
        let canonical = serde_json::to_string(self).map_err(|e| EtlError::Parse(e.to_string()))?;
        Ok(content_id("row", &canonical))
    }
}
```

`IdempotentSink` is a decorator: it wraps *any* `Sink`, keeps a set of keys it has already forwarded, and passes only the fresh rows to the inner sink:

```rust,ignore
// crates/etl-core/src/content.rs
#[async_trait]
impl<S: Sink> Sink for IdempotentSink<S> {
    async fn load(&self, rows: &[Row]) -> Result<usize, EtlError> {
        let mut fresh = Vec::new();
        {
            let mut seen = self.seen.lock().await;
            for row in rows {
                if seen.insert(row.content_key()?) {
                    fresh.push(row.clone());
                }
            }
        }
        if fresh.is_empty() {
            return Ok(0);
        }
        self.inner.load(&fresh).await
    }
}
```

Because it wraps `S: Sink` and *is* a `Sink`, you can slip it in front of the `JsonlSink` without either the sink or the orchestrator knowing — the same decorator pattern as any middleware. Two constructors: `IdempotentSink::new(inner)` starts with an empty seen-set (dedup *within* one run), and `IdempotentSink::with_seen(inner, keys)` pre-loads keys already on disk so re-runs *across processes* stay idempotent too. The shipped test `rerun_writes_zero_new_rows` loads two rows (returns `2`), then loads the identical two again and asserts the second call returns `0`.

## Why `content_key` is fallible — and the trap of "fixing" it

Look again at the signature: `content_key(&self) -> Result<String, EtlError>`. It would be *so* tempting to make it return a plain `String` — call sites get noisier when every key is a `?`. But serialization can genuinely fail (a `NaN` float has no JSON representation), and the question is what to do when it does. The wrong answer, `unwrap_or_default()`, is a quiet catastrophe:

```rust
use std::collections::HashSet;

fn digest(bytes: &[u8]) -> String {
    let mut h: u64 = 0xcbf29ce484222325;
    for &b in bytes {
        h ^= b as u64;
        h = h.wrapping_mul(0x100000001b3);
    }
    format!("{h:016x}")
}

fn content_id(kind: &str, canonical: &str) -> String {
    digest(format!("{kind}\0{canonical}").as_bytes())
}

struct Row {
    name: String,
    alt_km: f64,
}

fn canonical(row: &Row) -> Result<String, String> {
    if row.alt_km.is_nan() {
        return Err(format!("cannot canonicalize {}: NaN altitude", row.name));
    }
    Ok(format!("{}|{}", row.name, row.alt_km))
}

// THE BUG: swallow the error into a default. Every un-canonicalizable row now
// shares the SAME empty key — so the idempotent sink treats them as duplicates.
fn content_key_bad(row: &Row) -> String {
    canonical(row)
        .map(|c| content_id("row", &c))
        .unwrap_or_default()
}

fn main() {
    let a = Row { name: "Alpha".into(), alt_km: f64::NAN };
    let b = Row { name: "Beta".into(), alt_km: f64::NAN };

    // Two genuinely distinct rows...
    println!("bad key A = {:?}", content_key_bad(&a));
    println!("bad key B = {:?}", content_key_bad(&b));

    let mut seen = HashSet::new();
    let mut written = 0;
    for r in [&a, &b] {
        if seen.insert(content_key_bad(r)) {
            written += 1;
        }
    }
    // ...collapse to one empty key: the second is silently dropped as a "dup".
    println!("wrote {written} of 2 distinct rows (should be 2)");
}
```

```text
bad key A = ""
bad key B = ""
wrote 1 of 2 distinct rows (should be 2)
```

Two distinct rows, both un-serializable, both mapped to the empty string, and the idempotent sink concluded they were the *same* row and **dropped the second**. That is data loss disguised as deduplication — and it is silent, which makes it the worst kind. Returning a `Result` forces the error to the surface (`?` propagates it as `EtlError::Parse`), where a bad row is rejected loudly instead of being quietly merged into another. The shipped code's comment says it outright: fallible *on purpose*, never `unwrap_or_default`.

<div class="callout trap">
<span class="callout-label">A swallowed key error collapses distinct rows</span>
<code>unwrap_or_default()</code> on a key function turns "I could not compute this key" into "this key is the empty string" — and every row that hits the error now shares that one key. In a dedupe set, sharing a key means being discarded as a duplicate. So the failure mode is not a crash and not a bad key; it is <em>silent data loss</em>, with distinct rows vanishing because they were mistaken for each other. The fix is structural: make the key type <code>Result</code>, so the compiler will not let you ignore the failure.
</div>

## Why this is the linchpin for later arcs

Idempotent, content-addressed writes are what make everything downstream safe to *repeat*. The retry loop can re-run a request without fear of doubling rows. The orchestrator (Part V) can restart a crashed pipeline from the top and only the genuinely new rows land. The RAG arc (Part VI) keys embeddings by `content_id("embed", text)` so an expensive embedding is computed *once* and never recomputed for text it has already seen. None of that works without a stable, content-derived id — which is why this small hashing seam is installed here, before the arcs that spend it.

## Questions to lock

1. What does "idempotent" mean for a Load stage, and why does adding a retry loop *force* you to care about it?
2. In `content_id(kind, canonical)`, what two collisions do the `kind` tag and the `0x00` separator each prevent?
3. `IdempotentSink` wraps any `S: Sink` and is itself a `Sink`. What pattern is that, and what does it let you do without the inner sink's cooperation?
4. Why does `Row::content_key` return `Result<String, EtlError>` instead of `String`? Describe the exact failure that `unwrap_or_default()` would cause.
5. What is the difference between `IdempotentSink::new` and `IdempotentSink::with_seen`, and which one keeps re-runs idempotent *across separate processes*?

The graded quiz for this arc is at the end of the part, on the [Concept-Check: Pagination](./quiz.md) page.
