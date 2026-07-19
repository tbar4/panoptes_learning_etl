# Concept: The `Embedder` Trait & Content-Addressed Embeddings

> **Kind:** Concept. **Seam installed this arc:** the `Embedder` trait — the embedding boundary — plus the content-addressed cache that makes embedding idempotent.

## The idea

An embedder turns text into vectors. That is the entire job of the interface. But *how* you draw the boundary around that job decides whether the rest of your system is testable, swappable, and — the point of this chapter — cheap to re-run. The shipped `Embedder` trait is deliberately tiny:

```rust,ignore
// crates/etl-core/src/embed.rs
// (needs async-trait; not playground-safe — see the toy below for the same shape)
use async_trait::async_trait;

/// Turns text into vectors. One vector per input, same order.
#[async_trait]
pub trait Embedder: Send + Sync {
    async fn embed(&self, texts: &[String]) -> Result<Vec<Vec<f32>>, EtlError>;
}
```

If that shape looks familiar, it should: it is the *same move* as the first course's `ModelClient`. One `async` method, `Send + Sync` so it can be scheduled across threads, `Result<_, EtlError>` so failures flow through the one pipeline error type, and a batch in / batch out signature (`&[String]` → `Vec<Vec<f32>>`, one vector per input, same order). The trait names *what* an embedder does; it says nothing about *who* — Voyage, OpenAI, a local model, or a mock. That silence is the seam.

## Embeddings are the definitive expensive, idempotent, content-addressed workload

You met content-addressing in Part IV, for idempotent writes. Embeddings are the workload that makes it *matter*, because embedding has three properties that line up exactly:

- **Expensive.** Every `embed` call is a network round trip to a paid API. Embedding the same launch description twice is money spent twice for a byte-identical result.
- **Idempotent by nature.** A given model embedding a given string yields the same vector every time. The output is a pure function of the input — which is the precondition for caching to be *correct*, not just fast.
- **Content-addressed for free.** Because the output depends only on the text, the text (hashed) *is* the cache key. `content_id("embed", text)` — the same `sha256(kind ‖ 0x00 ‖ canonical)` you built for rows — gives a stable id for a piece of prose. Same text, same id, every run, across processes.

Put those together and the rule writes itself: **the same text must never be embedded twice.** Not within a run, not across runs, not across machines that share the index file. The content id is how you enforce it — you embed a text only if you have not already seen its id.

<div class="callout">
<span class="callout-label">Why this seam is worth its own trait</span>
Embedding is the one stage where "I accidentally ran it twice" has a dollar sign attached. A trait boundary lets you (a) put a real client behind it in production, (b) put a <em>free, instant, deterministic</em> mock behind it in every test, and (c) wrap it in a cache so the expensive path runs at most once per distinct text. All three depend on nothing but the text-in / vectors-out contract.
</div>

## A runnable toy: mock the embedder, cache by content

The real trait needs `async-trait`, so it will not run on the playground. Here is the *same shape* on a domain that cannot be mistaken for the answer key, reduced to synchronous std so it runs anywhere: a `MockEmbedder` that counts how many texts it was asked to embed, and a `seen` set keyed by content id that guarantees a repeated text is embedded exactly once.

```rust
use std::collections::HashSet;

/// Turns text into vectors. One vector per input, same order. Mirrors the real
/// `Embedder` trait — swappable and, crucially, mockable.
trait Embed {
    fn embed(&self, texts: &[String]) -> Vec<Vec<f32>>;
}

/// A test double: deterministic, counts how many texts it was asked to embed,
/// and never touches the network. This is what wiremock/mocks buy you.
struct MockEmbedder {
    calls: std::cell::Cell<usize>,
}
impl Embed for MockEmbedder {
    fn embed(&self, texts: &[String]) -> Vec<Vec<f32>> {
        self.calls.set(self.calls.get() + texts.len());
        texts.iter().map(|t| vec![t.len() as f32, 1.0]).collect()
    }
}

/// A content id for a piece of text. The real code hashes with sha256; here the
/// text itself is the key — the POINT is that identical text yields an identical
/// id, so a `seen` set can skip it.
fn content_id(text: &str) -> String {
    format!("embed\u{0}{text}")
}

fn main() {
    let embedder = MockEmbedder { calls: std::cell::Cell::new(0) };
    let mut seen: HashSet<String> = HashSet::new();

    // Two loads. The second repeats a text already embedded in the first.
    let batch1 = vec!["a launch of broadband sats".to_string(), "a resupply mission".to_string()];
    let batch2 = vec!["a resupply mission".to_string(), "a crewed test flight".to_string()];

    for batch in [batch1, batch2] {
        // Only embed texts whose content id we have not seen before.
        let fresh: Vec<String> = batch
            .into_iter()
            .filter(|t| seen.insert(content_id(t)))
            .collect();
        let vectors = embedder.embed(&fresh);
        println!("this load embedded {} fresh text(s)", vectors.len());
    }

    // 4 texts arrived across two loads, but one was a repeat.
    println!("total texts actually embedded: {}", embedder.calls.get());
}
```

