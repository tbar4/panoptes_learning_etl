# Concept: Pagination & Following `next`

> **Kind:** Concept. **The new problem this arc:** the data no longer fits in one response — you must follow a cursor to the end, and defend against a cursor that never ends.

## The idea

CelesTrak handed you every space object in one GET; the whole payload arrived in a single response. TheSpaceDevs (the Launch Library 2 API) does not work that way. There are thousands of launches, so the API returns them a **page** at a time — a hundred rows plus a pointer to the next hundred:

```text
GET /2.2.0/launch/?limit=100&mode=detailed
{
  "next": "https://…/2.2.0/launch/?limit=100&offset=100&mode=detailed",
  "results": [ {…}, {…}, … 100 launches … ]
}
```

The response carries its own continuation. `next` is a full URL for the following page; `results` is this page's rows. To collect everything you **follow the cursor**: fetch, take the rows, read `next`, fetch that, repeat — until `next` comes back `null`. That `null` is the only thing that ends the loop.

This is *cursor* pagination (the server hands you an opaque "where to go next"), as opposed to *offset* pagination (you compute `?offset=` yourself). The distinction rarely matters to the caller: either way, Extract for this source is not one request but a **loop of requests**, and the loop's termination condition lives in the response body, not in your code.

## A runnable toy: draining pages by their cursor

The real `extract` pulls in `reqwest`, `async-trait`, and `chrono`, so it will not run on the playground. Here is the *same shape* on a domain that cannot be mistaken for the answer key — an in-memory "API" that is just a map from URL to page. Fetch a page, take its items, follow its `next`, stop when `next` is `None`:

```rust
use std::collections::HashMap;
use std::collections::HashSet;

// A page: some items, plus an optional cursor to the NEXT page.
#[derive(Clone)]
struct Page {
    items: Vec<String>,
    next: Option<String>,
}

// Stand-in for "the API": a fixed set of pages keyed by URL.
fn fetch(pages: &HashMap<String, Page>, url: &str) -> Page {
    pages[url].clone()
}

// Follow the `next` cursor to the end, guarding against a cyclic `next`.
fn drain(pages: &HashMap<String, Page>, start: &str) -> Result<Vec<String>, String> {
    let mut all = Vec::new();
    let mut visited = HashSet::new();
    let mut next = Some(start.to_string());
    while let Some(url) = next {
        // The cycle guard: refuse to fetch a URL we have already fetched.
        if !visited.insert(url.clone()) {
            return Err(format!("pagination cycle at {url}"));
        }
        let page = fetch(pages, &url);
        all.extend(page.items);
        next = page.next; // follow the cursor; None ends the loop
    }
    Ok(all)
}

fn main() {
    let mut pages = HashMap::new();
    pages.insert("/p1".to_string(), Page { items: vec!["a".into(), "b".into()], next: Some("/p2".into()) });
    pages.insert("/p2".to_string(), Page { items: vec!["c".into()], next: None });
    println!("{:?}", drain(&pages, "/p1"));

    // A buggy server whose last page points back at the first — an infinite loop
    // without the guard.
    let mut cyclic = HashMap::new();
    cyclic.insert("/p1".to_string(), Page { items: vec!["a".into()], next: Some("/p2".into()) });
    cyclic.insert("/p2".to_string(), Page { items: vec!["b".into()], next: Some("/p1".into()) });
    println!("{:?}", drain(&cyclic, "/p1"));
}
```

```text
Ok(["a", "b", "c"])
Err("pagination cycle at /p1")
```

Two moves in that toy are exactly the moves the real source makes. The `while let Some(url) = next` loop is the follow-the-cursor pattern: the loop variable is refilled from each page's own `next`, and a `None` (the API's `null`) is what drains it. And `visited.insert(url.clone())` is the cycle guard, which the next section is entirely about.

## The trap: a cursor that never terminates

Following `next` to the end assumes the chain *reaches* an end. A correct server guarantees it: each `next` points strictly forward, and the last page's `next` is `null`. But your Extract loop is at the mercy of the server, and a buggy or malicious server can hand you a `next` that points **back at a page you already fetched** — or at itself. Now "follow `next` until it is `null`" never terminates. Your process spins forever, hammering the API, filling memory with duplicate rows, and never returning.

