# The Architecture at a Glance

<style>
/* Render each diagram on a fixed light "plate" so it stays legible on the book's
   dark navy theme (which would otherwise force the SVG text light-on-light). */
.mermaid { background:#f3f5f8; border:1px solid #d6dee7; border-radius:12px; padding:1.1rem 0.6rem; overflow-x:auto; }
.mermaid text, .mermaid tspan, .mermaid span, .mermaid div, .mermaid p, .mermaid .nodeLabel, .mermaid .edgeLabel, .mermaid foreignObject * { fill:#1b2530 !important; color:#1b2530 !important; }
.mermaid .edgeLabel, .mermaid .edgeLabel rect, .mermaid .edgeLabel div, .mermaid .edgeLabel span { background-color:#f3f5f8 !important; }
</style>

How the structs, enums, traits, and functions of the finished `panoptes_etl` connect. Four crates, built in dependency order — the arrow always points at `etl-core`:

`etl-sources` → `etl-orchestrate` → `panoptes-etl`, and all three → `etl-core`. `etl-orchestrate` depends on `etl-core` **only**: it schedules `dyn Source` and never names a concrete source, which is what lets your NASA capstone drop into the DAG with zero executor changes.

## The pipeline

Each source extracts raw bytes, a transform parses them into typed `Row`s, and an idempotent sink appends JSONL that `panoptes-gen` reads. The `Executor` runs the pipelines in dependency order — retrying transient failures, gating on upstream failure, skipping sources still fresh — and the retrieval arc embeds the prose fields into a vector index.

```mermaid
%%{init:{'theme':'base','flowchart':{'htmlLabels':false},'themeVariables':{'fontSize':'13px','lineColor':'#7c8b9a','textColor':'#1b2530','primaryColor':'#e9eef3','primaryBorderColor':'#9fb0bf','primaryTextColor':'#1b2530','clusterBkg':'#fbfcfd','clusterBorder':'#cbd5de'}}}%%
flowchart TB
  cli{{"panoptes-etl CLI · run · backfill · embed · status"}}:::bin
  ct[("CelesTrak · open TLE GET")]:::ext --> csrc["CelestrakSource"]:::src
  st[("Space-Track · login + cookie")]:::ext --> ssrc["SpaceTrackSource"]:::src
  sd[("TheSpaceDevs · paginated")]:::ext --> dsrc["SpaceDevsSource"]:::src
  bucket["TokenBucket · rate limit"]:::src -.-> ssrc
  retry["with_retry · backoff on transient EtlError"]:::core -.wraps.-> dsrc
  csrc --> raw["RawRecord"]:::core
  ssrc --> raw
  dsrc --> raw
  raw --> xf["Transform · parse TLE / CDM / launch JSON"]:::src
  xf --> row["Row · SpaceObject · Conjunction · Launch"]:::core
  row --> idem["IdempotentSink · content_id() dedupe"]:::core
  idem --> jsonl[("space_objects · conjunctions · launches · .jsonl")]:::store
  exec["Executor · Dag topological order · gate on failure · skip fresh"]:::orch
  exec -.drives.-> csrc
  exec -.drives.-> ssrc
  exec -.drives.-> dsrc
  ledger["Ledger · RunRecord · watermark"]:::core -.records.-> exec
  jsonl --> embed["EmbedSink · Embedder → VoyageEmbedder · prose only"]:::core
  embed --> vec[("embeddings.jsonl")]:::store
  vec --> idx["VectorIndex::top_k() · cosine"]:::core
  jsonl --> gen[("panoptes-gen reads the file contract")]:::ext
  cli -.-> exec

  classDef core fill:#e2efec,stroke:#1f8f80,color:#12231f;
  classDef src fill:#e6ecf5,stroke:#4a6fa5,color:#182436;
  classDef orch fill:#f6ead4,stroke:#b9791f,color:#3a2a10;
  classDef bin fill:#efe6f4,stroke:#7d5ba6,color:#2c1f36;
  classDef store fill:#eef1f5,stroke:#6b7a89,color:#1b2530;
  classDef ext fill:#f0f1f3,stroke:#9aa7b3,color:#3a444e,stroke-dasharray:4 3;
```

*Teal = `etl-core` · slate = `etl-sources` · amber = `etl-orchestrate` · plum = `panoptes-etl` · cylinders = files · dashed = external API or downstream.*

## Type relationships

The E/T/L trait triad — `Source`, `Transform`, `Sink` — is the spine. Concrete sources implement `Source`; the domain types are the payload of the `Row` enum the sink writes; and `Pipeline`/`Executor` compose the traits as boxed trait objects, which is what lets the orchestrator stay ignorant of any concrete source. The keystone is `Conjunction` — a real conjunction data message is the ground truth of a `ca_geo` collision-avoidance vignette.

```mermaid
%%{init:{'theme':'neutral','themeVariables':{'fontSize':'13px','lineColor':'#7c8b9a'}}}%%
classDiagram
  namespace etl_core {
    class NoradId {
      <<newtype>>
      +u32 inner
    }
    class OrbitClass {
      <<enum>>
      LEO
      MEO
      GEO
      HEO
    }
    class OrbitalElements {
      +DateTime epoch
      +f64 inclination_deg
      +f64 eccentricity
      +f64 mean_motion_rev_per_day
      +orbit_class() OrbitClass
    }
    class SpaceObject {
      +NoradId norad_id
      +String name
      +String intl_designator
      +OrbitalElements elements
    }
    class Conjunction {
      +String id
      +DateTime tca
      +f64 miss_distance_km
      +f64 collision_probability
      +NoradId primary
      +NoradId secondary
    }
    class Launch {
      +String id
      +String name
      +DateTime net
      +String provider
      +String description
    }
    class Row {
      <<enum>>
      SpaceObject
      Conjunction
      Launch
    }
    class EtlError {
      <<enum>>
      Network
      RateLimited
      Auth
      Permanent
      Parse
      Io
    }
    class RawRecord {
      +String source
      +String payload
      +DateTime fetched_at
    }
    class Source {
      <<trait>>
      +name() str
      +extract() Vec~RawRecord~
    }
    class Transform {
      <<trait>>
      +transform(RawRecord) Vec~Row~
    }
    class Sink {
      <<trait>>
      +load(rows) usize
    }
    class Ledger {
      +append(RunRecord)
      +last_success(source) RunRecord
    }
    class RunRecord {
      +String source
      +Outcome outcome
      +DateTime watermark
      +usize rows_written
    }
    class Outcome {
      <<enum>>
      Success
      Failed
    }
    class Embedder {
      <<trait>>
      +embed(texts) vectors
    }
    class VectorIndex {
      +top_k(query, k) hits
    }
  }
  namespace etl_sources {
    class CelestrakSource
    class SpaceTrackSource
    class SpaceDevsSource
    class TokenBucket
    class VoyageEmbedder
  }
  namespace etl_orchestrate {
    class Dag {
      +add_task(id) TaskId
      +add_dep(task, dep)
      +topological_order() Vec~TaskId~
    }
    class Pipeline {
      +Box~Source~ source
      +Box~Transform~ transform
      +Box~Sink~ sink
    }
    class Executor {
      +run() RunReport
    }
  }
  SpaceObject *-- NoradId
  SpaceObject *-- OrbitalElements
  OrbitalElements ..> OrbitClass
  Conjunction *-- NoradId
  Row *-- SpaceObject
  Row *-- Conjunction
  Row *-- Launch
  RunRecord *-- Outcome
  CelestrakSource ..|> Source
  SpaceTrackSource ..|> Source
  SpaceDevsSource ..|> Source
  VoyageEmbedder ..|> Embedder
  Pipeline o-- Source
  Pipeline o-- Transform
  Pipeline o-- Sink
  Executor *-- Dag
  Executor o-- Pipeline
  Executor *-- Ledger
```

*`*--` composition · `o--` aggregation (boxed trait object) · `..|>` implements · `«enum»`/`«trait»`/`«newtype»` stereotypes · `~T~` = generic parameter.*

## Per-crate inventory

| Crate | Owns | Key types |
|---|---|---|
| **etl-core** | The seams — domain model + E/T/L traits | `OrbitClass` · `EtlError` · `Row` · `Outcome` (enums); `NoradId`; `SpaceObject` · `OrbitalElements`; `Conjunction` (keystone) · `Launch`; `Source` · `Transform` · `Sink` · `Embedder` (traits); `Ledger` · `JsonlSink` · `IdempotentSink` · `VectorIndex`; `with_retry` · `content_id` · `cosine` |
| **etl-sources** | Concrete sources + the machinery each API forces | `parse_tle`; `CelestrakSource` · `SpaceTrackSource` (+ `Credentials`) · `SpaceDevsSource`; `TokenBucket`; `classify_status`; `VoyageEmbedder` |
| **etl-orchestrate** | The DAG executor (depends on `etl-core` only) | `Dag` · `TaskId`; `topological_order`; `Pipeline`; `Executor` · `RunReport` |
| **panoptes-etl** | The binary — the only crate that names concrete sources | `Cli` · `Command` (`run` / `backfill` / `embed` / `status`); `build_executor`; *NASA source = your capstone* |

The full, verified code behind every box is in the [Answer Key](./appendix-answer-key.md).
