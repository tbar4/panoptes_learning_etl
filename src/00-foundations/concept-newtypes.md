# Concept: Newtypes & Id-Safety

> **Kind:** Concept. No new crate — `serde` is assumed from the first course; this chapter is about a *pattern*, not a dependency.

## The idea

An ETL is, at bottom, a machine for moving ids around. A conjunction references a **primary** object and a **secondary** object, each by its NORAD catalog number. A launch has an id; a payload count is a number; a launch sequence is a number. Almost everything is "some integer that identifies something." And there lies a quiet, expensive class of bug: **all those integers have the same type, so the compiler cannot stop you from putting one where another belongs.**

Pass the secondary's id where the primary's was expected and the code compiles, the tests on the happy path pass, and you have silently mislabeled a collision-avoidance decision. Nothing crashes. The data is just wrong, and it stays wrong until someone downstream notices two satellites attributed backwards.

The **newtype** pattern closes this off. Wrap the raw integer in a one-field struct that means *this specific kind of id and no other*, and the compiler starts rejecting the mix-up at the type level — for free, with zero runtime cost, because the wrapper compiles away entirely.

## The bug, first — bare integers that compile and lie

Here is the failure mode with no newtypes. Two different ids, both `u32`, so the compiler sees them as interchangeable:

```rust
/// Both ids are just `u32`, so the compiler sees no difference between them.
fn ship_from_bin(bin: u32) {
    println!("dispatching a picker to bin {bin:04}");
}

fn main() {
    let bin_id: u32 = 42;
    let pallet_id: u32 = 7;
    // A copy-paste slip. This compiles cleanly and ships to the wrong place.
    ship_from_bin(pallet_id);
    let _ = bin_id;
}
```

```text
dispatching a picker to bin 0007
```

That program is a bug that runs perfectly. `ship_from_bin` wanted a *bin* and got a *pallet*, and nothing objected. In `panoptes_etl` the equivalent slip is swapping a conjunction's primary and secondary NORAD ids — same shape, higher stakes.

## The toy: a `BinId` newtype

Now give the id a type of its own. This mirrors `etl-core`'s real `NoradId` on a domain you cannot confuse with it — a warehouse bin location. The newtype wraps a `u32` and carries three behaviors: a padded `Display`, a validating `FromStr`, and a *transparent* serde form so the wire stays clean.

```rust,ignore
use std::fmt;
use std::str::FromStr;

use serde::{Deserialize, Serialize};

/// A warehouse bin location — a newtype over `u32`, not a bare integer.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(transparent)]
pub struct BinId(pub u32);

impl fmt::Display for BinId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // Bin labels are printed zero-padded to four digits.
        write!(f, "{:04}", self.0)
    }
}

impl FromStr for BinId {
    type Err = String;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        s.trim()
            .parse::<u32>()
            .map(BinId)
            .map_err(|_| format!("invalid bin id: {s:?}"))
    }
}

fn main() {
    // Display: padded label.
    println!("label:      {}", BinId(42));
    // FromStr: trims and parses.
    println!("parsed:     {}", " 1337 ".parse::<BinId>().unwrap());
    // Bad input is rejected, not silently coerced.
    println!("rejected:   {:?}", "aisle-7".parse::<BinId>());
    // serde(transparent): the wire form is the bare integer, no wrapper object.
    println!("as json:    {}", serde_json::to_string(&BinId(42)).unwrap());
    let back: BinId = serde_json::from_str("42").unwrap();
    println!("from json:  {back}");
}
```

> This block uses `serde` and `serde_json`, so it is marked `rust,ignore`. To run it, `cargo new` a binary crate and `cargo add serde --features derive` plus `cargo add serde_json`.

```text
label:      0042
parsed:     1337
rejected:   Err("invalid bin id: \"aisle-7\"")
as json:    42
from json:  0042
```

Read the four behaviors off the output:

- **`Display` is a presentation choice, not the storage.** `BinId(42)` stores the integer `42` and *displays* `0042`. The padding lives only in `fmt`; the value is untouched.
- **`FromStr` is a validation boundary.** Parsing `" 1337 "` trims and succeeds; parsing `"aisle-7"` returns `Err`. An unparseable string can never become a `BinId` — the type is only constructible from something that already validated. (Real ids will use `.parse::<BinId>()?` at the extract boundary, so bad input becomes an error *there* rather than a wrong value later.)
- **`#[serde(transparent)]` keeps the wire clean.** Without it, `BinId(42)` would serialize as a wrapper — `{"0": 42}` or a newtype-struct form — leaking the wrapper into JSON. With it, the id serializes as the *bare integer* `42` and round-trips straight back. The type safety is a compile-time-only fiction; the bytes on disk are just numbers.

## Now the bug cannot compile

Give the two ids distinct newtypes and the copy-paste slip from the top of the chapter becomes a compile error instead of a silent mislabeling:

```rust,compile_fail
struct BinId(u32);
struct PalletId(u32);

/// Ships whatever is in a given bin. It wants a *bin*, specifically.
fn ship_from_bin(_bin: BinId) {}

fn main() {
    let pallet = PalletId(7);
    ship_from_bin(pallet); // wrong id kind
}
```

That fails to compile with **`E0308: mismatched types`** — "expected `BinId`, found `PalletId`". The exact bug that ran perfectly with bare `u32`s is now caught before the program even builds. That is the entire payoff of the pattern: a class of mix-up bug converted from a runtime data-corruption into a compile error, at no runtime cost.

<div class="callout">
<span class="callout-label">Why this costs nothing at runtime</span>
A single-field struct like <code>BinId(u32)</code> has the identical memory layout to a bare <code>u32</code> — the wrapper is erased during compilation. You pay only in a little typing at the definition site and get back a compiler that refuses to confuse your ids. Zero-cost is not a slogan here; it is literally the same machine code.
</div>

## The mapping is one-for-one

`BinId` is `NoradId` with the domain swapped. Against `etl-core`'s real `ids.rs`:

- `BinId(pub u32)` ↔ **`NoradId(pub u32)`** — a one-field tuple struct over `u32`, with the same derive set (`Debug, Clone, Copy, PartialEq, Eq, Hash, …, Serialize, Deserialize`).
- `#[serde(transparent)]` ↔ **`#[serde(transparent)]`** — identical; a `NoradId` serializes as the bare integer `25544`, not a wrapper object.
- `Display` padding to **four** digits (`{:04}`) ↔ padding to **five** (`{:05}`) — NORAD ids are conventionally shown zero-padded to five, so `NoradId(733)` displays `00733` while storing `733`.
- `FromStr` trimming and parsing, `Err` on junk ↔ identical, except the real error type is **`EtlError::Parse`** (from the previous chapter) rather than a bare `String` — so a bad id flows into the same error taxonomy the retry loop understands, and it is correctly *permanent* (a malformed id will never parse no matter how many times you retry).

The domain is warehouse bins vs tracked satellites; the shape is the same shape. When you build `NoradId` in Part I's companion build work, you are re-deriving exactly the toy above with the padding width and error type changed.

## Questions to lock

1. Show a two-line example where two bare-`u32` ids let a wrong-argument bug compile. What does the newtype version do instead, and with what error code?
2. Where does a `BinId` do its *validation*, and what does that guarantee about every `BinId` value in the program?
3. What does `#[serde(transparent)]` change about the serialized form, and why do you want it for an id?
4. `NoradId(733)` stores `733` but displays `00733`. Where does the zero-padding live, and does it affect the stored value or the serialized bytes?
5. Why does a newtype cost nothing at runtime? What is its memory layout compared to the bare integer?
6. In the real `NoradId`, `FromStr` returns `EtlError::Parse` rather than a `String`. Using the previous chapter, is a parse failure transient or permanent, and why does that classification make sense for a malformed id?