This is not paranoia; it is the difference between a loop that is *correct on good input* and one that is *safe on any input*. The Extract stage talks to the outside world, so it must assume the outside world can be wrong.

The defense is a **visited set**: remember every URL you have fetched, and refuse to fetch one twice. The shipped `SpaceDevsSource::extract` does exactly this — one `HashSet` and one line:

```rust,ignore
// crates/etl-sources/src/spacedevs.rs
// (real code — needs reqwest + async-trait + chrono; not playground-safe.
//  The toy above is the runnable version of this loop.)
let mut next = Some(format!("{base}/2.2.0/launch/?limit=100&mode=detailed"));
let mut pages = Vec::new();
// Guard against a cyclic or self-referential `next` from a buggy server.
let mut visited = std::collections::HashSet::new();

while let Some(url) = next {
    if !visited.insert(url.clone()) {
        return Err(EtlError::Parse(format!("pagination cycle at {url}")));
    }
    let body = /* fetch url (with retry — see the next concept) */;
    let page: Page = serde_json::from_str(&body)?;
    next = page.next;
    pages.push(RawRecord { source: self.name().to_string(), payload: body, fetched_at: Utc::now() });
}
Ok(pages)
```

`HashSet::insert` returns `false` when the value was **already present** — so `if !visited.insert(url.clone())` reads as "if this URL was already seen, stop." One page produces one `RawRecord`; the transform parses each page's `results` later. The loop cannot run more than once per distinct URL, so it is guaranteed to terminate on any input the server can produce.

<div class="callout trap">
<span class="callout-label">The infinite-cursor trap</span>
"Follow <code>next</code> until it is <code>null</code>" is correct only if the server is. A <code>next</code> that loops back turns your Extract into an infinite request loop — the kind of bug that does not show up in a two-page test and does show up at 3am against a flaky upstream. A <code>visited</code> set makes termination a property of <em>your</em> code, not a promise you are trusting the server to keep. Note the classification: a detected cycle is an <code>EtlError::Parse</code> — <strong>permanent</strong>, not transient. It is a broken response, and retrying the same broken response would loop again; the retry loop (next concept) must not waste attempts on it.
</div>

## Why one `RawRecord` per page

Notice the loop pushes a *separate* `RawRecord` for each page rather than concatenating all the bodies into one. This keeps the E/T boundary clean: each `RawRecord` is exactly one server response, with its own `fetched_at` provenance, and the `Transform` parses one page's JSON at a time. If a later page is malformed, the parse error names *that* page; the good pages before it are untouched raw records. Extract's job is to bring back every page's bytes faithfully — deciding what those bytes *mean* is still the transform's job, one page at a time.

## The one-for-one mapping

Everything in the toy maps onto the real source with the domain swapped and the I/O made real. The toy's `HashMap<String, Page>` fetch becomes a real `reqwest::get` against `{base_url}/2.2.0/launch/?limit=100&mode=detailed`; the toy's `Page { items, next }` becomes the deserialized `Page { results, next }`; the toy's `all.extend(page.items)` becomes `pages.push(RawRecord { … })` (one raw record per page, parsed later); and the `visited` `HashSet<String>` is *identical* in both — the cycle guard does not change one character when the network becomes real. The shape is the loop; only the fetch inside it becomes async and fallible.

## Questions to lock

1. What ends the follow-`next` loop under normal operation — and where does that termination signal live, in your code or in the server's response?
2. What can a buggy server do to a naive "follow `next` until `null`" loop, and what single data structure defends against it?
3. `HashSet::insert` returns a `bool`. What does `false` mean, and how does `if !visited.insert(url)` use that to detect a cycle?
4. Why is a detected pagination cycle classified as `EtlError::Parse` (permanent) rather than a transient error — and what does that classification save the retry loop from doing?
5. Why does the source emit one `RawRecord` per page instead of concatenating every page's body into one record?

The graded quiz for this arc is at the end of the part, on the [Concept-Check: Pagination](./quiz.md) page.
