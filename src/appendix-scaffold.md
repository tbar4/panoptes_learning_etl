# Appendix: Workspace Scaffold

The finished workspace, annotated with the chapter that creates each file. The
dependency arrow always points *toward* `etl-core`; only the binary names
concrete sources.

## The tree

```text
panoptes_etl/
├── Cargo.toml                         # workspace manifest        (Part II)
├── crates/
│   ├── etl-core/                      # the seams — depends on nothing in-workspace
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs                 # module wiring + re-exports
│   │       ├── error.rs               # EtlError taxonomy         (Part I)
│   │       ├── ids.rs                 # NoradId newtype           (Part I)
│   │       ├── domain.rs              # SpaceObject/OrbitalElements/Conjunction/Launch (Part I–III)
│   │       ├── pipeline.rs            # Source/Transform/Sink, RawRecord, Row (Part II)
│   │       ├── sink_jsonl.rs          # JsonlSink — the file contract (Part II)
│   │       ├── ledger.rs              # RunRecord/Outcome/Ledger  (Part III)
│   │       ├── retry.rs               # with_retry, RetryPolicy   (Part IV)
│   │       ├── content.rs             # content_id, IdempotentSink (Part IV)
│   │       ├── embed.rs               # Embedder trait            (Part VI)
│   │       └── vector.rs              # cosine, VectorIndex, EmbedSink (Part VI)
│   ├── etl-sources/                   # concrete sources — depends on etl-core
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── tle.rs                 # hand-rolled TLE parser    (Part II)
│   │       ├── celestrak.rs           # CelestrakSource/Transform (Part II)
│   │       ├── http.rs                # classify_status taxonomy funnel (Part II–III)
│   │       ├── limiter.rs             # TokenBucket               (Part III)
│   │       ├── spacetrack.rs          # SpaceTrackSource/Transform (Part III)
│   │       ├── spacedevs.rs           # SpaceDevsSource/Transform (Part IV)
│   │       └── voyage.rs              # VoyageEmbedder            (Part VI)
│   ├── etl-orchestrate/               # the DAG executor — depends on etl-core ONLY
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── graph.rs               # Dag, TaskId, topological_order (Part V)
│   │       └── executor.rs            # Pipeline, Executor, RunReport (Part V)
│   └── panoptes-etl/                  # the binary — the only crate naming concrete sources
│       ├── Cargo.toml
│       └── src/
│           ├── main.rs                # wiring + build_executor   (Part V)
│           └── cli.rs                 # clap surface + status     (Part V)
└── data/                             # gitignored ETL output (JSONL) + runs.jsonl ledger
```

## The workspace manifest

```toml
[workspace]
resolver = "2"
members = ["crates/etl-core", "crates/etl-sources", "crates/etl-orchestrate", "crates/panoptes-etl"]

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }  # derive on the domain model
serde_json = "1"                                   # JSONL wire format
chrono = { version = "0.4", features = ["serde"] } # epochs, TCA, watermarks
thiserror = "2"                                     # the EtlError taxonomy (libs)
anyhow = "1"                                        # the binary's error type
async-trait = "0.1"                                 # dyn-compatible async traits
tokio = { version = "1", features = ["full"] }      # runtime, fs, time
reqwest = { version = "0.12", features = ["json"] } # HTTP client
clap = { version = "4", features = ["derive"] }     # CLI
sha2 = "0.10"                                        # content-addressing
itertools = "0.13"                                   # iterator helpers
wiremock = "0.6"                                      # mock every live API (dev)
pretty_assertions = "1"                               # readable test diffs (dev)
```

## Per-crate dependency matrix

| crate | etl-core | serde/json | chrono | thiserror | async-trait | tokio | reqwest | sha2 | clap | anyhow | wiremock (dev) |
|-------|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| **etl-core** | — | ✓ | ✓ | ✓ | ✓ | ✓ | | ✓ | | | |
| **etl-sources** | ✓ | ✓ | ✓ | | ✓ | ✓ | ✓ | | | | ✓ |
| **etl-orchestrate** | ✓ | | ✓ | | (dev) | (dev) | | | | | |
| **panoptes-etl** | ✓ | ✓ | ✓ | | | ✓ | | | ✓ | ✓ | |

The empty **etl-orchestrate → reqwest / etl-sources** cells are the point: the
executor cannot reach a concrete source. `cargo tree -p etl-orchestrate` shows
only `etl-core` (+ chrono/serde). That is the dependency-inversion seam, enforced
by the build, not by convention.

## Expected test progression

Cumulative workspace test count after each part (run `cargo test`):

| After Part | Arc | Cumulative `cargo test` |
|---|---|---|
| I | Foundations (etl-core types) | 13 |
| II | Extraction (CelesTrak) | 23 |
| III | Authenticated (Space-Track) | 31 |
| IV | Paginated (TheSpaceDevs) | 47 |
| V | Orchestration (DAG + CLI) | ~57 |
| VI | Retrieval (RAG) | 65 |

Final workspace: **65 tests** — etl-core 29, etl-sources 26, etl-orchestrate 7,
panoptes-etl 3 — clippy clean.
