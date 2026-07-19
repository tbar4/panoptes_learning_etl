# Concept-Check: Orchestration

This arc installed the seam that composes everything you built before it: the pipelines became nodes in a typed DAG, execution order became a *computed* property (Kahn's sort with a deterministic, sorted ready-set), and the executor learned two things a bare sort cannot do — skip a still-fresh source, and gate a node whose upstream failed. Above it all sits the dependency-inversion property that is the real payoff of the course: the executor depends on `etl-core` *only* and schedules `dyn Source`, so a fourth source drops in with zero orchestrator changes.

This check pins down the parts that are easy to get *almost* right — why the ready-set is a `BTreeSet` and not a queue, how cycle detection falls out of a length comparison, the difference between a *blocked* node and a *failed* one, and what `cargo tree` proves about the crate boundary.

Two of these are **Tracing** questions: read the program, decide whether it compiles, and if it does, predict its exact output before revealing the answer. Predicting is the point.

{{#quiz ../../quizzes/04-orchestration.toml}}
