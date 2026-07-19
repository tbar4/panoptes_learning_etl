# Author report — Part V: The Orchestration Arc

Authored 2026-07-19. Teaches the shipped code in `crates/etl-orchestrate/src/{graph.rs,executor.rs}` and `crates/panoptes-etl/src/{cli.rs,main.rs}`. No `mdbook build` was run (per instructions).

## Files written (under `panoptes_learning_etl/`)

| File | Kind | Notes |
| --- | --- | --- |
| `src/04-orchestration/concept-dag.md` | Concept | Typed DAG, Kahn's sort, BTreeSet determinism, cycle detection. Playground-safe toy mirror + hidden-line cycle example. |
| `src/04-orchestration/concept-dependency-inversion.md` | Concept | `etl-orchestrate` → `etl-core` only; crate boundary as proof; real `cargo tree` output; runnable `dyn Job` scheduler toy. |
| `src/04-orchestration/build-task-graph.md` | Build (spec only) | `Dag` + `topological_order`, Task 4.1. |
| `src/04-orchestration/build-executor-incremental.md` | Build (spec only) | `Executor`/`Pipeline`/`RunReport`: retries, freshness-skip, failure-gating, Task 4.2. |
| `src/04-orchestration/build-cli.md` | Build (spec only) | `panoptes-etl` CLI (run/backfill/embed/status), Task 4.3. |
| `src/04-orchestration/capstone-nasa.md` | Capstone (spec only, no walkthrough) | NASA source, `NASA_API_KEY` as query param, reader chooses endpoint + `Row` mapping, Task 4.4. |
| `src/04-orchestration/quiz.md` | Quiz intro | Points to `../../quizzes/04-orchestration.toml`. |
| `quizzes/04-orchestration.toml` | Quiz | 7 questions. |

SUMMARY.md already referenced all these paths — no edits needed there.

## Convention compliance

- Concept pages end with "Questions to lock" + a pointer to `./quiz.md`. `{{#quiz}}` appears ONLY on `quiz.md`.
- Build pages are spec + test names + build loop, no implementation.
- Capstone is spec only: no test names, no build loop, no walkthrough. Points to the `// CAPSTONE (you drive, unassisted)` comment in `main.rs`.

## Fence honesty

- **concept-dag.md**: both examples are pure `std` (`HashMap`/`HashSet`/`BTreeSet`/`Vec`) → ```rust (playground-safe). The cycle example uses hidden `#` lines to re-supply the `Dag` impl.
- **concept-dependency-inversion.md**: the `Job` scheduler toy is pure `std` → ```rust. The `cargo tree` output and the `Cargo.toml` snippet are ```text / ```toml (not run).

## Verification (compiled in scratch project + captured real output)

Scratch: `.../scratchpad/toys/` (cargo) and `.../scratchpad/*.rs` (rustc, edition 2021).

| Example | Result |
| --- | --- |
| concept-dag diamond toy (`dag.rs`) | `["sift", "grease", "whisk", "bake"]` / `stable: true` — captured verbatim. Confirms sorted ready-set tie-break (`grease` < `whisk`). |
| concept-dag cycle toy (`cycle.rs` + hidden-line reconstruction) | `rejected: cycle detected: ordered 0 of 2 tasks` — captured verbatim. |
| concept-dependency-inversion scheduler toy (`schedule.rs`) | `backup did 12 units` / `report did 3 units` / `total: 15` — captured verbatim. |
| quiz Tracing #1 `trace_topo.rs` | compiles; stdout `a,b,c,d`. |
| quiz Tracing #2 `trace_gate.rs` | compiles; stdout `ran=["extract", "clean"]` / `blocked=["report"]`. |
| `cargo tree -p etl-orchestrate` | Real output captured; only `chrono` + `etl-core` direct deps, `etl-sources` absent from the whole closure. Trimmed for the page. |

## Quiz composition (`quizzes/04-orchestration.toml`)

7 questions, all `id`s from `uuidgen`:

1. Tracing — deterministic topological order over a diamond (`a,b,c,d`). std-only, rustc-verified.
2. Tracing — failure-gating: blocked node never runs. std-only, rustc-verified. Multi-line stdout.
3. MultipleChoice — why the ready-set is a `BTreeSet` (determinism).
4. MultipleChoice — cycle detection via length mismatch.
5. ShortAnswer — the one crate `etl-orchestrate` depends on (`etl-core`).
6. MultipleChoice — only `panoptes-etl` names concrete sources.
7. ShortAnswer — blocked vs failed (`blocked`).

Coverage: topological ordering (1,3), cycle detection (4), failure-gating (2,7), dependency inversion (5,6). Meets MC + ShortAnswer + ≥2 Tracing.

## Notes / decisions

- Plan Phase 4 labels the quiz `05-orchestration.toml`, but the task instructions and the existing `SUMMARY.md` both say `04-orchestration.toml`. Used `04-` to match SUMMARY and the `quiz.md` link.
- `Command` enum has four variants (`Run`/`Backfill`/`Embed`/`Status`) per shipped `cli.rs`; build-cli.md documents all four (the plan mentioned three; `Embed` belongs to Part VI but is present in the shipped CLI, so it is listed honestly).
- Did NOT run `mdbook build` (per instructions). Tracing programs are rustc-verified standalone, which is what the build-time quiz validation would check.
