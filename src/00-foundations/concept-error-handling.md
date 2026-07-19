# Concept: Error Handling for Fallible I/O

> **Kind:** Concept. **New crate:** `thiserror` — this chapter shows it working before you build with it.

## The idea

Every stage of an ETL touches something it does not control — a network, an auth server, a disk, a payload someone else formatted. So every stage is **fallible**, and the interesting question is never "did it fail?" but "**is this failure worth retrying?**"

That single distinction — **transient vs permanent** — is the load-bearing idea of this whole chapter, and it is the thing your error type exists to encode:

- A **transient** error might succeed if you try again after a wait: a dropped connection, a timeout, a `429 Too Many Requests`. The world was briefly unavailable; it may be available in five seconds.
- A **permanent** error will fail *identically* every time: a malformed payload, bad credentials, a `404` for an endpoint that was renamed. Retrying it wastes time, hammers the server, and buries the real signal.

Get this wrong in the retrying direction and you build a machine that hits a `404` and retries it forever — a busy-loop against a resource that will never exist. Get it wrong in the other direction and you give up on a blip that a single retry would have fixed. The error type's job is to make the classification a property of the value, decided once, at the boundary where you actually know the cause.

## A toy that mirrors the real thing: a coffee order

Before we touch orbital data, here is the exact same shape on a domain you cannot mistake for the real code — a barista filling a coffee order. Some ways an order fails are worth trying again; some are not.

We model every failure as one enum, and we use `thiserror` to derive the `Display` messages and the `std::error::Error` impl. Then we add two methods that answer the only questions the retry loop will ask: *is this worth retrying,* and *did the failure come with a suggested wait?*

```rust,ignore
use std::time::Duration;

use thiserror::Error;

/// Every failure a coffee order can hit, in one type.
#[derive(Debug, Error)]
pub enum BaristaError {
    /// The espresso machine jammed mid-pull. Transient — try again.
    #[error("machine jammed: {0}")]
    MachineJam(String),

    /// Out of beans; the kitchen is restocking. Transient, with a hint.
    #[error("restocking; try again in {retry_after_secs}s")]
    Restocking { retry_after_secs: u64 },

    /// The card was declined. Permanent — retrying charges nothing new.
    #[error("card declined: {0}")]
    CardDeclined(String),

    /// That drink is not on the menu. Permanent.
    #[error("not on the menu: {0}")]
    NotOnMenu(String),

    /// The handwritten order slip was illegible. Permanent.
    #[error("unreadable slip: {0}")]
    UnreadableSlip(String),
}

impl BaristaError {
    /// Whether retrying (after a wait) could plausibly succeed.
    pub fn is_transient(&self) -> bool {
        matches!(
            self,
            BaristaError::MachineJam(_) | BaristaError::Restocking { .. }
        )
    }

    /// The wait the kitchen asked for, when it gave one.
    pub fn retry_after(&self) -> Option<Duration> {
        match self {
            BaristaError::Restocking { retry_after_secs } => {
                Some(Duration::from_secs(*retry_after_secs))
            }
            _ => None,
        }
    }
}
```

> `thiserror` is **not** available on the Rust playground, so these blocks are `rust,ignore` and have no play button. To run this yourself, `cargo new` a binary crate and `cargo add thiserror`. The `#[error("…")]` attribute is what generates the `Display` impl; `{0}` interpolates a tuple field and `{retry_after_secs}` a named struct field.

Drive it over one of each and print the classification:

```rust,ignore
fn main() {
    let failures = [
        BaristaError::MachineJam("group head stuck".into()),
        BaristaError::Restocking { retry_after_secs: 20 },
        BaristaError::CardDeclined("insufficient funds".into()),
        BaristaError::NotOnMenu("unicorn frappe".into()),
        BaristaError::UnreadableSlip("coffee smudge".into()),
    ];
    for f in &failures {
        println!(
            "{:>28}  transient={:<5}  retry_after={:?}",
            f.to_string(),
            f.is_transient(),
            f.retry_after()
        );
    }
}
```

```text
   machine jammed: group head stuck  transient=true   retry_after=None
       restocking; try again in 20s  transient=true   retry_after=Some(20s)
  card declined: insufficient funds  transient=false  retry_after=None
     not on the menu: unicorn frappe  transient=false  retry_after=None
     unreadable slip: coffee smudge  transient=false  retry_after=None
```

Two things fall out of the design. First, `is_transient` is a `matches!` over exactly the two retryable variants — the classification is *centralized*, not scattered across every call site. Second, only `Restocking` carries a `retry_after`, because only it comes with a suggested wait; every other variant returns `None`, and the retry loop falls back to its own backoff. A transient error may or may not carry a hint; a permanent error never does.

## The mapping is one-for-one

The `BaristaError` above is not an analogy — it is the same type as the one you will build, with the words swapped. Here is the correspondence, variant for variant, against `etl-core`'s real `EtlError`:

