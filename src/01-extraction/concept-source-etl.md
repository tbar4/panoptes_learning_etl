# Concept: The `Source` Trait & the E/T/L Boundary

> **Kind:** Concept. **Seam installed this arc:** the `Source` trait + the Extract / Transform / Load split.

## The idea

An ETL job does three things, and the whole discipline of this course is keeping them three *separate* things:

1. **Extract (E)** — talk to the outside world and come back with raw bytes. This is where the network lives: HTTP, timeouts, 429s, auth. It is `async`, it is fallible, and it knows *nothing* about what the bytes mean.
2. **Transform (T)** — turn those raw bytes into your normalized domain types. This is pure: no network, no clock, no files. Same input, same output, every time. It is where parsing — and therefore *validation* — happens.
3. **Load (L)** — write the normalized rows somewhere durable. For us that is a JSONL file; later it could be a database. It knows nothing about where the rows came from.

Draw the boundaries as types and each stage becomes independently testable, independently swappable, and — the payoff four arcs from now — independently *schedulable*. The orchestrator will hold a `Source`, a `Transform`, and a `Sink` as trait objects and run them in order, never knowing it is talking to CelesTrak versus Space-Track. That only works if the three stages never leak into each other.

The two typed boundaries are the load-bearing part. Between E and T sits a `RawRecord` (bytes plus provenance). Between T and L sits a `Row` (a normalized domain value). Everything upstream of a boundary can change without touching anything downstream, because the boundary is a type, not a convention.

```text
   external API           RawRecord            Row              file
        │                     │                 │                │
        ▼        E            ▼        T         ▼       L        ▼
     Source ───────────►  (raw bytes) ──────► (domain) ──────► Sink
     async, fallible       pure parse          durable write
```

## The three traits, exactly as `etl-core` declares them

Here are the real signatures you will implement. Read them as the contract for the whole pipeline:

```rust,ignore
// crates/etl-core/src/pipeline.rs
// (needs async-trait + chrono + serde; not playground-safe — see the toy below
//  for a runnable version of the same shape)

use async_trait::async_trait;

/// Raw bytes fetched from a source, before any parsing — the output of E.
#[derive(Debug, Clone, PartialEq)]
pub struct RawRecord {
    pub source: String,
    pub payload: String,
    pub fetched_at: DateTime<Utc>,
}

/// A normalized domain row — the typed boundary between T and L.
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum Row {
    SpaceObject(SpaceObject),
    Conjunction(Conjunction),
    Launch(Launch),
}

/// **E** — extract raw records from an external source.
#[async_trait]
pub trait Source: Send + Sync {
    fn name(&self) -> &str;
    async fn extract(&self) -> Result<Vec<RawRecord>, EtlError>;
}

/// **T** — parse raw records into normalized rows. Pure and synchronous.
pub trait Transform: Send + Sync {
    fn transform(&self, raw: &RawRecord) -> Result<Vec<Row>, EtlError>;
}

/// **L** — persist normalized rows to a sink; returns the count written.
#[async_trait]
pub trait Sink: Send + Sync {
    async fn load(&self, rows: &[Row]) -> Result<usize, EtlError>;
}
```

Four details in those signatures carry the whole design; notice each one:

- **`Source::extract` is `async`; `Transform::transform` is not.** The network lives in E and only in E. T is a plain synchronous function because parsing bytes needs no runtime — which is exactly why it is trivial to unit-test with a hand-written `RawRecord` and no server at all.
- **Every method returns `Result<_, EtlError>`.** One error type flows through the whole pipeline. The classification you built in Part I (`is_transient()`) is what the later retry loop reads off these results.
- **All three traits are `Send + Sync`.** That is the price of admission to being scheduled across threads on a tokio runtime. The orchestrator will move these across `.await` points and hold them behind `Arc`; without `Send + Sync` that would not compile.
- **`Row` is one enum over every entity, tagged by `kind`.** This is what lets a *single* `Sink` accept the output of *any* pipeline. We will come back to why the tag matters.

## A runnable toy: the receipt fetcher

The trait objects above pull in `async-trait`, `chrono`, and `serde`, so they will not run on the playground. Here is the *same shape* on a domain that cannot be mistaken for the answer key — a corner shop's receipts — reduced to synchronous std so it runs anywhere. Fetch raw text → parse to typed rows → store; three traits, two typed boundaries:

```rust
// The raw bytes a source hands off — the output of E, before any parsing.
#[derive(Debug, Clone)]
struct RawReceipt {
    store: String,
    payload: String, // one "ITEM<TAB>CENTS" line per purchase
}

// The normalized row — the typed boundary between T and L.
#[derive(Debug, Clone, PartialEq)]
struct LineItem {
    item: String,
    cents: u32,
}

// E — pull raw text from somewhere external.
trait Fetch {
    fn fetch(&self) -> RawReceipt;
}

// T — parse raw text into normalized rows. Pure and total over its input.
trait Parse {
    fn parse(&self, raw: &RawReceipt) -> Vec<LineItem>;
}

// L — persist normalized rows; report how many landed.
trait Store {
    fn store(&mut self, rows: &[LineItem]) -> usize;
}

struct StubShop;
impl Fetch for StubShop {
    fn fetch(&self) -> RawReceipt {
        RawReceipt {
            store: "corner-shop".into(),
            payload: "coffee\t450\nbagel\t325\n".into(),
        }
    }
}

struct ReceiptParser;
impl Parse for ReceiptParser {
    fn parse(&self, raw: &RawReceipt) -> Vec<LineItem> {
        raw.payload
            .lines()
            .filter(|l| !l.is_empty())
            .map(|l| {
                let (item, cents) = l.split_once('\t').expect("well-formed line");
                LineItem { item: item.to_string(), cents: cents.parse().expect("numeric cents") }
            })
            .collect()
    }
}

struct MemStore {
    rows: Vec<LineItem>,
}
impl Store for MemStore {
    fn store(&mut self, rows: &[LineItem]) -> usize {
        self.rows.extend_from_slice(rows);
        rows.len()
    }
}

fn main() {
    // The pipeline names only the three seams, never the concrete shop.
    let source = StubShop;
    let transform = ReceiptParser;
    let mut sink = MemStore { rows: vec![] };

    let raw = source.fetch();          // E
    let rows = transform.parse(&raw);  // T
    let n = sink.store(&rows);         // L

    println!("fetched from {}", raw.store);
    println!("parsed {} rows", rows.len());
    println!("stored {n} rows");
    println!("first: {:?}", sink.rows[0]);
}
```

```text
fetched from corner-shop
parsed 2 rows
stored 2 rows
first: LineItem { item: "coffee", cents: 450 }
```

`main` names `Fetch`, `Parse`, and `Store` — never `StubShop` by knowing what a corner shop is. Swap in a `SupermarketApi` that implements `Fetch` and the parse/store code does not change one character. That is the seam, in miniature.

## Why `Row` is one tagged enum

Look again at the `Sink` signature: `async fn load(&self, rows: &[Row]) -> …`. One method, one row type — yet CelesTrak emits space objects, Space-Track emits conjunctions, and TheSpaceDevs emits launches. If each source had its own row type, you would need a different `Sink` per source, and the orchestrator could not treat them uniformly.

So `Row` is a single enum over *all* of them, and the `#[serde(tag = "kind", …)]` attribute makes each variant serialize with a discriminant field on the wire:

```rust
# use serde::{Serialize, Deserialize};
#[derive(Serialize, Deserialize, Debug, PartialEq)]
#[serde(tag = "kind", rename_all = "snake_case")]
enum Row {
    SpaceObject { norad_id: u32 },
    Launch { name: String },
}

fn main() {
    let r = Row::Launch { name: "Starlink G-99".into() };
    let json = serde_json::to_string(&r).unwrap();
    println!("{json}");
    // Round-trips: the tag is how the reader knows which variant to build.
    let back: Row = serde_json::from_str(&json).unwrap();
    println!("{}", back == r);
}
```

```text
{"kind":"launch","name":"Starlink G-99"}
true
```

The `"kind":"launch"` field is not decoration. It is how the downstream reader — `panoptes-gen` — knows which variant to reconstruct from a line of the file *without* out-of-band knowledge of which source wrote it. Untagged, `{"name": "Starlink G-99"}` would be ambiguous the moment two variants shared a field shape. The tag is the contract that lets one file hold rows from every pipeline.

<div class="callout">
<span class="callout-label">Parse, don't validate — in the T stage</span>
Because <code>Transform</code> returns <code>Row</code> and <code>Row</code>'s fields are your domain types (a <code>NoradId</code>, an <code>OrbitClass</code> enum, not bare strings), a successful transform is <em>proof</em> the data is well-formed. An off-codebook orbit class cannot survive parsing — it fails to deserialize rather than sitting in the file as a bad string. Validation is not a separate stage; it is the type of the T→L boundary.
</div>

