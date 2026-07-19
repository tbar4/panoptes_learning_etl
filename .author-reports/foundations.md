# Author Report — Part I (Foundations)

Authored the five Foundations pages plus the quiz TOML for the `panoptes_learning_etl` mdBook course. All Rust examples and both Tracing quiz programs were compiled/run in isolation before writing; captured output is what appears in the pages. I did **not** run `mdbook build` (owner runs it centrally).

## Files written

1. `src/00-foundations/how-to-use.md` — sequel framing, predict-then-run TDD loop, concept-before-build, "build pages give spec + test names, not implementation," and an "assumed toolkit" pointer list for readers who skipped the first course.
2. `src/00-foundations/concept-mental-model.md` — the ETL mental model, the "simple now, with the joints an orchestrator needs" thesis, and the four seams (Source trait; E/T/L split; run ledger; idempotent content-addressed writes). Each seam states its real-world complication, the arc that installs it, and where it is spent (Parts V–VI). Derived from `pipeline.rs`, `error.rs`, `domain.rs`.
3. `src/00-foundations/concept-error-handling.md` — new-crate callout for `thiserror`; transient-vs-permanent split; a `BaristaError` (coffee-order) toy mirroring `EtlError` one-for-one; the "one transient bucket for every non-2xx" trap with the correct `classify` (429→retry, 4xx→permanent, 5xx→transient) and the match-order sub-trap (429 arm before 400..=499). Derived from `error.rs`.
4. `src/00-foundations/concept-newtypes.md` — id-safety; a `BinId` toy mirroring `NoradId` one-for-one; the bare-`u32` silent bug, then the `compile_fail` newtype version asserting E0308; ends with the one-for-one mapping paragraph. Derived from `ids.rs`.
5. `src/00-foundations/quiz.md` — short intro + `{{#quiz ../../quizzes/00-foundations.toml}}` (the only quiz include in the part).
6. `quizzes/00-foundations.toml` — 7 questions: 2 Tracing, 3 MultipleChoice, 2 ShortAnswer. Covers the four seams, transient-vs-permanent classification, and newtype id-safety.

## Examples compiled + captured output

Scratch crates under the session scratchpad (`found-scratch` with `thiserror`, `serde` + derive, `serde_json`; `pipe-scratch` with `async-trait`, `chrono`, `thiserror`, `serde`). Rust edition 2024.

- **`barista.rs`** (`rust,ignore`, needs `thiserror`) — compiled and ran. Output (transient/retry_after per variant) captured verbatim into concept-error-handling.md.
- **`classify.rs`** (plain `rust`, std only) — compiled and ran. Output for statuses [200,404,429,400,503,401] captured verbatim.
- **`binid.rs`** (`rust,ignore`, needs serde/serde_json) — compiled and ran. Output (`0042`, `1337`, `Err(...)`, `42`, `0042`) captured verbatim into concept-newtypes.md.
- **`bareu32.rs`** (plain `rust`, std only) — compiled and ran; prints `dispatching a picker to bin 0007`. Captured.
- **mixed-newtype** (`rust,compile_fail`, std only) — `rustc --edition 2024` confirmed **E0308: mismatched types** ("expected `BinId`, found `PalletId`"). Error code asserted in the page.
- **pipeline trait sketch** (`rust,ignore`) — full E/T/L trait set compiled clean in `pipe-scratch` to confirm the sketch is accurate; shown abbreviated in the page.
- **Tracing A** (quiz, std only) — newtype mix-up; `rustc` confirms it does **not** compile (E0308). `answer.doesCompile = false`.
- **Tracing B** (quiz, std only) — `is_transient` match; compiles and prints `false true`. `answer.doesCompile = true`, `answer.stdout = "false true"`.

## UUIDs used (quiz question ids)

- `7e792ad2-bf6f-49b4-aa96-1f74e8532d14` — Tracing: newtype mix-up (doesCompile=false)
- `185081e6-eb9d-4e1b-967d-f4e8999d7f0f` — Tracing: classification order (stdout "false true")
- `c9a7313a-9900-405e-b6d3-0e4c52c32245` — MC: which seam makes re-runs safe
- `6576ab17-c751-47a1-b5e2-b1dcd97faba2` — MC: classify a 404
- `2ebf0641-6a2d-43e7-b6c9-4a74c6e1a968` — MC: which stage is pure/sync (Transform)
- `13344c71-0df5-456a-8d84-0e663febc93d` — SA: method name `is_transient`
- `a6e63be2-84c1-49d4-a09f-e05411bb9882` — SA: `#[serde(transparent)]`

## Assumptions

- Treated `thiserror` as the only *new* crate in this part (crate callout added there); `serde` is assumed from the first course, so the newtypes chapter has no new-crate callout.
- Marked the serde-using `BinId` block `rust,ignore` (with a scratch dep note) rather than plain `rust`, to avoid claiming Playground support for serde's `derive` feature that I could not verify. The `classify` and bare-`u32` blocks are plain `rust` (std only, Playground-safe).
- `answer.stdout` written without a trailing newline, matching the sibling course's convention (`00-orientation-borrowing.toml`).
- Followed the exemplar callout classes: `.callout`, `.callout.trap`, `.callout.predict`.
- Did not modify `SUMMARY.md` (stubs already wired) or run `mdbook build`.