```text
this load embedded 2 fresh text(s)
this load embedded 1 fresh text(s)
total texts actually embedded: 3
```

Four texts arrived across the two loads. One — `"a resupply mission"` — appeared in both. The `seen` set caught it the second time (`HashSet::insert` returns `false` when the key is already present), so the mock was asked to embed only **3** distinct texts. That is the content-addressed cache in miniature: the expensive operation runs once per *distinct content*, no matter how many times the content shows up or how many separate loads it is spread across.

<div class="callout trap">
<span class="callout-label">The mock is what makes the cache <em>testable</em></span>
You cannot assert "this text was embedded exactly once" against a real API — you would be counting billing events. The <code>calls</code> counter on the mock turns an economic property into a unit-test assertion: <code>assert_eq!(calls, 3)</code>. That is precisely what the shipped <code>cached_text_not_re_embedded</code> test does with its <code>CountingEmbedder</code>. Design the trait so the cache's correctness is <em>observable</em> without spending a cent.
</div>

## From "same run" to "across runs" — seeding the seen set

An in-memory `seen` set dedupes within a single process. But an ETL job runs on a schedule: today's run must not re-embed prose that yesterday's run already paid for. The fix is the same one the idempotent sink uses — *seed* the cache from what is already on disk. The shipped `EmbedSink` has two constructors for exactly this:

```rust,ignore
// crates/etl-core/src/vector.rs — needs etl-core (Embedder, content_id, ...)
impl<E: Embedder> EmbedSink<E> {
    /// Fresh: nothing embedded yet.
    pub fn new(embedder: E, path: impl Into<PathBuf>) -> Self { /* ... */ }

    /// Pre-load already-embedded ids so re-runs across processes never re-embed
    /// the same prose (embedding is expensive — this is the whole point).
    pub fn with_seen(
        embedder: E,
        path: impl Into<PathBuf>,
        ids: impl IntoIterator<Item = String>,
    ) -> Self { /* ... */ }
}
```

At startup the binary reads the existing `embeddings.jsonl`, collects every `EmbeddedRow.id`, and hands those ids to `with_seen`. The `seen` set therefore begins each run *already populated* with everything ever embedded — so the very first thing a re-run does is recognize that all its prose is old and make **zero** API calls. The content id is the bridge that makes an in-memory set behave like a durable, cross-run cache: the ids on disk and the ids computed this run are the same function of the same text, so they match.

## The one-for-one mapping

The toy's `Embed` trait is `etl_core::Embedder` with the async stripped off for the playground. `MockEmbedder` is the test-suite's `CountingEmbedder` — deterministic vectors, a call counter. The toy's `content_id` is the real `content_id("embed", text)` (sha256 instead of a format string, same contract: same text → same id). The toy's `seen` `HashSet` is `EmbedSink`'s `Mutex<HashSet<String>>`, and the toy's "filter to fresh, then embed" loop is the body of `EmbedSink::load`. The real production embedder behind the trait is `VoyageEmbedder` — bearer-authenticated, wiremock-tested, never hitting the live API in a test. Learn the boundary and the cache here; the build page is filling in a template you already understand.

## Questions to lock

1. What is the exact signature of `Embedder::embed`, and what does each part (`async`, `Send + Sync`, `&[String]` → `Vec<Vec<f32>>`, `Result<_, EtlError>`) buy you?
2. Embeddings are called the "definitive" content-addressed workload. Name the three properties that make content-addressed caching both *correct* and *worth it* here.
3. `HashSet::insert` returns a `bool`. How does that one return value implement "embed each distinct text at most once"?
4. Why can you assert "embedded exactly N times" against a mock but not against the real API, and which shipped test relies on this?
5. An in-memory `seen` set only dedupes within a run. What does `EmbedSink::with_seen` seed it from, and why does the content id make an in-memory set behave like a durable cross-run cache?

The graded quiz for this arc is at the end of the part, on the [Concept-Check: Retrieval](./quiz.md) page.
