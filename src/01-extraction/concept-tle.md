# Concept: Parsing the TLE Format

> **Kind:** Concept. **This is the T stage's hardest job:** turning a 1960s fixed-width text format into typed orbital elements, correctly.

## The idea

CelesTrak hands you satellite orbits as **two-line element sets** (TLEs) — a fixed-width text format designed for punch cards and never changed since. It looks like this (the canonical ISS example that every TLE tutorial uses, and that the real `tle.rs` test suite uses):

```text
ISS (ZARYA)
1 25544U 98067A   08264.51782528 -.00002182  00000-0 -11606-4 0  2927
2 25544  51.6416 247.4627 0006703 130.5360 325.0288 15.72125391563537
```

A block is three lines: a name, then line 1 and line 2 of the actual elements. Nothing in it is delimited — there are no commas, no tabs, no key=value. **A field is defined purely by which columns it occupies.** The inclination is "whatever characters sit in columns 9 through 16 of line 2." Parse it by splitting on whitespace and you will be wrong the moment a field is blank or a sign is missing; you must slice by column.

Three things about this format bite everyone, and this chapter is about all three: **column indexing** (the format is specified 1-based; Rust slices are 0-based), the **mod-10 checksum** (how each line proves it was not corrupted in transit), and a pair of **encoding quirks** (a 2-digit year that pivots across a century, and an eccentricity with its decimal point left out).

## A toy fixed-width record

Before the real thing, the same *shape* on a domain that cannot be the answer key — a ground-station log line. It has the two features that make TLEs tricky: fields pinned to columns, and an implied-decimal number. The spec is written the way real fixed-width specs are, **1-based and inclusive**:

```text
cols 1-3   station code    e.g. "KWJ"
cols 5-8   range, km       e.g. " 412"
cols 10-15 elevation       implied leading "0." → "123456" means 0.123456
```

```rust
// A toy fixed-width record that mirrors TLE's shape.
const REC: &str = "KWJ 412  123456";
//                  123456789012345   <- 1-based column ruler

/// Translate a 1-based inclusive column span [a, b] to a Rust byte slice.
/// The rule that removes the off-by-one for good: `line[a-1 .. b]`.
fn cols(line: &str, a: usize, b: usize) -> &str {
    &line[a - 1..b]
}

fn main() {
    let station = cols(REC, 1, 3);
    let range = cols(REC, 5, 8).trim();
    let elev_digits = cols(REC, 10, 15);
    let elevation: f64 = format!("0.{elev_digits}").parse().unwrap();

    println!("station:   {station:?}");
    println!("range:     {range:?}");
    println!("elevation: {elevation}");
}
```

```text
station:   "KWJ"
range:     "412"
elevation: 0.123456
```

Two moves in that toy are exactly the moves the real parser makes: `cols(line, a, b)` maps a 1-based inclusive spec span onto `line[a-1..b]`, and the implied decimal is recovered with `format!("0.{digits}")`. Hold onto both.

## The trap: 0-based slice vs 1-based columns

Here is the single most common TLE-parsing bug, and it is silent. The spec says a field is in "columns 1–3." You reach for a Rust range and write `line[1..3]` because the numbers match. But Rust ranges are **0-based and end-exclusive**, while the spec is **1-based and end-inclusive** — so `line[1..3]` grabs the *wrong two characters* and, crucially, does not panic:

```rust
const REC: &str = "KWJ 412  123456";

fn main() {
    // Spec: station code is columns 1-3 (1-based, inclusive) => "KWJ".
    let correct = &REC[0..3]; // a-1 .. b  =>  1-1 .. 3
    println!("correct REC[0..3]: {correct:?}");

    // THE TRAP: reading "columns 1-3" as the Rust range 1..3.
    let wrong = &REC[1..3];
    println!("naive   REC[1..3]: {wrong:?}  <- off by one, no panic");
}
```

```text
correct REC[0..3]: "KWJ"
naive   REC[1..3]: "WJ"  <- off by one, no panic
```

