# Summary

[Introduction](./intro.md)
[The Panoptes Thesis at a Glance](./thesis-architecture.md)
[The Architecture at a Glance](./architecture.md)

# Part I — Foundations

- [How to Use This Course](./00-foundations/how-to-use.md)
- [Concept: The ETL Mental Model & the Four Seams](./00-foundations/concept-mental-model.md)
- [Concept: Error Handling for Fallible I/O](./00-foundations/concept-error-handling.md)
- [Concept: Newtypes & Id-Safety](./00-foundations/concept-newtypes.md)
- [Concept-Check: Foundations](./00-foundations/quiz.md)

# Part II — The Extraction Arc (CelesTrak)

- [Concept: The `Source` Trait & the E/T/L Boundary](./01-extraction/concept-source-etl.md)
- [Concept: Parsing the TLE Format](./01-extraction/concept-tle.md)
- [Build: Workspace + Domain Model + the `Source` Trait](./01-extraction/build-workspace-domain.md)
- [Build: The CelesTrak Source — Extract + Transform](./01-extraction/build-celestrak.md)
- [Build: The JSONL Sink — Load](./01-extraction/build-jsonl-sink.md)
- [Concept-Check: Extraction](./01-extraction/quiz.md)

# Part III — The Authenticated Arc (Space-Track)

- [Concept: Session Auth & Secrets from the Environment](./02-authenticated/concept-auth-secrets.md)
- [Concept: Rate Limiting with a Token Bucket](./02-authenticated/concept-rate-limiting.md)
- [Concept: The Run Ledger](./02-authenticated/concept-ledger.md)
- [Build: The Space-Track Source](./02-authenticated/build-spacetrack.md)
- [Build: The Ledger + the `Conjunction` Keystone Entity](./02-authenticated/build-ledger-conjunction.md)
- [Concept-Check: Authentication](./02-authenticated/quiz.md)

# Part IV — The Paginated Arc (TheSpaceDevs)

- [Concept: Pagination & Following `next`](./03-paginated/concept-pagination.md)
- [Concept: Retry with Backoff & 429 / Retry-After](./03-paginated/concept-retry-backoff.md)
- [Concept: Idempotent, Content-Addressed Writes](./03-paginated/concept-content-addressing.md)
- [Build: The TheSpaceDevs Source — Pagination](./03-paginated/build-spacedevs.md)
- [Build: Retry Middleware + the Idempotent Sink](./03-paginated/build-retry-idempotent.md)
- [Concept-Check: Pagination](./03-paginated/quiz.md)

# Part V — The Orchestration Arc

- [Concept: Modeling a Pipeline as a Typed DAG](./04-orchestration/concept-dag.md)
- [Concept: Dependency Inversion — the Executor Knows Only the Trait](./04-orchestration/concept-dependency-inversion.md)
- [Build: The Task Graph + Topological Execution](./04-orchestration/build-task-graph.md)
- [Build: Retries, Watermarks & Incremental Runs](./04-orchestration/build-executor-incremental.md)
- [Build: The `panoptes-etl` CLI](./04-orchestration/build-cli.md)
- [Capstone: Add the NASA Source (You Drive)](./04-orchestration/capstone-nasa.md)
- [Concept-Check: Orchestration](./04-orchestration/quiz.md)

# Part VI — The Retrieval Arc (RAG)

- [Concept: Structured vs Semantic Retrieval](./05-retrieval/concept-structured-vs-semantic.md)
- [Concept: The `Embedder` Trait & Content-Addressed Embeddings](./05-retrieval/concept-embedder.md)
- [Build: The Embed Load Target](./05-retrieval/build-embed-sink.md)
- [Build: The Brute-Force Retriever + Wiring it as a DAG Node](./05-retrieval/build-retriever-node.md)
- [Concept-Check: Retrieval](./05-retrieval/quiz.md)

# Part VII — Wrap-Up

- [Where This Plugs Into Panoptes](./06-wrap/panoptes-integration.md)
- [The Graduation Path](./06-wrap/graduation-path.md)

---

[Appendix: Task-to-Chapter Map](./appendix-task-map.md)
[Appendix: The Answer Key (Full Task Plan)](./appendix-answer-key.md)
[Appendix: Workspace Scaffold](./appendix-scaffold.md)
[Appendix: Reference Books](./appendix-books.md)
