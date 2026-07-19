# Where This Plugs Into Panoptes

> **Kind:** Wrap-up.

You have built the pipeline, the seams, and a real mini-orchestrator over them. This chapter is the one that makes all of it *count*: it shows where the files you write are read, and why the whole ETL was worth building. The ETL was never the point. Grounding Panoptes' scenarios in real orbital data was the point — and that grounding is a handful of files on disk.

## The connection is a file contract, not a call

The single most important design decision in this course is one you never had to write code for: `panoptes_etl` and Panoptes are joined by **files on disk**, not by a shared database or an in-process crate dependency. The ETL runs, writes normalized JSONL to an output directory, and stops. Later — minutes or days later — `panoptes-gen` reads those files when it builds vignettes. Neither crate imports the other. Neither holds a lock the other waits on.

This mirrors Panoptes' own existing `scenarios/` directory: a producer writes, a consumer reads, and the contract between them is a serialization format nobody can silently break, because breaking it means a `serde` deserialize error at the boundary — loud, immediate, and impossible to miss.

The contract has exactly one shape. Every entity the ETL emits is a variant of one enum:

```rust,ignore
#[derive(Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum Row {
    SpaceObject(SpaceObject),
    Conjunction(Conjunction),
    Launch(Launch),
}
```

Because `Row` is internally tagged by `kind`, every line of every output file is self-describing. A consumer can read a mixed stream and dispatch on `kind` without a schema handshake. That is the whole file contract, in one derive.

## What each file carries

The binary writes four artifacts into the output directory, plus the ledger. Each has one reader and one job:

| File | Carries | Read by |
|---|---|---|
| `space_objects.jsonl` | `Row::SpaceObject` — NORAD id, name, designator, operator, and current `OrbitalElements` | `panoptes-gen`, to name and attribute the objects in a conjunction |
| `conjunctions.jsonl` | `Row::Conjunction` — **the keystone**; the raw collision-avoidance event | `panoptes-gen`, as the ground truth of a `ca_geo` scenario |
| `launches.jsonl` | `Row::Launch` — id, name, date, provider, payloads, and a prose `description` | `panoptes-gen` (structured facts) and the embed step (the prose field) |
| `embeddings.jsonl` | `EmbeddedRow` — `{ id, text, vector }` for each embedded prose field | the retriever, for top-k cosine over the prose corpus |
| `runs.jsonl` | `RunRecord` — the append-only ledger of what ran, when, at what watermark | the ETL itself, for incremental runs and `status`; the graduation scheduler later |

Note what is *not* a top-level file: `OrbitalElements` never stands alone. It lives inside each `SpaceObject` as the `elements` field, and the derived quantities — period, apogee, perigee, orbit class — are computed on demand rather than stored, so the file stays canonical. When `panoptes-gen` needs to know that an object is in GEO, it calls `orbit_class()`; the file carries the seven Keplerian numbers, and the classification is a function of them.

## The keystone: a Conjunction *is* a `ca_geo` scenario

Everything in this course orbited one entity. Here is the payoff for making it the keystone.

Panoptes' one live scenario family, `ca_geo`, is collision avoidance in geostationary orbit: an operator deciding whether to maneuver, under time pressure, with uncertain attribution of who caused the conjunction. The family has four parameters — `attribution_confidence`, `time_pressure` (Hours/Days), `reversibility`, and `info_request`. A `Conjunction` row is the factual spine those parameters dress:

```rust,ignore
pub struct Conjunction {
    pub id: String,               // content-addressed at load time
    pub tca: DateTime<Utc>,       // time of closest approach
    pub miss_distance_km: f64,
    pub collision_probability: f64,
    pub primary: NoradId,
    pub secondary: NoradId,
}
```

Read that against the scenario grid and the mapping is almost mechanical:

- **`tca`** is where `time_pressure` comes from. A TCA a few hours out is the `Hours` cell; days out is the `Days` cell. The pressure is not invented — it is the clock on a real event.
- **`primary` / `secondary`** are the join keys back into `space_objects.jsonl`. Resolve the secondary and its `operator` field, and you have the raw material for `attribution_confidence`: a named commercial operator reads differently from an unattributed debris fragment. This is precisely why the other entities exist — *to enrich and attribute conjunctions.*
- **`miss_distance_km`** and **`collision_probability`** are the stakes. They are what makes the decision hard rather than obvious, and they are measured, not stipulated.

What the ETL does *not* supply is `reversibility` and `info_request`. Those are decision framings Panoptes overlays on the factual event — knobs on the vignette, not facts about the sky. That division is exactly right: the ETL owns *what is true*, and Panoptes owns *how the question is posed*. A real Conjunction Data Message is a collision-avoidance decision waiting to be asked; the ETL's job was to deliver it real, and the keystone is where "real" lives.

<div class="callout">
<span class="callout-label">Why grounding matters here specifically</span>
A scenario a committee can challenge — "is this a real conjunction, or did you make it up?" — is only defensible if the answer is a NORAD id and a TCA it can look up. The file contract turns that challenge into a lookup. That is the difference between an eval that measures decision-making and one that measures a model's reaction to plausible-sounding fiction.
</div>

## Retrieval feeds Panoptes two ways

The RAG arc produced `embeddings.jsonl` and a brute-force retriever. Only *prose* was ever embedded — `Row::prose()` returns text for a `Launch` description and `None` for everything numeric, because you never semantic-search the numbers. That retriever hands Panoptes two distinct capabilities, and the split matters:

1. **Grounding a vignette's prose.** When `panoptes-gen` writes the narrative around a conjunction — the mission context, the operator's history, the launch that put the object up there — it can retrieve the most relevant real descriptions and weave them in. The vignette reads as factual because it *is* built from facts pulled by cosine similarity over real launch prose.

2. **As a retrieval eval treatment arm.** Retrieval can instead be the *thing under test*: one arm of the experiment gives the model retrieved context, another withholds it, and Panoptes measures whether grounding changes the decision. Here retrieval is not a writing aid — it is an independent variable.

Both consume the same `embeddings.jsonl` through the same `top_k` interface. The choice between "grounding" and "treatment arm" is **Panoptes' to make, not the ETL's** — augment-and-generate was always out of scope here. The ETL's contract ends at the vector file. What Panoptes does with a retrieved fact is a research-design decision, and keeping that decision on Panoptes' side of the file boundary is the same discipline as everything else in this course: produce the artifact, expose it cleanly, and let the consumer decide what it means.

## What you actually shipped

The harness reads real satellites, real elements, and real conjunctions from four files, and it can regenerate scenarios from scratch whenever the sky changes. That is the deliverable. The seams — the `Source` trait, the E/T/L split, the ledger, content-addressing — were the discipline that let you build it without painting yourself into a corner. The next chapter is about the corners you deliberately left open.
