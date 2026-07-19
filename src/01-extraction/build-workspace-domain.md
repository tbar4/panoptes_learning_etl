# Build: Workspace + Domain Model + the `Source` Trait

> **Maps to:** Phase 0 / Task 1.1 (the `etl-core` seams). **Kind:** Build — spec and test names only. You write the implementation.

## Objective

Turn the project into a Cargo **workspace** and build the `etl-core` crate that every other crate will depend on: the domain model (`OrbitalElements`, `SpaceObject`, and the `Conjunction` keystone), and the three pipeline traits (`Source`, `Transform`, `Sink`) plus their boundary types (`RawRecord`, `Row`). This is the crate the dependency arrow always points *toward* — it names nothing else in the workspace.

You already sketched `EtlError` and `NoradId` in Part I; this chapter is where they get a home, alongside the domain and the traits.

## Scaffold

**Convert to a workspace.** Replace the top-level `Cargo.toml` with a *virtual* manifest (no `[package]`, just `[workspace]`). It lists members and — the piece that pays off across six crates — a `[workspace.dependencies]` table every crate inherits from, so a version is pinned in exactly one place:

```toml
# Cargo.toml (workspace root)
[workspace]
resolver = "2"
members = ["crates/etl-core"]   # more crates join as later arcs add them

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
chrono = { version = "0.4", features = ["serde"] }
thiserror = "2"
async-trait = "0.1"
tokio = { version = "1", features = ["full"] }
sha2 = "0.10"
# dev
pretty_assertions = "1"
```

**Create the crate:** `crates/etl-core/Cargo.toml`, `crates/etl-core/src/lib.rs`, and the modules `error.rs`, `ids.rs`, `domain.rs`, `pipeline.rs`.

**`crates/etl-core/Cargo.toml`** — inherit each dependency with `{ workspace = true }`, and know *why* each is here:

| Dependency | Why `etl-core` needs it |
| --- | --- |
| `serde` (+ derive) | every domain type and `Row` derives `Serialize`/`Deserialize` — this is the file contract |
| `serde_json` | the JSONL encoding of `Row` |
| `chrono` (+ serde) | `DateTime<Utc>` fields (`epoch`, `tca`, `net`) that must (de)serialize |
| `thiserror` | derives `EtlError` (Part I) |
| `async-trait` | makes the `async` methods on `Source`/`Sink` dyn-compatible |
| `tokio` | the runtime the async traits and their tests run on |
| `sha2` | content-addressing, used from Part IV — declare it now so the crate is stable |
| `pretty_assertions` (dev) | readable `assert_eq!` diffs in tests |

Add `tokio = { workspace = true, features = ["test-util"] }` under `[dev-dependencies]` so async tests can use paused time later.

**Expected result:** `cargo test -p etl-core` → all domain and pipeline tests green.

## The spec (givens)

### Domain: `OrbitClass` (in `domain.rs`)

A coarse regime enum. **Derives:** `Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize`. **serde:** `#[serde(rename_all = "SCREAMING_SNAKE_CASE")]`. **Variants:** `Leo`, `Meo`, `Geo`, `Heo` (which serialize as `"LEO"`, `"MEO"`, `"GEO"`, `"HEO"`).

### Domain: `OrbitalElements` (in `domain.rs`)

The Keplerian mean elements a TLE carries. **Derives:** `Debug, Clone, Copy, PartialEq, Serialize, Deserialize`. **Fields** (all `f64` except the first):

```text
epoch: DateTime<Utc>
inclination_deg, raan_deg, eccentricity,
arg_perigee_deg, mean_anomaly_deg, mean_motion_rev_per_day
```

Derived quantities are **methods, not stored fields**, so the stored form stays canonical. Constants and formulas (given — this is physics, not the lesson):

```text
EARTH_MU_KM3_S2 = 398_600.4418   // Earth's gravitational parameter, km^3/s^2
EARTH_RADIUS_KM = 6378.137       // equatorial radius

period_minutes()      = 1440.0 / mean_motion_rev_per_day
semi_major_axis_km()  = cbrt(MU / n^2), where n = mean_motion * 2π / 86_400  (rad/s)
apogee_km()           = a * (1 + ecc) - R_earth
perigee_km()          = a * (1 - ecc) - R_earth
```

`orbit_class()` classifies **in this order** (the order matters — eccentricity is checked first):

```text
ecc > 0.25                      -> HEO
else period_minutes < 128       -> LEO
else period_minutes in [1430, 1450]  -> GEO
else                            -> MEO
```

### Domain: `SpaceObject`, `Conjunction`, `Launch` (in `domain.rs`)

All three derive `Debug, Clone, PartialEq, Serialize, Deserialize`.

