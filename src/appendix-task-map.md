# Appendix: Task-to-Chapter Map

Each build chapter maps to one task in the [Answer Key](./appendix-answer-key.md).
The reference implementation is `panoptes_etl`; every task's code compiles and its
tests pass (65 in total).

| Part | Chapter | Task | Seam installed |
|---|---|---|---|
| I | Build: Workspace + Domain + `Source` Trait | 0.1–0.5, 1.1 | `Source`/`Transform`/`Sink` |
| II | Build: The CelesTrak Source | 1.2 | E/T/L split |
| II | Build: The JSONL Sink | 1.3 | file contract |
| III | Build: The Space-Track Source | 2.1–2.2 | rate limiting |
| III | Build: The Ledger + `Conjunction` | 2.3 | **run ledger** |
| IV | Build: TheSpaceDevs Source | 3.3 | pagination |
| IV | Build: Retry + Idempotent Sink | 3.1–3.2 | **content-addressed writes** |
| V | Build: The Task Graph | 4.1 | typed DAG |
| V | Build: Executor + Incremental | 4.2 | watermark/retry/gating |
| V | Build: The CLI | 4.3 | wiring |
| V | Capstone: The NASA Source | 4.4 | *(you drive)* |
| VI | Build: The Embed Load Target | 5.1–5.2 | embeddings |
| VI | Build: Retriever + DAG Node | 5.2–5.3 | semantic retrieval |

## The invariants

These held throughout the build and are worth re-checking against your own copy:

1. **One seam per arc.** Every arc ships a working pipeline *and* installs exactly
   one seam an orchestrator later needs.
2. **No build chapter uses machinery no earlier chapter introduced.** If a build
   needs a crate or API with no lead-in, a concept chapter precedes it.
3. **`etl-orchestrate` depends on `etl-core` only.** The executor schedules
   `dyn Source` and cannot name a concrete source; `cargo tree -p etl-orchestrate`
   proves it. Only `panoptes-etl` (the binary) names concrete sources.
4. **Every live API is mocked** with `wiremock`; no test hits the network. Time-
   based tests (limiter, retry) use paused tokio time, so they are deterministic.
5. **The keystone is `Conjunction`.** A conjunction data message is a raw
   collision-avoidance decision — the ground truth a Panoptes `ca_geo` vignette
   is built from.
