# Author Report — Part III: The Authenticated Arc (Space-Track)

Authored 2026-07-19. All content derived from the verified shipped code in
`panoptes_etl` (`crates/etl-sources/src/{spacetrack,limiter,http}.rs`,
`crates/etl-core/src/ledger.rs`) and Phase 2 of the course plan. Where the plan
and the shipped code differ, the SHIPPED reality was taught.

## Files written (under panoptes_learning_etl/)

- `src/02-authenticated/concept-auth-secrets.md` — cookie-session login handshake
  + secrets from the environment, never in the URL/on disk. Toy: members-only
  clubhouse. Maps to `Credentials`/`SpaceTrackSource::extract`.
- `src/02-authenticated/concept-rate-limiting.md` — hand-rolled token bucket, why
  not `governor`, and paused-clock (`start_paused = true`) determinism. Toy:
  logical-clock bucket. Maps to `TokenBucket`.
- `src/02-authenticated/concept-ledger.md` — append-only JSONL run ledger as the
  seed of incrementality/retry/backfill; `last_success` filters failures. Toy:
  backup log. Maps to `Ledger`/`RunRecord`/`Outcome`.
- `src/02-authenticated/build-spacetrack.md` — build page (spec + test names, no
  impl): Part A `TokenBucket` (limiter.rs), Part B `Credentials` +
  `SpaceTrackSource` (E), Part C `SpaceTrackTransform` (T → `Row::Conjunction`).
- `src/02-authenticated/build-ledger-conjunction.md` — build page: the
  `Conjunction` keystone recap (id-safety payoff) + the `Ledger` (ledger.rs).
- `src/02-authenticated/quiz.md` — intro + `{{#quiz ../../quizzes/02-authenticated.toml}}`.
- `quizzes/02-authenticated.toml` — 7 questions (3 Tracing, 2 MultipleChoice,
  2 ShortAnswer).

## Convention compliance

- Concept pages end with a "Questions to lock" list + a one-line pointer to
  `./quiz.md`. The `{{#quiz}}` include appears ONLY on `quiz.md`.
- Playground-safe examples are ```rust (clubhouse, logical-clock bucket, backup
  ledger); shipped code using reqwest/tokio/chrono/serde is ```rust,ignore with a
  dep note; the ledger page carries one ```rust,compile_fail asserting E0369.
- Callout classes match the house style: `callout`, `callout trap`,
  `callout predict`. Build pages are spec-only (no implementation shown beyond
  shipped-signature recaps that concept pages already introduced).

## Verification (scratch cargo project, cargo 1.96.0, edition 2024)

Every ```rust block was compiled and RUN; outputs pasted from real execution:
- clubhouse toy → session handshake + leaky-vs-safe URL output.
- token-bucket toy → burst of 3 Ok, 4th `Err(2.0)`, refill Ok at t=2.
- ledger toy → watermark `Some(200)` after failure, `Some(202)` after success,
  `None` for unknown job.
- compile_fail (missing `PartialEq` on `Outcome`) → `error[E0369]`, confirmed on
  BOTH edition 2021 and 2024 (edition-robust).

Quiz Tracing programs (std-only, rustc-compiled, unique uuidgen ids):
- token-bucket refill/clamp → stdout `2` (doesCompile true).
- last_success filter → stdout `Some(150)` (doesCompile true).
- Set-Cookie `split(';').next()` → stdout `session=abc123` (doesCompile true).

TOML validated with tomllib: 7 questions, type mix {Tracing:3, MC:2,
ShortAnswer:2}, all ids present and unique.

## Shipped-vs-plan reconciliation (taught the shipped code)

- Test names use the SHIPPED names: `login_then_query_uses_cookie`,
  `login_failure_is_auth_error`, `transform_parses_cdm_to_conjunction`,
  `bucket_allows_burst_up_to_capacity`, `bucket_blocks_when_empty_then_refills`,
  `append_then_last_success_reads_back`, `last_success_ignores_failures`,
  `watermark_advances_across_successful_runs` (plan's `respects_rate_limit`,
  `missing_credentials_is_auth_error`, `watermark_advances_across_runs` differ).
- `Outcome::Failed { reason: String }` (struct variant, per shipped code), not the
  plan's `Failed(String)`.
- Login and query BOTH funnel non-2xx through `http::classify_status`
  (429→RateLimited, 401/403→Auth, other 4xx→Permanent, 5xx→Network) — taught as
  the transient-vs-permanent-at-login point.
- CDM quirks taught from shipped transform: numbers arrive as JSON strings;
  `MIN_RNG` is meters → `/1000.0` for km; `PC` optional → `0.0`; `id` is the
  `{primary}_{secondary}_{TCA}` composite (content-addressing deferred to Part IV).

## Not done (out of scope / done centrally)

- Did NOT run `mdbook build` (per instructions — done centrally).
- SUMMARY.md already references all six pages; no edits needed.
