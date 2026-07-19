# Concept-Check: Extraction

You have installed two seams this arc: the `Source` / `Transform` / `Sink` traits, and the Extract / Transform / Load split, with the CelesTrak source and the JSONL sink as the first concrete implementations. This check pins down the parts that are easy to get *almost* right — the TLE checksum arithmetic, why `Row` carries a tag, which stage owns parsing, and the object-safety property that lets the orchestrator schedule a `dyn Source` later.

Two of these are **Tracing** questions: read the program, decide whether it compiles, and if it does, predict its exact output before revealing the answer. Predicting is the point.

{{#quiz ../../quizzes/01-extraction.toml}}