## Object safety: why the orchestrator can schedule a `dyn Source`

The orchestrator (Part V) does not know about `CelestrakSource`. It holds a `Box<dyn Source>` — a source with its concrete type erased behind a vtable — and calls `.extract()` through that pointer. For that to compile, `Source` must be **dyn-compatible** (the current name for "object-safe"): every method must be callable through a vtable.

The real `Source::extract` is `async`, which normally is *not* dyn-compatible — but `#[async_trait]` rewrites it into a method returning a boxed future, a single concrete signature a vtable can hold. That is the whole reason the trait carries the attribute. Here is the boxed-source pattern working (same shape as `a_source_can_be_boxed_as_dyn` in the real test suite):

```rust,ignore
// deps: async-trait = "0.1", tokio = { version = "1", features = ["full"] }
use async_trait::async_trait;

#[derive(Debug)]
struct RawReceipt { store: String, payload: String }

#[async_trait]
trait Fetch: Send + Sync {
    fn name(&self) -> &str;
    async fn fetch(&self) -> RawReceipt;
}

struct StubShop;

#[async_trait]
impl Fetch for StubShop {
    fn name(&self) -> &str { "corner-shop" }
    async fn fetch(&self) -> RawReceipt {
        RawReceipt { store: self.name().to_string(), payload: "coffee\t450\n".into() }
    }
}

#[tokio::main]
async fn main() {
    // The orchestrator holds exactly this: a source with its type erased.
    let source: Box<dyn Fetch> = Box::new(StubShop);
    let raw = source.fetch().await;
    println!("{} -> {:?}", source.name(), raw.store);
}
```

```text
corner-shop -> "corner-shop"
```

### The trap: a generic method silently makes the trait un-schedulable

Object safety is easy to break by accident, and the error only shows up at the `Box<dyn …>` line — far from the method that caused it. Add one generic method and the whole trait can no longer be a trait object:

```rust,compile_fail
trait Source {
    // The generic `T` is the poison: each call could pick a different T,
    // so there is no single function pointer to put in a vtable.
    fn extract_as<T: From<String>>(&self) -> T;
}

fn main() {
    // The error lands HERE, not on the method above.
    let _boxed: Box<dyn Source> = todo!();
}
```

```text
error[E0038]: the trait `Source` is not dyn compatible
note: for a trait to be dyn compatible it needs to allow building a vtable
```

That `E0038` is the compiler telling you the orchestrator would never be able to hold this trait. The fix is to keep trait methods monomorphic — take `&self` and concrete arguments, return concrete types — which is exactly why `Source`, `Transform`, and `Sink` have the plain, generic-free signatures they do. **Schedulability is a property you design in from the first line, by keeping the trait dyn-compatible.**

## The one-for-one mapping

Everything in the receipt toy maps exactly onto what you are about to build, with the domain swapped and the network made real. `RawReceipt` becomes `RawRecord { source, payload, fetched_at }`; `LineItem` becomes the tagged `Row` enum; `Fetch::fetch` (sync) becomes `Source::extract` (async, returning `Result<Vec<RawRecord>, EtlError>`); `Parse::parse` becomes `Transform::transform` (`&RawRecord → Vec<Row>`, still pure); `Store::store` becomes `Sink::load` (async, appends JSONL, returns the count). The concrete `StubShop` becomes `CelestrakSource { base_url, group }`, `ReceiptParser` becomes `CelestrakTransform` (3-line TLE blocks → `Row::SpaceObject`), and `MemStore` becomes `JsonlSink { path }`. And where the toy's `main` named only the three traits, the orchestrator names only `Box<dyn Source>`, `dyn Transform`, and `dyn Sink`. The shape is identical; only the domain and the I/O change — learn the shape here and the CelesTrak build is filling in a template you already understand.

## Questions to lock

1. Which stage is `async` and which is pure, and why is that split what makes the transform trivially unit-testable?
2. What are the two typed boundaries in the pipeline, and what type sits on each?
3. Why is `Row` a single enum tagged by `kind` rather than one row type per source? What downstream problem does the tag solve?
4. What does "dyn-compatible" mean, and which one change to a trait's method would make the orchestrator unable to hold a `Box<dyn Source>`?
5. Where does validation happen in E/T/L, and why is a successful `Transform` a proof of well-formedness rather than a step you run afterward?

The graded quiz for this arc is at the end of the part, on the [Concept-Check: Extraction](./quiz.md) page.