```text
SpaceObject { norad_id: NoradId, name: String, intl_designator: String,
              operator: Option<String>, elements: OrbitalElements }
   // intl_designator is the raw TLE field, e.g. "98067A" (not hyphenated)
   // convenience: orbit_class(&self) -> OrbitClass delegates to elements

Conjunction { id: String, tca: DateTime<Utc>, miss_distance_km: f64,
              collision_probability: f64, primary: NoradId, secondary: NoradId }
   // the keystone entity; `id` is content-addressed at load time (Part IV)

Launch { id: String, name: String, net: DateTime<Utc>, provider: String,
         payloads: Vec<String>, description: String }
   // `description` is prose — the field the RAG arc later embeds
```

### The three traits + boundary types (in `pipeline.rs`)

Copy these signatures exactly; they are the contract the rest of the course builds against.

```text
RawRecord { source: String, payload: String, fetched_at: DateTime<Utc> }
   derives: Debug, Clone, PartialEq

Row  (derives: Debug, Clone, PartialEq, Serialize, Deserialize)
   #[serde(tag = "kind", rename_all = "snake_case")]
   variants: SpaceObject(SpaceObject), Conjunction(Conjunction), Launch(Launch)

#[async_trait]
trait Source: Send + Sync {
    fn name(&self) -> &str;
    async fn extract(&self) -> Result<Vec<RawRecord>, EtlError>;
}

trait Transform: Send + Sync {
    fn transform(&self, raw: &RawRecord) -> Result<Vec<Row>, EtlError>;
}

#[async_trait]
trait Sink: Send + Sync {
    async fn load(&self, rows: &[Row]) -> Result<usize, EtlError>;
}
```

**`lib.rs`** declares the modules and re-exports the public names (`EtlError`, `NoradId`, the domain types, `RawRecord`, `Row`, `Source`, `Transform`, `Sink`) so downstream crates write `use etl_core::Source;`, not `use etl_core::pipeline::Source;`.

## Test names to hit

In `domain.rs`:

- `iss_is_low_earth_orbit` — ISS-like elements (mean motion ≈ 15.5): assert `period_minutes()` ≈ 92.90, `orbit_class()` == `Leo`, and `perigee_km()` in `300.0..500.0`.
- `geostationary_is_classified_geo` — GEO-like elements (mean motion ≈ 1.0027): assert `period_minutes()` ≈ 1436.1 and `orbit_class()` == `Geo`.
- `orbit_class_serializes_screaming` — `serde_json::to_string(&OrbitClass::Geo)` == `"\"GEO\""`.
- `space_object_roundtrips_through_json` — serialize a `SpaceObject`, deserialize it, assert equal.

In `pipeline.rs`:

- `row_tags_by_kind` — serialize a `Row::Launch`, assert the JSON contains `"kind":"launch"`, then round-trip it back to an equal `Row`.
- `a_source_can_be_boxed_as_dyn` — define a `Stub` impl of `Source` inside the test, bind it as `Box<dyn Source>`, `.extract().await` it, and assert on the result. **This test is the point:** it proves the trait is dyn-compatible, which is the property the orchestrator depends on four arcs from now.

## The build loop (you drive)

1. **Workspace first.** Convert the root `Cargo.toml`, create `crates/etl-core/`, and get an empty `cargo build -p etl-core` to succeed before writing any logic.
2. **`OrbitClass` and `OrbitalElements`.** Write `orbit_class_serializes_screaming` and `iss_is_low_earth_orbit` *first*. **Predict** the period for mean motion 15.5 before you run — `1440 / 15.5` — so a green test confirms your formula, not your luck.
3. **Predict the classifier ordering.** Before implementing `orbit_class()`, ask: an object with `ecc = 0.3` and a 100-minute period — LEO or HEO? The branch order in the spec answers it. Write the code to match your prediction, then check.
4. `SpaceObject` + the roundtrip test.
5. **The traits.** Add `RawRecord`, `Row`, and the three trait definitions. Write `row_tags_by_kind` and `a_source_can_be_boxed_as_dyn`. If `a_source_can_be_boxed_as_dyn` will not compile, re-read the object-safety section of the concept page — the fix is in the trait, not the test.
6. **Run green, commit.**

<div class="callout">
<span class="callout-label">Predict before you run</span>
For every test above, say out loud what value or JSON string you expect before pressing enter. The classifier thresholds and the SCREAMING_SNAKE_CASE rename are exactly the kind of thing you <em>think</em> you know until the runner disagrees. A wrong prediction here costs you thirty seconds; the same misunderstanding in the CelesTrak transform costs you an afternoon.
</div>

## Done when

`cargo test -p etl-core` is green across all six tests above (plus your Part I `error`/`ids` tests). You now have the crate every other arc imports — the domain, the seams, and the proof (`a_source_can_be_boxed_as_dyn`) that those seams can be scheduled. Next you make `Source` and `Transform` real against CelesTrak.
