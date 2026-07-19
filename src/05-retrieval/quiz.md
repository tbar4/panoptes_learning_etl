# Concept-Check: Retrieval

You have installed the fourth and final seam this arc: the retrieval split — structured filters for the numbers, embeddings for the prose — behind the `Embedder` trait and a brute-force cosine `VectorIndex`. This check pins down the parts that are easy to get *almost* right: which field is meaning-bearing prose (and why the numbers stay out of the index), why embedding is the definitive content-addressed workload, how the `seen`/`content_id` cache guarantees a text is embedded at most once, and the exact behavior of `cosine` and `top_k`.

Three of these are **Tracing** questions: read the program, decide whether it compiles, and if it does, predict its exact output before revealing the answer. Predicting is the point — the cosine and `top_k` traces are the same arithmetic the shipped index runs.

{{#quiz ../../quizzes/05-retrieval.toml}}