It compiles, it runs, it returns a plausible-looking string — and every downstream number is shifted by one column. That is why the real parser routes every field through a single helper. Look at `tle.rs`:

```rust,ignore
fn slice(line: &str, start: usize, end: usize) -> Result<&str, EtlError> {
    line.get(start..end)
        .ok_or_else(|| EtlError::Parse(format!("line too short for cols {start}..{end}: {line:?}")))
}
```

Note two defenses at once. First, the real code writes the spans **already converted to 0-based half-open form** — the ISS inclination, spec columns 9–16, is fetched as `slice(line2, 8, 16)`. Converting once, at the call site, and never re-deriving it is how you kill the off-by-one. Second, it uses `line.get(start..end)` (which returns `Option`) rather than `line[start..end]` (which panics): a truncated line becomes an `EtlError::Parse`, not a crash.

## The checksum: catching corruption

Each TLE line ends in a single **mod-10 checksum digit** in column 69. The rule is deliberately crude, because it was designed to survive teletype transmission: walk columns 1–68, add each **digit**'s value, count each **minus sign** as 1, count **everything else** (letters, spaces, periods, plus signs) as 0, and take the total mod 10. Here is the real `tle_checksum`, verbatim:

```rust
/// mod-10 checksum: digits sum, each '-' counts 1, all else 0.
fn tle_checksum(line: &str) -> u8 {
    let sum: u32 = line
        .chars()
        .take(68) // columns 1..=68; the 69th char is the checksum itself
        .map(|c| match c {
            '0'..='9' => c.to_digit(10).unwrap(),
            '-' => 1,
            _ => 0,
        })
        .sum();
    (sum % 10) as u8
}

fn main() {
    // The canonical ISS TLE. Column 69 (the last char) is the checksum the
    // line claims: 7 on each line.
    let l1 = "1 25544U 98067A   08264.51782528 -.00002182  00000-0 -11606-4 0  2927";
    let l2 = "2 25544  51.6416 247.4627 0006703 130.5360 325.0288 15.72125391563537";

    println!("line 1 computed checksum: {}", tle_checksum(l1));
    println!("line 1 claims:            {}", l1.chars().nth(68).unwrap());
    println!("line 2 computed checksum: {}", tle_checksum(l2));
    println!("line 2 claims:            {}", l2.chars().nth(68).unwrap());

    // Flip one digit and the checksum no longer matches — corruption caught.
    let corrupt = l1.replacen("25544", "25545", 1);
    println!("after 1-digit edit:       {}", tle_checksum(&corrupt));
}
```

```text
line 1 computed checksum: 7
line 1 claims:            7
line 2 computed checksum: 7
line 2 claims:            7
after 1-digit edit:       8
```

Two things to internalize. The `.take(68)` is not arbitrary: it stops *before* the checksum digit, because you cannot include the checksum in the sum you are checking it against. And `'-' => 1` is the rule everyone forgets — minus signs contribute, plus signs and periods do not. The last line shows the payoff: change one character in the payload and the computed checksum (8) no longer equals what the line claims (7), so the corruption is caught before you ever try to parse a bad number.

<div class="callout trap">
<span class="callout-label">The checksum trap: forgetting the minus sign</span>
The single most common wrong checksum implementation counts only digits and ignores <code>'-'</code>. It will pass on lines that happen to have no minus signs and fail mysteriously on lines that do — and TLE lines routinely carry minus signs in the drag term and mean-motion derivatives. If your checksum matches on line 2 (usually no leading sign) but not line 1, this is almost always why. Minus counts 1; plus and period count 0.
</div>

The parser calls this through `verify_checksum`, which reads the claimed digit from column 69 and compares:

```rust,ignore
fn verify_checksum(line: &str) -> Result<(), EtlError> {
    let expected = line
        .chars()
        .nth(68) // column 69, 0-based index 68
        .and_then(|c| c.to_digit(10))
        .ok_or_else(|| EtlError::Parse(format!("missing checksum digit in {line:?}")))?
        as u8;
    let got = tle_checksum(line);
    if got != expected {
        return Err(EtlError::Parse(format!(
            "checksum mismatch in {line:?}: computed {got}, line says {expected}"
        )));
    }
    Ok(())
}
```