- `MachineJam(String)` ↔ **`Network(String)`** — a connection reset or timeout. Transient.
- `Restocking { retry_after_secs }` ↔ **`RateLimited { retry_after_secs }`** — the server said slow down (`429`). Transient, *with a hint*.
- `CardDeclined(String)` ↔ **`Auth(String)`** — bad or missing credentials (`401`/`403`). Permanent.
- `NotOnMenu(String)` ↔ **`Permanent(String)`** — a `404`, a `400`, a renamed endpoint. Permanent.
- `UnreadableSlip(String)` ↔ **`Parse(String)`** — a payload that would not parse into the domain model. Permanent.

`EtlError` also carries one variant the toy omits: `Io(#[from] std::io::Error)`, for a local read/write failure while touching a sink. The `#[from]` gives you a free `From<std::io::Error>` conversion so `?` lifts an I/O error straight into `EtlError`. And the two methods are identical: `EtlError::is_transient()` is `matches!(self, Network(_) | RateLimited { .. })`, and `EtlError::retry_after()` returns `Some(Duration)` only for `RateLimited`. The domain is coffee vs conjunctions; the shape is the same shape. This classification is precisely what the Part IV retry loop consumes: `with_retry` retries only while `is_transient()` is true, and honors `retry_after()` when it is `Some`.

<div class="callout predict">
<span class="callout-label">Predict before you read on</span>
When the Part IV retry loop calls <code>with_retry</code> on an operation that returns <code>EtlError::Parse</code>, how many times does it call the operation? What about <code>EtlError::RateLimited { retry_after_secs: 30 }</code>? Say it before the next section — the answer is the whole point of the classification.
</div>

## The trap: one transient bucket for every non-2xx

Here is the mistake the extraction arc walks straight into, and it is worth seeing now because it is *seductive*. When you first wire up an HTTP source, the tempting shortcut is: "the request succeeded if the status was `2xx`; otherwise it failed, so treat every non-`2xx` as a transient network error and let the retry loop sort it out."

That code compiles, passes the happy-path test, and is wrong in a way that only shows up in production. It classifies a `404` as transient — so the retry loop hammers a nonexistent endpoint, backing off longer and longer, forever, before finally giving up much later than it should. It classifies a `401` as transient — so a bad API key triggers a retry storm instead of failing fast with "fix your credentials." The one bucket that *should* be transient, `429`, gets lumped in with everything else and loses its `Retry-After` hint.

The correct classification reads the status **by class, with `429` pulled out first**:

```rust
/// Classify an HTTP status the way the retry loop must: not "2xx vs the rest".
fn classify(status: u16) -> &'static str {
    match status {
        200..=299 => "success",
        429 => "transient: honor Retry-After, then retry",
        400..=499 => "permanent: do NOT retry",
        500..=599 => "transient: back off and retry",
        _ => "unknown",
    }
}

fn main() {
    for status in [200, 404, 429, 400, 503, 401] {
        println!("{status} -> {}", classify(status));
    }
}
```

```text
200 -> success
404 -> permanent: do NOT retry
429 -> transient: honor Retry-After, then retry
400 -> permanent: do NOT retry
503 -> transient: back off and retry
401 -> permanent: do NOT retry
```

The rule, in three lines:

- **`429` → transient, with backoff.** The server explicitly told you to slow down and usually said for how long (`Retry-After`). This becomes `EtlError::RateLimited`.
- **Other `4xx` → permanent.** `400`, `401`, `403`, `404` are all *your* request being wrong: bad input, bad credentials, wrong URL. Retrying an unchanged request gets the identical rejection. These become `EtlError::Auth` (`401`/`403`) or `EtlError::Permanent` (the rest).
- **`5xx` → transient.** The server broke on its end; the same request may well succeed once it recovers. This becomes `EtlError::Network`.

<div class="callout trap">
<span class="callout-label">The match-order trap hides inside the fix</span>
Notice <em>where</em> the <code>429</code> arm sits: <strong>above</strong> the <code>400..=499</code> arm. Rust matches arms top to bottom, and <code>429</code> is inside <code>400..=499</code>. Put the range first and it swallows <code>429</code>, quietly turning your one retryable client error into a permanent one — the exact opposite of the bug you were trying to fix, and one no happy-path test will catch. This ordering is itself a quiz question. Read your <code>match</code> arms from most specific to least, always.
</div>

## Why an enum, and not just strings or `anyhow`

You might ask why the source layer needs a typed `EtlError` at all when `anyhow` exists. The answer is exactly the retry decision. `anyhow::Error` is opaque — you cannot ask it "were you a `429`?" without stringly-typed sniffing. A `thiserror` enum makes the classification a *method on the type*, checkable at compile time and impossible to forget a case of. The binary at the top of the stack can still use `anyhow` for its `main` (and does); but the library stages that a retry loop reasons about need a type that can *answer questions about itself*. That is the division the first course drew, and it holds here.

## Questions to lock

1. State the transient-vs-permanent distinction in one sentence, and give one example error on each side.
2. In `EtlError`, which two variants are transient? Which single variant carries a `retry_after`, and why only that one?
3. What does `#[from]` on `Io(#[from] std::io::Error)` generate, and how does it change what `?` can do?
4. Why is classifying every non-`2xx` status as one transient bucket a bug? What specifically goes wrong for a `404`? For a `401`?
5. In the correct `classify`, why must the `429` arm come *before* the `400..=499` arm?
6. Why does the source layer use a typed `thiserror` enum instead of `anyhow` for its errors?
