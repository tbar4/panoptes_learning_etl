# Concept: Structured vs Semantic Retrieval

> **Kind:** Concept. **Seam installed this arc:** the retrieval split — SQL/filter for the numbers, embeddings for the prose — and a brute-force cosine index.

## The idea

Four arcs in, your pipeline has produced a pile of normalized `Row`s: space objects with orbital elements, conjunctions with miss distances and probabilities, launches with dates and providers. Now something downstream wants to *retrieve* from that pile. There are two fundamentally different kinds of question it can ask, and conflating them is the single most common mistake in a first RAG system.

1. **Structured questions** are about the *numeric and categorical* fields. "Conjunctions with a collision probability above 0.001." "Launches by SpaceX after July." "The 20 objects with the lowest perigee." These are exact predicates over values that already have a precise meaning. You answer them by **filtering and sorting** — SQL `WHERE`/`ORDER BY`, or the Rust equivalent, `iter().filter(...).sort_by(...)`.

2. **Semantic questions** are about *meaning in prose*. "Launches that sound like rideshare missions." "Anything describing a broadband constellation." The words in a `description` field carry meaning that no `WHERE` clause can match, because "rideshare" and "batch of smallsats for multiple customers" are close in meaning but share no keywords. You answer these by **embedding** the prose into vectors and ranking by similarity.

The whole discipline of this arc is a single rule: **use the right mechanism for each kind of field, and never cross the streams.** You do not embed the numbers, and you do not try to `WHERE`-clause the prose.

## Why you must not semantic-search the numbers

It is tempting — especially once embeddings feel like magic — to just embed *everything*: turn each whole row into text, embed that, and let cosine similarity handle every query. Resist it. Embedding the numbers is wrong on every axis that matters:

- **It is lossy where you can least afford it.** A collision probability of `0.001` versus `0.01` is a *tenfold* difference that decides whether an operator moves a satellite. Rounded into a many-dimensional vector alongside a text blob, that distinction dissolves. Numbers have exact meaning; embeddings approximate. You would be throwing away precision you already had for free.
- **It answers the wrong question.** Cosine similarity ranks by *closeness*, but "probability above 0.001" is a *threshold*, not a ranking. There is no query vector whose nearest neighbors are exactly the rows over a cutoff. The operation you need — a filter — is not the operation an index performs.
- **It is expensive and non-deterministic** for a job a comparison operator does instantly and exactly. Embedding costs an API call per text; a `>` costs nothing and never flakes.

The numeric entities — `SpaceObject`, `Conjunction`, and the numeric fields of `Launch` — are for **structured retrieval**. The prose — and in this project that means exactly one field, a launch's `description` — is for **semantic retrieval**. That is why, when you look at the shipped code, `Row::prose()` returns `Some` for a non-empty launch description and `None` for everything else:

```rust,ignore
// crates/etl-core/src/vector.rs — needs the Row enum from etl-core
impl Row {
    /// The prose worth embedding, if any. Only text fields — never the numbers.
    pub fn prose(&self) -> Option<String> {
        match self {
            Row::Launch(l) if !l.description.trim().is_empty() => Some(l.description.clone()),
            _ => None,
        }
    }
}
```

A `SpaceObject` returns `None`. A `Conjunction` returns `None`. A `Launch` with an empty description returns `None`. That `Option` *is* the retrieval split, expressed as a method: it is the codebase's single point of truth for "what, in this row, is meaning-bearing prose rather than a number." Everything the embed stage does keys off it.

## A prose field is not a vector

Before you can rank prose by meaning, you have to *turn it into* a vector. This is not a formality — it is a type-level fact, and the compiler will remind you. `cosine` takes two `&[f32]`; a string is not a slice of floats:

```rust,compile_fail
fn cosine(a: &[f32], b: &[f32]) -> f32 {
    a.iter().zip(b).map(|(x, y)| x * y).sum()
}

fn main() {
    // Raw prose is text, not a vector. You cannot rank meaning without first
    // turning each side into an embedding.
    let query = ["broadband satellites"];
    let doc = ["a launch of broadband sats"];
    let _ = cosine(&query, &doc);
}
```

```text
error[E0308]: arguments to this function are incorrect
   |             ^^^^^^ ------  ---- expected `&[f32]`, found `&[&str; 1]`
```

That `E0308` is the type system stating the whole reason the embed stage exists: there is no path from text to a similarity score that skips the embedder. Semantic retrieval always has two halves — embed the corpus once (the load target you build next chapter), and embed the *query* the same way at search time, then compare vectors. The two "sides" the toy below feeds to `cosine` are both already vectors precisely because both were embedded first.

## A runnable toy: the two mechanisms side by side

Here is the split on a domain that cannot be mistaken for the answer key — a bookstore. Each book has a **numeric** `price_cents` (structured: filter it) and a **prose** blurb that has already been embedded into `blurb_vec` (semantic: cosine-rank it). Watch the two queries use two completely different operations over the same data:

```rust
// Cosine similarity; 0.0 if either side is the zero vector. Same shape as
// etl-core's `cosine` — the whole semantic index is built on this one line.
fn cosine(a: &[f32], b: &[f32]) -> f32 {
    let dot: f32 = a.iter().zip(b).map(|(x, y)| x * y).sum();
    let na = a.iter().map(|x| x * x).sum::<f32>().sqrt();
    let nb = b.iter().map(|x| x * x).sum::<f32>().sqrt();
    if na == 0.0 || nb == 0.0 { 0.0 } else { dot / (na * nb) }
}

struct Book {
    title: &'static str,
    price_cents: u32,    // a number: filter/sort it, never embed it
    blurb_vec: Vec<f32>, // the prose blurb, already embedded
}

fn main() {
    let catalog = vec![
        Book { title: "Orbital Mechanics",   price_cents: 4200, blurb_vec: vec![0.9, 0.1, 0.0] },
        Book { title: "The Quiet Garden",    price_cents: 1500, blurb_vec: vec![0.0, 0.2, 0.9] },
        Book { title: "Rockets for Kids",    price_cents: 1200, blurb_vec: vec![0.7, 0.3, 0.2] },
    ];

    // STRUCTURED: "books under $20" is a numeric predicate. Exact, cheap,
    // no embeddings — asking a vector index this would be absurd.
    println!("under $20:");
    for b in catalog.iter().filter(|b| b.price_cents < 2000) {
        println!("  {} (${}.{:02})", b.title, b.price_cents / 100, b.price_cents % 100);
    }

    // SEMANTIC: "something like a spaceflight book" is a meaning query. We
    // embed the QUERY the same way the blurbs were embedded, then rank by
    // cosine. The numbers (price) play no part here.
    let query = vec![0.9, 0.1, 0.05]; // pretend-embedding of "spaceflight"
    let mut ranked: Vec<(&str, f32)> = catalog
        .iter()
        .map(|b| (b.title, cosine(&query, &b.blurb_vec)))
        .collect();
    ranked.sort_by(|a, b| b.1.total_cmp(&a.1));
    println!("closest in meaning:");
    for (title, score) in ranked.iter().take(2) {
        println!("  {title} ({score:.3})");
    }
}
```

```text
under $20:
  The Quiet Garden ($15.00)
  Rockets for Kids ($12.00)
closest in meaning:
  Orbital Mechanics (0.998)
  Rockets for Kids (0.938)
```

Look at what each query touched. The "under $20" query read `price_cents` and never looked at a vector; a cheap paperback about a garden and a cheap kids' book both qualify because price is all that matters. The "closest in meaning" query read `blurb_vec` and never looked at price; the two spaceflight-ish books rise to the top, the $15 garden book sinks, and the $42 price of the top hit is *irrelevant to the ranking*. Same catalog, two orthogonal mechanisms. That is the split you are building, with the domain swapped and the vectors made real.

<div class="callout">
<span class="callout-label">The query gets embedded too</span>
Notice <code>query</code> in the toy is a <code>Vec&lt;f32&gt;</code>, not a string — it is the embedding of "spaceflight", produced by the same embedder that made the blurb vectors. This is the half of semantic search that is easy to forget: an index of embedded documents is useless until you embed the incoming query <em>with the same model</em> and compare vector-to-vector. Cross models and the geometry does not line up; the scores are noise.
</div>

## Brute-force cosine is the correct tool here — no vector database

Read the total workload honestly: this project's corpus is *thousands* of launch descriptions, not millions of web pages. `top_k` over the whole index is a linear scan — compute cosine against every row, sort, take `k`. At a few thousand rows that is sub-millisecond, runs in a plain `Vec<EmbeddedRow>`, needs no extra process, no index build, no ANN approximation, and returns the *exact* nearest neighbors every time.

A vector database (FAISS, Qdrant, pgvector, a hosted service) buys you approximate nearest-neighbor search that stays fast at millions-to-billions of vectors — and charges you an operational dependency, an index that must be rebuilt as data changes, and *approximate* results for that speed. At this corpus size you would be paying all of that cost to make a fast, exact operation slightly faster and less correct. **Brute force is not the cheap placeholder here; it is the right engineering answer for the scale.** The shipped `VectorIndex` is a `Vec` and a sort, on purpose.

<div class="callout trap">
<span class="callout-label">"We'll need a vector DB eventually" is a scale decision, not a default</span>
The instinct to reach for a hosted vector store on day one is the same instinct that reaches for Kafka to move ten messages a day. The honest question is "how many vectors, growing how fast?" Below roughly a hundred thousand, a linear scan in memory is simpler, exact, and plenty fast; you add an ANN index when profiling — not vibes — says the scan is the bottleneck. Designing the retriever behind a small surface (<code>cosine</code>, <code>top_k</code>) means that swap, if it ever comes, changes one struct and no callers.
</div>

## The one-for-one mapping

Everything in the bookstore toy maps onto what you build this arc. `Book.price_cents` is the numeric side — `Conjunction.collision_probability`, `Launch.net`, an orbital perigee — retrieved by `filter`/`sort` (structured). `Book.blurb_vec` is `Launch.description` after it passes through `Row::prose()` and the embedder — retrieved by `cosine` + `top_k` (semantic). The toy's `cosine` is `etl_core::cosine` verbatim. The toy's `ranked.sort_by(...).take(2)` is `VectorIndex::top_k`. And the toy's hard separation — price never enters the meaning query, vectors never enter the price query — is the discipline that keeps `Row::prose()` returning `None` for everything that is not a launch description. Learn the split here; the build pages are filling in a template you already understand.

## Questions to lock

1. Give one structured question and one semantic question over the space corpus, and name the mechanism each one uses.
2. State three concrete reasons embedding a `collision_probability` (or any number) is the wrong tool.
3. Which single field in this project is meaning-bearing prose, and what does `Row::prose()` return for a `SpaceObject` or a `Conjunction`?
4. Semantic search has two halves. What has to happen to the incoming *query* before you can compute a similarity against the index, and why must it use the same model?
5. Why is a brute-force cosine scan the *correct* choice at this corpus size rather than a stopgap — and what specific evidence would justify adding a vector database later?

The graded quiz for this arc is at the end of the part, on the [Concept-Check: Retrieval](./quiz.md) page.
