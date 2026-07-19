# Concept-Check: Pagination

You installed three seams this arc: cursor pagination with a cycle guard, `with_retry` (capped exponential backoff, transient-only, `Retry-After` wins), and content-addressed idempotent writes. This check pins down the parts that are easy to get *almost* right — where the follow-`next` loop terminates and how it defends against a cursor that never does, exactly which failures a retry loop should and should not retry (and how long it waits), and why the content key is fallible.

Three of these are **Tracing** questions: read the program, decide whether it compiles, and if it does, predict its exact output before revealing the answer. Predicting is the point.

{{#quiz ../../quizzes/03-paginated.toml}}