A bad checksum is an `EtlError::Parse` — which, from Part I, is **permanent, not transient**. Retrying a corrupt line will never fix it, so the retry loop (Part IV) must not waste attempts on it. The error classification you built earlier is exactly what makes "reject and move on" the correct behavior here.

## The two encoding quirks

### Epoch year pivot

The epoch is stored as a 2-digit year in columns 19–20 plus a fractional day-of-year in columns 21–32. Two digits cannot span a century, so TLEs use a fixed pivot: **57–99 mean 1957–1999, and 00–56 mean 2000–2056** (1957 is chosen because that is Sputnik — nothing was in orbit before it). The real `decode_epoch` encodes exactly that:

```rust
fn main() {
    for yy in [57, 99, 0, 8, 26, 56] {
        // The TLE century pivot, verbatim from decode_epoch:
        let year = if yy < 57 { 2000 + yy } else { 1900 + yy };
        println!("{yy:02} -> {year}");
    }
}
```

```text
57 -> 1957
99 -> 1999
00 -> 2000
08 -> 2008
26 -> 2026
56 -> 2056
```

So the ISS line's `08` decodes to **2008**, not 1908 — and a satellite launched in 2026 carries `26`, decoding to 2026. Get the pivot backwards and every recent satellite jumps to the early 1900s, decades before spaceflight existed.

### Implied-decimal eccentricity

Eccentricity is always between 0 and 1, so the format saves a character by omitting the leading `0.`. Columns 27–33 of line 2 hold `0006703`, which means **0.0006703**. The parser puts the decimal back exactly the way the toy did:

```rust
fn main() {
    // Columns 27-33 of the ISS line 2, as raw characters:
    let raw = "0006703";
    let eccentricity: f64 = format!("0.{raw}").parse().unwrap();
    println!("eccentricity = {eccentricity}");
}
```

```text
eccentricity = 0.0006703
```

Read those seven characters as a bare number and you get `6703.0` — an "eccentricity" seven million times too large, which would classify the ISS as a wildly elliptical comet. The `format!("0.{}")` is small but load-bearing.

## Putting it together

`parse_tle(line1, line2)` runs the whole gauntlet in order: verify both checksums, confirm both lines carry the same NORAD id (a mismatch means the 3-line block was misaligned by an upstream truncation — something checksums alone cannot catch), then slice each field by its 0-based-converted column span and parse it. The worked ISS result, from the real `parses_iss_tle_to_elements` test:

```text
inclination_deg        ≈ 51.6416   (cols 9-16  of line 2)
eccentricity           ≈ 0.0006703 (cols 27-33, decimal implied)
mean_motion_rev_per_day≈ 15.72125  (cols 53-63 of line 2)
orbit_class            = LEO        (derived: period ≈ 91.6 min < 128)
```

Every one of those numbers depends on getting the column span right, the checksum passing first, and the eccentricity's decimal restored. Miss any of the three traps and you get a plausible-looking `OrbitalElements` that is quietly, dangerously wrong — the worst kind of parse bug, because nothing crashes.

## Questions to lock

1. The spec says a field is in "columns 9–16." What is the correct Rust slice range, and why does `line[9..16]` give the wrong answer *without panicking*?
2. In the mod-10 checksum, what does a `-` contribute, and what do `+`, `.`, and letters contribute? Which of these does the common broken implementation get wrong?
3. Why does `tle_checksum` take only the first 68 characters instead of all 69?
4. The ISS line's epoch year field is `08`. What year is that, and what year would `57` be? State the pivot rule.
5. Columns 27–33 of line 2 read `0006703`. What eccentricity does that encode, and what would you get by parsing it as a bare number?
6. Why is a bad-checksum line an `EtlError::Parse` (permanent) rather than an `EtlError::Network` (transient), and what does the retry loop do differently because of that?

The graded quiz for this arc is at the end of the part, on the [Concept-Check: Extraction](./quiz.md) page.
