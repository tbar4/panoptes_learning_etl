# How to Use This Course

This is a **sequel**. It assumes you finished the first Panoptes course — or that you already think fluently in ownership, borrowing, `serde`, traits, `async`/`tokio`, `wiremock`, and `clap`. We do not re-teach the language here. What is new is the *shape of ETL*: partial failure, retries, idempotency, and the discipline of leaving seams for a future you cannot yet see. That is the whole subject, and it is what these chapters build.

## The two chapter types

Every arc alternates between two kinds of chapter, and they ask different things of you.

**Concept chapters** are the lecture. Read them without touching the keyboard. They are deliberately verbose and they argue *why* before *how* — because in ETL the "why" (why this error is transient, why this write must be idempotent) is the part that bites you in production, not the syntax. Each concept chapter ends with a short **Questions to lock** list. If any of those are fuzzy, that is the signal to stop and re-read before moving to the build.

**Build chapters** are the pair-programming session. Here you write the code. A build chapter gives you the **spec and the test names — never the implementation.** You drive. Each one follows the same test-driven loop you already know:

1. **Write the failing test.** You write it, from the spec and the test name we give you — not by copying an answer.
2. **Predict the failure.** Say out loud (or note down) exactly what the compiler or the test runner will report.
3. **Run it and check your prediction.** The gap between prediction and reality is the lesson.
4. **Write the minimal implementation.** Just enough to make the test pass. No more.
5. **Run again, watch it pass.**
6. **Commit.**

<div class="callout">
<span class="callout-label">Concept before build, always</span>
The arcs are sequenced so that no build chapter uses machinery an earlier chapter did not first demonstrate. If a build feels like it is asking for something out of nowhere, you probably skipped its concept chapter. Go back — the concept is where the shape was installed.
</div>

## The predict-then-run habit

The single most valuable thing you can do in a build chapter is **write down your prediction before you run**. A wrong prediction followed by the real message is the fastest way to learn what the types are actually enforcing. This matters more in an ETL than in a pure library, because many of the failures here are *runtime* failures — a mock returning a `429`, a re-run that should write zero new rows — and predicting them trains the instinct you need when the real network misbehaves.

## The concept-check quizzes

Each part ends with one graded quiz, carried only on that part's quiz page. These are not busywork — they target the exact misconceptions that cause bugs three arcs later: retrying a `404` forever, confusing two integer ids, forgetting which stage owns parsing. Answer honestly before revealing. Some questions are **Tracing** questions: a short program whose output — or whose refusal to compile — you must predict. Those are compiled by the real Rust compiler when the book builds, so the answer is not a matter of opinion.

## Assumed toolkit — if you skipped the first course

This course will not stop to explain the following. If any of these is genuinely new to you, work the first Panoptes course (or an equivalent) before starting Part II, because every build chapter leans on them:

- **Ownership, borrowing, and moves** — why a function takes `&self` vs `self`, when a value is moved, why the borrow checker rejects aliasing a mutable borrow.
- **`serde`** — `#[derive(Serialize, Deserialize)]`, container and field attributes (`#[serde(tag = …)]`, `rename_all`, `transparent`), and `serde_json`.
- **Traits and trait objects** — defining a trait, `impl Trait for T`, `dyn Trait`, and object safety.
- **`async`/`await` and `tokio`** — `async fn`, `.await`, `#[tokio::main]`/`#[tokio::test]`, and why async traits need `async_trait`.
- **`wiremock`** — starting a mock HTTP server in a test and pointing a client with an injectable base URL at it. Every source in this course is tested this way; **no test hits a live network.**
- **`clap`** — deriving a CLI with subcommands.
- **The TDD loop itself** — red, green, refactor, commit.

<div class="callout trap">
<span class="callout-label">The one anti-pattern to avoid</span>
Copying a test or an implementation from the answer-key plan <em>before</em> attempting it yourself. It feels productive and teaches almost nothing. In this course the answer key is the verified implementation plan — treat it as a check, not a script. The distance between your version and the plan's version is precisely where the learning lives.
</div>
