# Appendix: The Answer Key (Full Task Plan)

This is the complete, verified reference implementation — the code you check your
own work against. It is the `panoptes_etl` workspace exactly as it ships: it
compiles, passes **65 tests**, and is clippy-clean.

Read it as a reference, not a script. Every file is reproduced in full, including
its `#[cfg(test)]` module, because the tests *are* part of the answer — they pin
the behavior each build chapter asks you to reach. The listings here are
reference-only (they are not compiled by the book build), but they are the real,
shipped source.

The sections follow the course arcs in dependency order:
**Foundations → Extraction → Authenticated → Paginated → Orchestration → Retrieval.**
Each file has a stable anchor (`id="task-…"`) so a build chapter can deep-link
straight to its answer.

> The dependency arrow always points *toward* `etl-core`. `etl-orchestrate`
> depends on `etl-core` only and schedules `dyn Source`; only the `panoptes-etl`
> binary names concrete sources. That constraint is enforced by the crate
> boundaries below, not by convention.

---

## Part I — Foundations

The workspace skeleton and the two Part I prerequisites — the error taxonomy and
the `NoradId` newtype — that every later file leans on.

<h3 id="task-workspace-cargo">Cargo.toml — Workspace manifest (Part I)</h3>

The virtual workspace: four member crates and one shared dependency table so
versions never drift between crates.

```toml
[workspace]
resolver = "2"
members = ["crates/etl-core", "crates/etl-sources", "crates/etl-orchestrate", "crates/panoptes-etl"]

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
chrono = { version = "0.4", features = ["serde"] }
thiserror = "2"
anyhow = "1"
async-trait = "0.1"
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
clap = { version = "4", features = ["derive"] }
sha2 = "0.10"
itertools = "0.13"
# dev
wiremock = "0.6"
pretty_assertions = "1"
```

<h3 id="task-etl-core-cargo">crates/etl-core/Cargo.toml — the core crate manifest (Part I)</h3>

`etl-core` depends on nothing else in the workspace. It pulls the `test-util`
feature of tokio only as a dev-dependency, for the paused-time tests.

```toml
[package]
name = "etl-core"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { workspace = true }
serde_json = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
async-trait = { workspace = true }
sha2 = { workspace = true }
tokio = { workspace = true }

[dev-dependencies]
pretty_assertions = { workspace = true }
tokio = { workspace = true, features = ["test-util"] }
```

<h3 id="task-etl-core-lib">crates/etl-core/src/lib.rs — the core crate root (Part I)</h3>

The module list and the flat re-export surface every other crate imports from.
It names modules built across every later arc (`ledger`, `retry`, `content`,
`embed`, `vector`) — this is the *final* root; you grow it arc by arc.

```rust,ignore
//! `etl-core` — the seams every other crate depends on.
//!
//! Domain model, the `Source` / `Transform` / `Sink` traits, the error
//! taxonomy, and id newtypes. This crate depends on nothing else in the
//! workspace; the dependency arrow always points *toward* it.

pub mod content;
pub mod domain;
pub mod embed;
pub mod error;
pub mod ids;
pub mod ledger;
pub mod pipeline;
pub mod retry;
pub mod sink_jsonl;
pub mod vector;

pub use content::{content_id, IdempotentSink};
pub use domain::{Conjunction, Launch, OrbitClass, OrbitalElements, SpaceObject};
pub use embed::Embedder;
pub use error::EtlError;
pub use ids::NoradId;
pub use ledger::{Ledger, Outcome, RunRecord};
pub use pipeline::{RawRecord, Row, Sink, Source, Transform};
pub use retry::{with_retry, RetryPolicy};
pub use sink_jsonl::JsonlSink;
pub use vector::{cosine, EmbedSink, EmbeddedRow, VectorIndex};
```

<h3 id="task-etl-core-error">crates/etl-core/src/error.rs — the error taxonomy (Part I)</h3>

The single `EtlError` every stage returns. The load-bearing distinction is
transient vs permanent: `is_transient()` decides what the retry layer will ever
retry, and `retry_after()` surfaces a server-supplied backoff.

```rust,ignore
use std::time::Duration;

use thiserror::Error;

/// The single error type every ETL stage returns.
///
/// The load-bearing distinction is **transient vs permanent**: a transient
/// error (a dropped connection, a rate-limit response) is worth retrying with
/// backoff; a permanent one (a malformed payload, bad credentials) will fail
/// identically every time and must surface immediately.
#[derive(Debug, Error)]
pub enum EtlError {
    /// A network-level failure — connection reset, timeout, 5xx. Transient.
    #[error("network error: {0}")]
    Network(String),

    /// The server asked us to slow down. Transient, with a hint. `429`.
    #[error("rate limited; retry after {retry_after_secs}s")]
    RateLimited { retry_after_secs: u64 },

    /// Authentication failed — bad or missing credentials. Permanent. `401`/`403`.
    #[error("auth error: {0}")]
    Auth(String),

    /// A non-retryable client error — a `404`, a `400`, a renamed endpoint.
    /// Permanent: it will fail identically on every retry.
    #[error("permanent error: {0}")]
    Permanent(String),

    /// A payload could not be parsed into the domain model. Permanent.
    #[error("parse error: {0}")]
    Parse(String),

    /// A local I/O failure while reading or writing a sink.
    #[error("io error: {0}")]
    Io(#[from] std::io::Error),
}

impl EtlError {
    /// Whether retrying this error (after backoff) could plausibly succeed.
    pub fn is_transient(&self) -> bool {
        matches!(self, EtlError::Network(_) | EtlError::RateLimited { .. })
    }

    /// The server-requested backoff, when the error carries one.
    pub fn retry_after(&self) -> Option<Duration> {
        match self {
            EtlError::RateLimited { retry_after_secs } => {
                Some(Duration::from_secs(*retry_after_secs))
            }
            _ => None,
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use pretty_assertions::assert_eq;

    #[test]
    fn network_and_rate_limit_are_transient() {
        assert!(EtlError::Network("reset".into()).is_transient());
        assert!(EtlError::RateLimited { retry_after_secs: 30 }.is_transient());
    }

    #[test]
    fn parse_auth_and_permanent_are_not_transient() {
        assert!(!EtlError::Parse("bad line".into()).is_transient());
        assert!(!EtlError::Auth("401".into()).is_transient());
        assert!(!EtlError::Permanent("404".into()).is_transient());
    }

    #[test]
    fn retry_after_only_on_rate_limit() {
        assert_eq!(
            EtlError::RateLimited { retry_after_secs: 12 }.retry_after(),
            Some(Duration::from_secs(12))
        );
        assert_eq!(EtlError::Network("x".into()).retry_after(), None);
    }
}
```

Tests: `network_and_rate_limit_are_transient`, `parse_auth_and_permanent_are_not_transient`, `retry_after_only_on_rate_limit`.

<h3 id="task-etl-core-ids">crates/etl-core/src/ids.rs — the NoradId newtype (Part I)</h3>

A `u32` NORAD catalog number wrapped so it can't be confused with any other
integer id. `#[serde(transparent)]` keeps the wire form a bare integer; `Display`
zero-pads to five digits; `FromStr` returns an `EtlError::Parse`.

```rust,ignore
use std::fmt;
use std::str::FromStr;

use serde::{Deserialize, Serialize};

use crate::error::EtlError;

/// A NORAD catalog number — the canonical id for a tracked space object.
///
/// A newtype rather than a bare `u32` so it cannot be confused with any other
/// numeric id (a launch sequence, a payload count). The `#[serde(transparent)]`
/// makes it serialize as the plain integer, so the wire format stays clean.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, Serialize, Deserialize)]
#[serde(transparent)]
pub struct NoradId(pub u32);

impl fmt::Display for NoradId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // NORAD ids are conventionally shown zero-padded to five digits.
        write!(f, "{:05}", self.0)
    }
}

impl FromStr for NoradId {
    type Err = EtlError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        s.trim()
            .parse::<u32>()
            .map(NoradId)
            .map_err(|_| EtlError::Parse(format!("invalid NORAD id: {s:?}")))
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use pretty_assertions::assert_eq;

    #[test]
    fn display_is_zero_padded_to_five() {
        assert_eq!(NoradId(25544).to_string(), "25544");
        assert_eq!(NoradId(733).to_string(), "00733");
    }

    #[test]
    fn parses_from_trimmed_digits() {
        assert_eq!("25544".parse::<NoradId>().unwrap(), NoradId(25544));
        assert_eq!(" 00733 ".parse::<NoradId>().unwrap(), NoradId(733));
    }

    #[test]
    fn rejects_non_numeric() {
        assert!("ISS".parse::<NoradId>().is_err());
    }

    #[test]
    fn serializes_as_bare_integer() {
        assert_eq!(serde_json::to_string(&NoradId(25544)).unwrap(), "25544");
        assert_eq!(
            serde_json::from_str::<NoradId>("25544").unwrap(),
            NoradId(25544)
        );
    }
}
```

Tests: `display_is_zero_padded_to_five`, `parses_from_trimmed_digits`, `rejects_non_numeric`, `serializes_as_bare_integer`.

---

## Part II — The Extraction Arc (CelesTrak)

The domain model, the three E/T/L traits, the JSONL sink, and the first real
source — CelesTrak — with its TLE parser and HTTP-status classifier.

<h3 id="task-etl-core-domain">crates/etl-core/src/domain.rs — the domain model (Part II)</h3>

`OrbitalElements`, the derived orbit-class classifier, and the three stored
entities (`SpaceObject`, `Conjunction`, `Launch`). Derived quantities (period,
apogee, class) are computed on demand so the stored form stays canonical.

```rust,ignore
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

use crate::ids::NoradId;

/// Standard gravitational parameter of Earth, km^3 / s^2.
const EARTH_MU_KM3_S2: f64 = 398600.4418;
/// Equatorial radius of Earth, km.
const EARTH_RADIUS_KM: f64 = 6378.137;

/// A coarse orbit-regime classification, derived from the elements.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "SCREAMING_SNAKE_CASE")]
pub enum OrbitClass {
    /// Low Earth orbit.
    Leo,
    /// Medium Earth orbit.
    Meo,
    /// Geostationary / geosynchronous.
    Geo,
    /// Highly elliptical orbit.
    Heo,
}

/// The Keplerian mean elements of an orbit at a given epoch.
///
/// These are the numbers a TLE carries. The derived quantities (period,
/// apogee, perigee, class) are computed on demand rather than stored, so the
/// stored form stays canonical.
#[derive(Debug, Clone, Copy, PartialEq, Serialize, Deserialize)]
pub struct OrbitalElements {
    pub epoch: DateTime<Utc>,
    pub inclination_deg: f64,
    pub raan_deg: f64,
    pub eccentricity: f64,
    pub arg_perigee_deg: f64,
    pub mean_anomaly_deg: f64,
    pub mean_motion_rev_per_day: f64,
}

impl OrbitalElements {
    /// Orbital period in minutes, straight from mean motion.
    pub fn period_minutes(&self) -> f64 {
        1440.0 / self.mean_motion_rev_per_day
    }

    /// Semi-major axis in km, from mean motion via Kepler's third law.
    pub fn semi_major_axis_km(&self) -> f64 {
        let n_rad_s = self.mean_motion_rev_per_day * 2.0 * std::f64::consts::PI / 86_400.0;
        (EARTH_MU_KM3_S2 / (n_rad_s * n_rad_s)).cbrt()
    }

    /// Apogee altitude above the equator, km.
    pub fn apogee_km(&self) -> f64 {
        self.semi_major_axis_km() * (1.0 + self.eccentricity) - EARTH_RADIUS_KM
    }

    /// Perigee altitude above the equator, km.
    pub fn perigee_km(&self) -> f64 {
        self.semi_major_axis_km() * (1.0 - self.eccentricity) - EARTH_RADIUS_KM
    }

    /// A coarse regime classification from period and eccentricity.
    pub fn orbit_class(&self) -> OrbitClass {
        let period = self.period_minutes();
        if self.eccentricity > 0.25 {
            OrbitClass::Heo
        } else if period < 128.0 {
            OrbitClass::Leo
        } else if (1430.0..=1450.0).contains(&period) {
            OrbitClass::Geo
        } else {
            OrbitClass::Meo
        }
    }
}

/// A tracked space object with its most recent elements.
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct SpaceObject {
    pub norad_id: NoradId,
    pub name: String,
    /// The international designator as it appears in the TLE (e.g. `"98067A"`),
    /// i.e. the raw 2-digit-year field — not the hyphenated COSPAR form.
    pub intl_designator: String,
    pub operator: Option<String>,
    pub elements: OrbitalElements,
}

impl SpaceObject {
    pub fn orbit_class(&self) -> OrbitClass {
        self.elements.orbit_class()
    }
}

/// A conjunction between two objects — the keystone entity, a raw
/// collision-avoidance decision. `id` is content-addressed at load time.
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct Conjunction {
    pub id: String,
    pub tca: DateTime<Utc>,
    pub miss_distance_km: f64,
    pub collision_probability: f64,
    pub primary: NoradId,
    pub secondary: NoradId,
}

/// A launch, sourced from TheSpaceDevs. `description` is a prose field —
/// the kind of text the RAG arc later embeds.
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct Launch {
    pub id: String,
    pub name: String,
    pub net: DateTime<Utc>,
    pub provider: String,
    pub payloads: Vec<String>,
    pub description: String,
}

#[cfg(test)]
mod tests {
    use super::*;
    use pretty_assertions::assert_eq;

    fn iss_elements() -> OrbitalElements {
        OrbitalElements {
            epoch: "2026-07-01T00:00:00Z".parse().unwrap(),
            inclination_deg: 51.64,
            raan_deg: 250.0,
            eccentricity: 0.0007,
            arg_perigee_deg: 130.0,
            mean_anomaly_deg: 230.0,
            mean_motion_rev_per_day: 15.5,
        }
    }

    fn geo_elements() -> OrbitalElements {
        OrbitalElements {
            epoch: "2026-07-01T00:00:00Z".parse().unwrap(),
            inclination_deg: 0.05,
            raan_deg: 95.0,
            eccentricity: 0.0002,
            arg_perigee_deg: 270.0,
            mean_anomaly_deg: 90.0,
            mean_motion_rev_per_day: 1.0027,
        }
    }

    #[test]
    fn iss_is_low_earth_orbit() {
        let e = iss_elements();
        // 1440 / 15.5 = 92.9 min.
        assert!((e.period_minutes() - 92.90).abs() < 0.01);
        assert_eq!(e.orbit_class(), OrbitClass::Leo);
        // ISS perigee sits a few hundred km up.
        assert!((300.0..500.0).contains(&e.perigee_km()));
    }

    #[test]
    fn geostationary_is_classified_geo() {
        let e = geo_elements();
        assert!((e.period_minutes() - 1436.1).abs() < 0.5);
        assert_eq!(e.orbit_class(), OrbitClass::Geo);
    }

    #[test]
    fn orbit_class_serializes_screaming() {
        assert_eq!(
            serde_json::to_string(&OrbitClass::Geo).unwrap(),
            "\"GEO\""
        );
    }

    #[test]
    fn space_object_roundtrips_through_json() {
        let obj = SpaceObject {
            norad_id: NoradId(25544),
            name: "ISS (ZARYA)".into(),
            intl_designator: "98067A".into(),
            operator: Some("NASA".into()),
            elements: iss_elements(),
        };
        let json = serde_json::to_string(&obj).unwrap();
        let back: SpaceObject = serde_json::from_str(&json).unwrap();
        assert_eq!(back, obj);
    }
}
```

Tests: `iss_is_low_earth_orbit`, `geostationary_is_classified_geo`, `orbit_class_serializes_screaming`, `space_object_roundtrips_through_json`.

<h3 id="task-etl-core-pipeline">crates/etl-core/src/pipeline.rs — the E/T/L traits (Part II)</h3>

`RawRecord` (the output of E), the tagged `Row` enum (the T→L boundary), and the
three object-safe traits `Source` / `Transform` / `Sink`. `Source` and `Sink`
are async via `async_trait`; `Transform` is pure and synchronous.

```rust,ignore
use async_trait::async_trait;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

use crate::domain::{Conjunction, Launch, SpaceObject};
use crate::error::EtlError;

/// Raw bytes fetched from a source, before any parsing — the output of E.
#[derive(Debug, Clone, PartialEq)]
pub struct RawRecord {
    pub source: String,
    pub payload: String,
    pub fetched_at: DateTime<Utc>,
}

/// A normalized domain row — the typed boundary between T and L.
///
/// A single enum over every entity a source can emit lets one `Sink` accept
/// the output of any pipeline without knowing which source produced it.
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum Row {
    SpaceObject(SpaceObject),
    Conjunction(Conjunction),
    Launch(Launch),
}

/// **E** — extract raw records from an external source.
///
/// `Send + Sync` and object-safe (via `async_trait`) so the orchestrator can
/// schedule a `dyn Source` without knowing the concrete type.
#[async_trait]
pub trait Source: Send + Sync {
    fn name(&self) -> &str;
    async fn extract(&self) -> Result<Vec<RawRecord>, EtlError>;
}

/// **T** — parse raw records into normalized rows. Pure and synchronous.
pub trait Transform: Send + Sync {
    fn transform(&self, raw: &RawRecord) -> Result<Vec<Row>, EtlError>;
}

/// **L** — persist normalized rows to a sink; returns the count written.
#[async_trait]
pub trait Sink: Send + Sync {
    async fn load(&self, rows: &[Row]) -> Result<usize, EtlError>;
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::domain::{Launch, OrbitalElements, SpaceObject};
    use crate::ids::NoradId;
    use pretty_assertions::assert_eq;

    #[test]
    fn row_tags_by_kind() {
        let launch = Row::Launch(Launch {
            id: "ll2-42".into(),
            name: "Starlink G-99".into(),
            net: "2026-07-10T12:00:00Z".parse().unwrap(),
            provider: "SpaceX".into(),
            payloads: vec!["Starlink".into()],
            description: "A batch of broadband satellites.".into(),
        });
        let json = serde_json::to_string(&launch).unwrap();
        assert!(json.contains("\"kind\":\"launch\""));
        let back: Row = serde_json::from_str(&json).unwrap();
        assert_eq!(back, launch);
    }

    #[tokio::test]
    async fn a_source_can_be_boxed_as_dyn() {
        struct Stub;
        #[async_trait]
        impl Source for Stub {
            fn name(&self) -> &str {
                "stub"
            }
            async fn extract(&self) -> Result<Vec<RawRecord>, EtlError> {
                Ok(vec![RawRecord {
                    source: "stub".into(),
                    payload: "hi".into(),
                    fetched_at: "2026-07-01T00:00:00Z".parse().unwrap(),
                }])
            }
        }
        let source: Box<dyn Source> = Box::new(Stub);
        let raw = source.extract().await.unwrap();
        assert_eq!(source.name(), "stub");
        assert_eq!(raw.len(), 1);
    }

    // Silences unused-import warnings for types exercised only in other tests.
    #[allow(dead_code)]
    fn _touch(_: SpaceObject, _: OrbitalElements, _: NoradId) {}
}
```

Tests: `row_tags_by_kind`, `a_source_can_be_boxed_as_dyn`.

<h3 id="task-etl-core-sink-jsonl">crates/etl-core/src/sink_jsonl.rs — the JSONL sink (Part II)</h3>

The **L** contract `panoptes-gen` reads: newline-delimited `Row` values,
appended (never truncated) so a run never clobbers prior output.

```rust,ignore
use std::path::PathBuf;

use async_trait::async_trait;
use tokio::fs::OpenOptions;
use tokio::io::AsyncWriteExt;

use crate::error::EtlError;
use crate::pipeline::{Row, Sink};

/// Appends normalized rows to a file as JSON Lines — one JSON object per line.
///
/// This is the load contract `panoptes-gen` reads: newline-delimited `Row`
/// values, appended so a run never truncates prior output.
pub struct JsonlSink {
    pub path: PathBuf,
}

impl JsonlSink {
    pub fn new(path: impl Into<PathBuf>) -> Self {
        Self { path: path.into() }
    }
}

#[async_trait]
impl Sink for JsonlSink {
    async fn load(&self, rows: &[Row]) -> Result<usize, EtlError> {
        let mut file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(&self.path)
            .await?;
        let mut buf = String::new();
        for row in rows {
            let line = serde_json::to_string(row)
                .map_err(|e| EtlError::Parse(e.to_string()))?;
            buf.push_str(&line);
            buf.push('\n');
        }
        file.write_all(buf.as_bytes()).await?;
        Ok(rows.len())
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::domain::Launch;
    use pretty_assertions::assert_eq;

    fn sample_rows() -> Vec<Row> {
        vec![
            Row::Launch(Launch {
                id: "a".into(),
                name: "One".into(),
                net: "2026-07-10T12:00:00Z".parse().unwrap(),
                provider: "SpaceX".into(),
                payloads: vec![],
                description: "first".into(),
            }),
            Row::Launch(Launch {
                id: "b".into(),
                name: "Two".into(),
                net: "2026-07-11T12:00:00Z".parse().unwrap(),
                provider: "RocketLab".into(),
                payloads: vec![],
                description: "second".into(),
            }),
        ]
    }

    #[tokio::test]
    async fn writes_one_row_per_line_and_roundtrips() {
        let dir = std::env::temp_dir().join(format!("etl-jsonl-{}", std::process::id()));
        tokio::fs::create_dir_all(&dir).await.unwrap();
        let path = dir.join("rows.jsonl");
        let _ = tokio::fs::remove_file(&path).await;

        let sink = JsonlSink::new(&path);
        let rows = sample_rows();
        let n = sink.load(&rows).await.unwrap();
        assert_eq!(n, 2);

        let text = tokio::fs::read_to_string(&path).await.unwrap();
        let lines: Vec<&str> = text.lines().collect();
        assert_eq!(lines.len(), 2);
        let back: Row = serde_json::from_str(lines[0]).unwrap();
        assert_eq!(back, rows[0]);
    }

    #[tokio::test]
    async fn append_accumulates_across_loads() {
        let dir = std::env::temp_dir().join(format!("etl-jsonl-app-{}", std::process::id()));
        tokio::fs::create_dir_all(&dir).await.unwrap();
        let path = dir.join("rows.jsonl");
        let _ = tokio::fs::remove_file(&path).await;

        let sink = JsonlSink::new(&path);
        sink.load(&sample_rows()).await.unwrap();
        sink.load(&sample_rows()).await.unwrap();

        let text = tokio::fs::read_to_string(&path).await.unwrap();
        assert_eq!(text.lines().count(), 4);
    }
}
```

Tests: `writes_one_row_per_line_and_roundtrips`, `append_accumulates_across_loads`.

<h3 id="task-etl-sources-cargo">crates/etl-sources/Cargo.toml — the sources crate manifest (Part II)</h3>

Concrete sources. Depends on `etl-core` and `reqwest`; mocks every live API with
`wiremock` as a dev-dependency.

```toml
[package]
name = "etl-sources"
version = "0.1.0"
edition = "2024"

[dependencies]
etl-core = { path = "../etl-core" }
serde = { workspace = true }
serde_json = { workspace = true }
chrono = { workspace = true }
async-trait = { workspace = true }
reqwest = { workspace = true }
tokio = { workspace = true }

[dev-dependencies]
pretty_assertions = { workspace = true }
tokio = { workspace = true, features = ["test-util"] }
wiremock = { workspace = true }
```

<h3 id="task-etl-sources-lib">crates/etl-sources/src/lib.rs — the sources crate root (Part II)</h3>

The module list and re-exports for every concrete source and its support
machinery. Like the core root, this is the final form; you grow it arc by arc.

```rust,ignore
//! `etl-sources` — concrete `Source`/`Transform` implementations and the
//! machinery each real API forces on us (TLE parsing, rate limiting, retry,
//! pagination). Depends only on `etl-core`.

pub mod celestrak;
pub mod http;
pub mod limiter;
pub mod spacedevs;
pub mod spacetrack;
pub mod tle;
pub mod voyage;

pub use celestrak::{CelestrakSource, CelestrakTransform};
pub use limiter::TokenBucket;
pub use spacedevs::{SpaceDevsSource, SpaceDevsTransform};
pub use spacetrack::{Credentials, SpaceTrackSource, SpaceTrackTransform};
pub use voyage::VoyageEmbedder;
```

<h3 id="task-etl-sources-http">crates/etl-sources/src/http.rs — the HTTP-status classifier (Part II)</h3>

The single place that maps a non-success HTTP status onto the `EtlError`
taxonomy — so the transient/permanent decision is made once, correctly, and
every source funnels through it.

```rust,ignore
//! Mapping HTTP responses onto the [`EtlError`] taxonomy.
//!
//! This is the single place that decides which statuses are worth retrying.
//! Every source funnels non-success responses through [`classify_status`] so
//! the transient/permanent distinction is made once, correctly.

use etl_core::EtlError;
use reqwest::header::HeaderMap;
use reqwest::StatusCode;

/// Parse a `Retry-After` header value, in seconds. Absent/unparseable ⇒ 1s.
///
/// Only the delta-seconds form is handled; the HTTP-date form
/// (`Retry-After: Wed, 21 Oct 2026 07:28:00 GMT`) falls back to 1s. Real
/// APIs here use seconds; parsing the date form is left as a hardening step.
pub fn retry_after_secs(headers: &HeaderMap) -> u64 {
    headers
        .get(reqwest::header::RETRY_AFTER)
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.trim().parse().ok())
        .unwrap_or(1)
}

/// Map a non-success status onto the error taxonomy.
///
/// - `429` ⇒ [`EtlError::RateLimited`] (transient, carries the backoff)
/// - `401` / `403` ⇒ [`EtlError::Auth`] (permanent)
/// - other `4xx` ⇒ [`EtlError::Permanent`] (permanent — retrying won't help)
/// - `5xx` and anything else ⇒ [`EtlError::Network`] (transient)
pub fn classify_status(status: StatusCode, headers: &HeaderMap) -> EtlError {
    match status.as_u16() {
        429 => EtlError::RateLimited {
            retry_after_secs: retry_after_secs(headers),
        },
        401 | 403 => EtlError::Auth(format!("http {status}")),
        s if (400..500).contains(&s) => EtlError::Permanent(format!("http {status}")),
        _ => EtlError::Network(format!("http {status}")),
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    fn headers_with_retry_after(v: &str) -> HeaderMap {
        let mut h = HeaderMap::new();
        h.insert(reqwest::header::RETRY_AFTER, v.parse().unwrap());
        h
    }

    #[test]
    fn maps_429_to_rate_limited_with_backoff() {
        let err = classify_status(StatusCode::TOO_MANY_REQUESTS, &headers_with_retry_after("30"));
        assert!(matches!(err, EtlError::RateLimited { retry_after_secs: 30 }));
        assert!(err.is_transient());
    }

    #[test]
    fn maps_401_403_to_permanent_auth() {
        assert!(matches!(
            classify_status(StatusCode::UNAUTHORIZED, &HeaderMap::new()),
            EtlError::Auth(_)
        ));
        assert!(!classify_status(StatusCode::FORBIDDEN, &HeaderMap::new()).is_transient());
    }

    #[test]
    fn maps_404_to_permanent_not_network() {
        let err = classify_status(StatusCode::NOT_FOUND, &HeaderMap::new());
        assert!(matches!(err, EtlError::Permanent(_)));
        assert!(!err.is_transient());
    }

    #[test]
    fn maps_5xx_to_transient_network() {
        let err = classify_status(StatusCode::INTERNAL_SERVER_ERROR, &HeaderMap::new());
        assert!(matches!(err, EtlError::Network(_)));
        assert!(err.is_transient());
    }
}
```

Tests: `maps_429_to_rate_limited_with_backoff`, `maps_401_403_to_permanent_auth`, `maps_404_to_permanent_not_network`, `maps_5xx_to_transient_network`.

<h3 id="task-tle-parser">crates/etl-sources/src/tle.rs — TLE parser (Part II)</h3>

A hand-rolled parser for the fixed-width two-line element format: mod-10
checksum (minus counts as 1), the 2-digit-year epoch pivot, byte-slice column
extraction, and the cross-line catalog-number check that catches misalignment a
checksum can't.

```rust,ignore
//! A hand-rolled parser for the two-line element (TLE) set format.
//!
//! TLE is a fixed-width legacy text format: fields live at exact column
//! positions, and each line ends in a mod-10 checksum digit. Parsing it by
//! column — and validating the checksum — is the extraction lesson.

use chrono::{DateTime, Duration, TimeZone, Utc};
use etl_core::{EtlError, OrbitalElements};

/// The mod-10 checksum of a TLE line: sum the digits in columns 1–68, count
/// each minus sign as 1 and everything else as 0, take the result mod 10.
pub fn tle_checksum(line: &str) -> u8 {
    let sum: u32 = line
        .chars()
        .take(68)
        .map(|c| match c {
            '0'..='9' => c.to_digit(10).unwrap(),
            '-' => 1,
            _ => 0,
        })
        .sum();
    (sum % 10) as u8
}

/// Decode a TLE epoch (2-digit year + fractional day-of-year) to UTC.
///
/// Years 57–99 are 1957–1999; 00–56 are 2000–2056 (the TLE convention).
pub fn decode_epoch(year2: &str, day_frac: &str) -> Result<DateTime<Utc>, EtlError> {
    let yy: i32 = year2
        .trim()
        .parse()
        .map_err(|_| EtlError::Parse(format!("bad epoch year: {year2:?}")))?;
    let year = if yy < 57 { 2000 + yy } else { 1900 + yy };
    let doy: f64 = day_frac
        .trim()
        .parse()
        .map_err(|_| EtlError::Parse(format!("bad epoch day: {day_frac:?}")))?;

    let start = Utc
        .with_ymd_and_hms(year, 1, 1, 0, 0, 0)
        .single()
        .ok_or_else(|| EtlError::Parse(format!("bad epoch year: {year}")))?;
    // Day-of-year is 1-based; the fractional part is a fraction of a day.
    let seconds = (doy - 1.0) * 86_400.0;
    Ok(start + Duration::milliseconds((seconds * 1000.0).round() as i64))
}

/// The NORAD catalog number carried in columns 3–7 of either line.
pub fn tle_norad(line1: &str) -> Result<u32, EtlError> {
    slice(line1, 2, 7)?
        .trim()
        .parse()
        .map_err(|_| EtlError::Parse(format!("bad NORAD id in {line1:?}")))
}

/// The international designator, columns 10–17 of line 1.
pub fn tle_intl_designator(line1: &str) -> Result<String, EtlError> {
    Ok(slice(line1, 9, 17)?.trim().to_string())
}

/// Parse a name + two element lines into [`OrbitalElements`], validating both
/// checksums first.
pub fn parse_tle(line1: &str, line2: &str) -> Result<OrbitalElements, EtlError> {
    verify_checksum(line1)?;
    verify_checksum(line2)?;

    // Both lines carry the catalog number; a mismatch means the 3-line block
    // was misaligned (an upstream truncation) — checksums can't catch that.
    if slice(line1, 2, 7)?.trim() != slice(line2, 2, 7)?.trim() {
        return Err(EtlError::Parse(format!(
            "TLE line mismatch: line 1 is {:?}, line 2 is {:?}",
            slice(line1, 2, 7)?,
            slice(line2, 2, 7)?
        )));
    }

    let epoch = decode_epoch(slice(line1, 18, 20)?, slice(line1, 20, 32)?)?;

    let inclination_deg = field(line2, 8, 16)?;
    let raan_deg = field(line2, 17, 25)?;
    let eccentricity = format!("0.{}", slice(line2, 26, 33)?.trim())
        .parse::<f64>()
        .map_err(|_| EtlError::Parse(format!("bad eccentricity in {line2:?}")))?;
    let arg_perigee_deg = field(line2, 34, 42)?;
    let mean_anomaly_deg = field(line2, 43, 51)?;
    let mean_motion_rev_per_day = field(line2, 52, 63)?;

    Ok(OrbitalElements {
        epoch,
        inclination_deg,
        raan_deg,
        eccentricity,
        arg_perigee_deg,
        mean_anomaly_deg,
        mean_motion_rev_per_day,
    })
}

fn verify_checksum(line: &str) -> Result<(), EtlError> {
    let expected = line
        .chars()
        .nth(68)
        .and_then(|c| c.to_digit(10))
        .ok_or_else(|| EtlError::Parse(format!("missing checksum digit in {line:?}")))?
        as u8;
    let got = tle_checksum(line);
    if got != expected {
        return Err(EtlError::Parse(format!(
            "checksum mismatch in {line:?}: computed {got}, line says {expected}"
        )));
    }
    Ok(())
}

fn slice(line: &str, start: usize, end: usize) -> Result<&str, EtlError> {
    line.get(start..end)
        .ok_or_else(|| EtlError::Parse(format!("line too short for cols {start}..{end}: {line:?}")))
}

fn field(line: &str, start: usize, end: usize) -> Result<f64, EtlError> {
    slice(line, start, end)?
        .trim()
        .parse()
        .map_err(|_| EtlError::Parse(format!("bad number at cols {start}..{end} in {line:?}")))
}

#[cfg(test)]
mod tests {
    use super::*;
    use chrono::{Datelike, Timelike};
    use etl_core::OrbitClass;
    use pretty_assertions::assert_eq;

    // The canonical ISS TLE (Wikipedia's worked example). Both checksums are 7.
    const L1: &str = "1 25544U 98067A   08264.51782528 -.00002182  00000-0 -11606-4 0  2927";
    const L2: &str = "2 25544  51.6416 247.4627 0006703 130.5360 325.0288 15.72125391563537";

    #[test]
    fn checksum_matches_known_lines() {
        assert_eq!(tle_checksum(L1), 7);
        assert_eq!(tle_checksum(L2), 7);
    }

    #[test]
    fn rejects_bad_checksum() {
        // Flip the last digit; verification must fail as a parse error.
        let bad = format!("{}9", &L2[..68]);
        let err = parse_tle(L1, &bad).unwrap_err();
        assert!(matches!(err, EtlError::Parse(_)));
    }

    #[test]
    fn decodes_2008_epoch() {
        let e = decode_epoch("08", "264.51782528").unwrap();
        assert_eq!(e.year(), 2008);
        assert_eq!(e.month(), 9);
        assert_eq!(e.day(), 20);
        assert_eq!(e.hour(), 12);
    }

    #[test]
    fn parses_iss_tle_to_elements() {
        let e = parse_tle(L1, L2).unwrap();
        assert!((e.inclination_deg - 51.6416).abs() < 1e-4);
        assert!((e.eccentricity - 0.0006703).abs() < 1e-9);
        assert!((e.mean_motion_rev_per_day - 15.72125391).abs() < 1e-6);
        assert_eq!(e.orbit_class(), OrbitClass::Leo);
    }

    #[test]
    fn reads_norad_and_designator() {
        assert_eq!(tle_norad(L1).unwrap(), 25544);
        assert_eq!(tle_intl_designator(L1).unwrap(), "98067A");
    }

    #[test]
    fn rejects_misaligned_line_pair() {
        // A line 2 for a *different* catalog number, with a corrected checksum,
        // so only the cross-line mismatch (not a checksum) can reject it.
        let other_body = &L2.replacen("25544", "12345", 1)[..68];
        let mismatched = format!("{other_body}{}", tle_checksum(other_body));
        let err = parse_tle(L1, &mismatched).unwrap_err();
        assert!(matches!(err, EtlError::Parse(_)));
    }
}
```

Tests: `checksum_matches_known_lines`, `rejects_bad_checksum`, `decodes_2008_epoch`, `parses_iss_tle_to_elements`, `reads_norad_and_designator`, `rejects_misaligned_line_pair`.

<h3 id="task-celestrak-source">crates/etl-sources/src/celestrak.rs — the CelesTrak source (Part II)</h3>

The first real E/T split: `CelestrakSource::extract` is an open HTTP GET
returning TLE text; `CelestrakTransform` chunks the text into 3-line blocks and
parses each into a `Row::SpaceObject`. Wiremock covers success, 500, 429, and 404.

```rust,ignore
//! The CelesTrak source: an open HTTP GET returning TLE text (Extract), plus
//! the transform that parses 3-line TLE blocks into `Row::SpaceObject`.

use async_trait::async_trait;
use chrono::Utc;
use etl_core::{EtlError, RawRecord, Row, SpaceObject, Source, Transform};

use crate::http::classify_status;
use crate::tle::{parse_tle, tle_intl_designator, tle_norad};
use etl_core::NoradId;

/// Fetches a named group of elements from CelesTrak in TLE format.
pub struct CelestrakSource {
    pub base_url: String,
    pub group: String,
}

impl CelestrakSource {
    pub fn new(base_url: impl Into<String>, group: impl Into<String>) -> Self {
        Self {
            base_url: base_url.into(),
            group: group.into(),
        }
    }
}

#[async_trait]
impl Source for CelestrakSource {
    fn name(&self) -> &str {
        "celestrak"
    }

    async fn extract(&self) -> Result<Vec<RawRecord>, EtlError> {
        let url = format!(
            "{}/NORAD/elements/gp.php?GROUP={}&FORMAT=tle",
            self.base_url.trim_end_matches('/'),
            self.group
        );
        let resp = reqwest::get(&url)
            .await
            .map_err(|e| EtlError::Network(e.to_string()))?;
        if !resp.status().is_success() {
            return Err(classify_status(resp.status(), resp.headers()));
        }
        let payload = resp
            .text()
            .await
            .map_err(|e| EtlError::Network(e.to_string()))?;
        Ok(vec![RawRecord {
            source: self.name().to_string(),
            payload,
            fetched_at: Utc::now(),
        }])
    }
}

/// Parses CelesTrak TLE text into normalized space objects.
pub struct CelestrakTransform;

impl Transform for CelestrakTransform {
    fn transform(&self, raw: &RawRecord) -> Result<Vec<Row>, EtlError> {
        let lines: Vec<&str> = raw
            .payload
            .lines()
            .map(|l| l.trim_end())
            .filter(|l| !l.is_empty())
            .collect();

        let mut rows = Vec::new();
        for block in lines.chunks(3) {
            if block.len() != 3 {
                return Err(EtlError::Parse(format!(
                    "trailing partial TLE block: {block:?}"
                )));
            }
            let (name, l1, l2) = (block[0], block[1], block[2]);
            let elements = parse_tle(l1, l2)?;
            rows.push(Row::SpaceObject(SpaceObject {
                norad_id: NoradId(tle_norad(l1)?),
                name: name.trim().to_string(),
                intl_designator: tle_intl_designator(l1)?,
                operator: None,
                elements,
            }));
        }
        Ok(rows)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use pretty_assertions::assert_eq;
    use wiremock::matchers::{method, path, query_param};
    use wiremock::{Mock, MockServer, ResponseTemplate};

    const BODY: &str = "ISS (ZARYA)\n\
        1 25544U 98067A   08264.51782528 -.00002182  00000-0 -11606-4 0  2927\n\
        2 25544  51.6416 247.4627 0006703 130.5360 325.0288 15.72125391563537\n";

    #[tokio::test]
    async fn extract_returns_raw_tle_text() {
        let server = MockServer::start().await;
        Mock::given(method("GET"))
            .and(path("/NORAD/elements/gp.php"))
            .and(query_param("GROUP", "geo"))
            .and(query_param("FORMAT", "tle"))
            .respond_with(ResponseTemplate::new(200).set_body_string(BODY))
            .mount(&server)
            .await;

        let src = CelestrakSource::new(server.uri(), "geo");
        let raw = src.extract().await.unwrap();
        assert_eq!(raw.len(), 1);
        assert_eq!(raw[0].source, "celestrak");
        assert!(raw[0].payload.contains("25544"));
    }

    #[tokio::test]
    async fn server_error_is_transient_network() {
        let server = MockServer::start().await;
        Mock::given(method("GET"))
            .respond_with(ResponseTemplate::new(500))
            .mount(&server)
            .await;
        let src = CelestrakSource::new(server.uri(), "geo");
        let err = src.extract().await.unwrap_err();
        assert!(matches!(err, EtlError::Network(_)));
        assert!(err.is_transient());
    }

    #[tokio::test]
    async fn rate_limit_is_retryable_with_backoff() {
        let server = MockServer::start().await;
        Mock::given(method("GET"))
            .respond_with(ResponseTemplate::new(429).insert_header("retry-after", "42"))
            .mount(&server)
            .await;
        let src = CelestrakSource::new(server.uri(), "geo");
        let err = src.extract().await.unwrap_err();
        assert!(matches!(err, EtlError::RateLimited { retry_after_secs: 42 }));
        assert!(err.is_transient());
    }

    #[tokio::test]
    async fn not_found_is_permanent_not_retryable() {
        let server = MockServer::start().await;
        Mock::given(method("GET"))
            .respond_with(ResponseTemplate::new(404))
            .mount(&server)
            .await;
        let src = CelestrakSource::new(server.uri(), "nonesuch");
        let err = src.extract().await.unwrap_err();
        assert!(matches!(err, EtlError::Permanent(_)));
        assert!(!err.is_transient());
    }

    #[test]
    fn transform_parses_tle_blocks_to_space_objects() {
        let raw = RawRecord {
            source: "celestrak".into(),
            payload: BODY.into(),
            fetched_at: Utc::now(),
        };
        let rows = CelestrakTransform.transform(&raw).unwrap();
        assert_eq!(rows.len(), 1);
        match &rows[0] {
            Row::SpaceObject(o) => {
                assert_eq!(o.norad_id, NoradId(25544));
                assert_eq!(o.name, "ISS (ZARYA)");
                assert_eq!(o.intl_designator, "98067A");
            }
            other => panic!("expected SpaceObject, got {other:?}"),
        }
    }
}
```

Tests: `extract_returns_raw_tle_text`, `server_error_is_transient_network`, `rate_limit_is_retryable_with_backoff`, `not_found_is_permanent_not_retryable`, `transform_parses_tle_blocks_to_space_objects`.

---

## Part III — The Authenticated Arc (Space-Track)

Cookie-session auth, a hand-rolled token bucket, and the run ledger — the
append-only artifact the orchestrator later reads for watermarks and retries.

<h3 id="task-etl-core-ledger">crates/etl-core/src/ledger.rs — the run ledger (Part III)</h3>

An append-only JSONL log of every run. `last_success` reads back the most recent
successful `RunRecord` for a source — the seed of incremental pulls, retries, and
backfill.

```rust,ignore
//! The run ledger — an append-only record of every pipeline run.
//!
//! This one artifact is the seed of everything an orchestrator adds later:
//! incremental pulls read the last watermark, retries read the last outcome,
//! backfill reads the gaps. It mirrors the first course's append-only response
//! log.

use std::path::PathBuf;

use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use tokio::fs::OpenOptions;
use tokio::io::AsyncWriteExt;

use crate::error::EtlError;

/// How a run ended.
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(tag = "outcome", rename_all = "snake_case")]
pub enum Outcome {
    Success,
    Failed { reason: String },
}

/// One line in the ledger: a single run of a single source.
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct RunRecord {
    pub source: String,
    pub started_at: DateTime<Utc>,
    pub finished_at: DateTime<Utc>,
    pub outcome: Outcome,
    /// The high-water mark reached. The taught incremental path skips a source
    /// when its last success is recent (see `Executor::is_fresh`, which reads
    /// `finished_at`); consuming this watermark to bound a query — "fetch only
    /// rows after it" — is the graduation step covered in Part VII.
    pub watermark: Option<DateTime<Utc>>,
    pub rows_written: usize,
}

/// Append-only JSONL ledger.
pub struct Ledger {
    pub path: PathBuf,
}

impl Ledger {
    pub fn new(path: impl Into<PathBuf>) -> Self {
        Self { path: path.into() }
    }

    /// Append one run record.
    pub async fn append(&self, rec: &RunRecord) -> Result<(), EtlError> {
        let mut file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(&self.path)
            .await?;
        let line = serde_json::to_string(rec).map_err(|e| EtlError::Parse(e.to_string()))?;
        file.write_all(line.as_bytes()).await?;
        file.write_all(b"\n").await?;
        Ok(())
    }

    /// The most recent *successful* run for a source, if any.
    pub async fn last_success(&self, source: &str) -> Result<Option<RunRecord>, EtlError> {
        let text = match tokio::fs::read_to_string(&self.path).await {
            Ok(t) => t,
            Err(e) if e.kind() == std::io::ErrorKind::NotFound => return Ok(None),
            Err(e) => return Err(e.into()),
        };
        let mut last = None;
        for line in text.lines().filter(|l| !l.trim().is_empty()) {
            let rec: RunRecord =
                serde_json::from_str(line).map_err(|e| EtlError::Parse(e.to_string()))?;
            if rec.source == source && rec.outcome == Outcome::Success {
                last = Some(rec);
            }
        }
        Ok(last)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use pretty_assertions::assert_eq;

    fn rec(source: &str, watermark: &str, outcome: Outcome) -> RunRecord {
        RunRecord {
            source: source.into(),
            started_at: "2026-07-19T00:00:00Z".parse().unwrap(),
            finished_at: "2026-07-19T00:01:00Z".parse().unwrap(),
            outcome,
            watermark: Some(watermark.parse().unwrap()),
            rows_written: 10,
        }
    }

    fn temp_ledger(tag: &str) -> Ledger {
        let dir = std::env::temp_dir().join(format!("etl-ledger-{tag}-{}", std::process::id()));
        std::fs::create_dir_all(&dir).unwrap();
        let path = dir.join("runs.jsonl");
        let _ = std::fs::remove_file(&path);
        Ledger::new(path)
    }

    #[tokio::test]
    async fn append_then_last_success_reads_back() {
        let ledger = temp_ledger("read");
        let r = rec("celestrak", "2026-07-19T00:00:00Z", Outcome::Success);
        ledger.append(&r).await.unwrap();
        assert_eq!(ledger.last_success("celestrak").await.unwrap(), Some(r));
        assert_eq!(ledger.last_success("spacetrack").await.unwrap(), None);
    }

    #[tokio::test]
    async fn last_success_ignores_failures() {
        let ledger = temp_ledger("fail");
        ledger
            .append(&rec("celestrak", "2026-07-18T00:00:00Z", Outcome::Success))
            .await
            .unwrap();
        ledger
            .append(&rec(
                "celestrak",
                "2026-07-19T00:00:00Z",
                Outcome::Failed { reason: "boom".into() },
            ))
            .await
            .unwrap();
        let last = ledger.last_success("celestrak").await.unwrap().unwrap();
        // The failed run must not advance the watermark.
        assert_eq!(last.watermark, Some("2026-07-18T00:00:00Z".parse().unwrap()));
    }

    #[tokio::test]
    async fn watermark_advances_across_successful_runs() {
        let ledger = temp_ledger("advance");
        ledger
            .append(&rec("celestrak", "2026-07-18T00:00:00Z", Outcome::Success))
            .await
            .unwrap();
        ledger
            .append(&rec("celestrak", "2026-07-19T00:00:00Z", Outcome::Success))
            .await
            .unwrap();
        let last = ledger.last_success("celestrak").await.unwrap().unwrap();
        assert_eq!(last.watermark, Some("2026-07-19T00:00:00Z".parse().unwrap()));
    }
}
```

Tests: `append_then_last_success_reads_back`, `last_success_ignores_failures`, `watermark_advances_across_successful_runs`.

<h3 id="task-token-bucket">crates/etl-sources/src/limiter.rs — the token-bucket rate limiter (Part III)</h3>

A classic token bucket built by hand: up to `capacity` tokens, refilling
continuously at `refill_per_sec`; `acquire` sleeps until a token is free. Tested
under paused tokio time so the timing is deterministic.

```rust,ignore
//! A hand-rolled token-bucket rate limiter.
//!
//! Space-Track bans clients that hammer it, so every request must pass through
//! a bucket that refills at a fixed rate. Building it by hand — rather than
//! reaching for `governor` — is the rate-limiting lesson.

use std::sync::Arc;

use tokio::sync::Mutex;
use tokio::time::{Duration, Instant};

/// A classic token bucket: up to `capacity` tokens, refilling continuously at
/// `refill_per_sec`. `acquire` returns as soon as a token is available,
/// sleeping (and letting the bucket refill) when it is empty.
#[derive(Clone)]
pub struct TokenBucket {
    capacity: f64,
    refill_per_sec: f64,
    state: Arc<Mutex<State>>,
}

struct State {
    tokens: f64,
    last: Instant,
}

impl TokenBucket {
    pub fn new(capacity: u32, refill_per_sec: f64) -> Self {
        Self {
            capacity: capacity as f64,
            refill_per_sec,
            state: Arc::new(Mutex::new(State {
                tokens: capacity as f64,
                last: Instant::now(),
            })),
        }
    }

    /// Wait until a token is free, then consume it.
    pub async fn acquire(&self) {
        loop {
            let wait = {
                let mut s = self.state.lock().await;
                let now = Instant::now();
                let elapsed = now.duration_since(s.last).as_secs_f64();
                s.tokens = (s.tokens + elapsed * self.refill_per_sec).min(self.capacity);
                s.last = now;
                if s.tokens >= 1.0 {
                    s.tokens -= 1.0;
                    return;
                }
                // Not enough yet: sleep for the time it takes to refill one token.
                Duration::from_secs_f64((1.0 - s.tokens) / self.refill_per_sec)
            };
            tokio::time::sleep(wait).await;
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test(start_paused = true)]
    async fn bucket_allows_burst_up_to_capacity() {
        let bucket = TokenBucket::new(3, 0.5);
        let start = Instant::now();
        for _ in 0..3 {
            bucket.acquire().await;
        }
        // A full bucket serves the burst with no waiting.
        assert!(start.elapsed() < Duration::from_millis(50));
    }

    #[tokio::test(start_paused = true)]
    async fn bucket_blocks_when_empty_then_refills() {
        let bucket = TokenBucket::new(1, 1.0);
        let start = Instant::now();
        bucket.acquire().await; // consumes the one starting token
        bucket.acquire().await; // must wait ~1s for a refill
        assert!(start.elapsed() >= Duration::from_millis(900));
    }
}
```

Tests: `bucket_allows_burst_up_to_capacity`, `bucket_blocks_when_empty_then_refills`.

<h3 id="task-spacetrack-source">crates/etl-sources/src/spacetrack.rs — the Space-Track source (Part III)</h3>

Cookie-session auth (log in, carry the `Set-Cookie` by hand) plus rate limiting
on every request, then the transform of CDM JSON — where numbers arrive as
strings — into `Row::Conjunction`, the keystone entity. Credentials come from the
environment, never the URL.

```rust,ignore
//! The Space-Track source: cookie-session auth + rate limiting (Extract), plus
//! the transform from CDM JSON into `Row::Conjunction` — the keystone entity.

use async_trait::async_trait;
use chrono::{NaiveDateTime, Utc};
use etl_core::{Conjunction, EtlError, NoradId, RawRecord, Row, Source, Transform};
use serde::Deserialize;

use crate::limiter::TokenBucket;

/// Space-Track login credentials. Kept out of the URL and off disk.
pub struct Credentials {
    pub user: String,
    pub pass: String,
}

impl Credentials {
    pub fn new(user: impl Into<String>, pass: impl Into<String>) -> Self {
        Self {
            user: user.into(),
            pass: pass.into(),
        }
    }

    /// Read credentials from the environment. Missing vars are an auth error.
    pub fn from_env() -> Result<Self, EtlError> {
        let user = std::env::var("SPACETRACK_USER")
            .map_err(|_| EtlError::Auth("SPACETRACK_USER not set".into()))?;
        let pass = std::env::var("SPACETRACK_PASS")
            .map_err(|_| EtlError::Auth("SPACETRACK_PASS not set".into()))?;
        Ok(Self { user, pass })
    }
}

/// Fetches public conjunction data messages, authenticated and rate-limited.
pub struct SpaceTrackSource {
    pub base_url: String,
    pub creds: Credentials,
    pub limiter: TokenBucket,
}

impl SpaceTrackSource {
    pub fn new(base_url: impl Into<String>, creds: Credentials, limiter: TokenBucket) -> Self {
        Self {
            base_url: base_url.into(),
            creds,
            limiter,
        }
    }
}

#[async_trait]
impl Source for SpaceTrackSource {
    fn name(&self) -> &str {
        "spacetrack"
    }

    async fn extract(&self) -> Result<Vec<RawRecord>, EtlError> {
        let base = self.base_url.trim_end_matches('/');
        let client = reqwest::Client::new();

        // 1. Log in — a token is spent on every request, login included.
        self.limiter.acquire().await;
        let login = client
            .post(format!("{base}/ajaxauth/login"))
            .form(&[
                ("identity", self.creds.user.as_str()),
                ("password", self.creds.pass.as_str()),
            ])
            .send()
            .await
            .map_err(|e| EtlError::Network(e.to_string()))?;
        if !login.status().is_success() {
            // 401/403 -> Auth (permanent); 5xx/429 during login stay transient
            // so the executor still retries them. classify_status does both.
            return Err(crate::http::classify_status(login.status(), login.headers()));
        }
        let cookie = login
            .headers()
            .get(reqwest::header::SET_COOKIE)
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.split(';').next())
            .ok_or_else(|| EtlError::Auth("login returned no session cookie".into()))?
            .to_string();

        // 2. Query the CDM class, carrying the session cookie by hand.
        self.limiter.acquire().await;
        let resp = client
            .get(format!(
                "{base}/basicspacedata/query/class/cdm_public/orderby/TCA/format/json"
            ))
            .header(reqwest::header::COOKIE, cookie)
            .send()
            .await
            .map_err(|e| EtlError::Network(e.to_string()))?;
        if !resp.status().is_success() {
            return Err(crate::http::classify_status(resp.status(), resp.headers()));
        }
        let payload = resp.text().await.map_err(|e| EtlError::Network(e.to_string()))?;
        Ok(vec![RawRecord {
            source: self.name().to_string(),
            payload,
            fetched_at: Utc::now(),
        }])
    }
}

/// One CDM as Space-Track serializes it — numbers arrive as strings.
#[derive(Deserialize)]
struct RawCdm {
    #[serde(rename = "TCA")]
    tca: String,
    #[serde(rename = "MIN_RNG")]
    min_rng_m: String,
    #[serde(rename = "PC")]
    pc: Option<String>,
    #[serde(rename = "SAT_1_ID")]
    sat1: String,
    #[serde(rename = "SAT_2_ID")]
    sat2: String,
}

/// Parses Space-Track CDM JSON into conjunctions.
pub struct SpaceTrackTransform;

impl Transform for SpaceTrackTransform {
    fn transform(&self, raw: &RawRecord) -> Result<Vec<Row>, EtlError> {
        let cdms: Vec<RawCdm> =
            serde_json::from_str(&raw.payload).map_err(|e| EtlError::Parse(e.to_string()))?;
        let mut rows = Vec::with_capacity(cdms.len());
        for c in cdms {
            let tca = NaiveDateTime::parse_from_str(&c.tca, "%Y-%m-%dT%H:%M:%S%.f")
                .map_err(|e| EtlError::Parse(format!("bad TCA {:?}: {e}", c.tca)))?
                .and_utc();
            let miss_distance_km = c
                .min_rng_m
                .trim()
                .parse::<f64>()
                .map_err(|_| EtlError::Parse(format!("bad MIN_RNG {:?}", c.min_rng_m)))?
                / 1000.0;
            let collision_probability = match c.pc.as_deref() {
                Some(s) if !s.trim().is_empty() => s
                    .trim()
                    .parse::<f64>()
                    .map_err(|_| EtlError::Parse(format!("bad PC {:?}", c.pc)))?,
                _ => 0.0,
            };
            let primary: NoradId = c.sat1.parse()?;
            let secondary: NoradId = c.sat2.parse()?;
            rows.push(Row::Conjunction(Conjunction {
                id: format!("{}_{}_{}", primary.0, secondary.0, tca.format("%Y%m%dT%H%M%S")),
                tca,
                miss_distance_km,
                collision_probability,
                primary,
                secondary,
            }));
        }
        Ok(rows)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use chrono::Utc;
    use pretty_assertions::assert_eq;
    use wiremock::matchers::{header_exists, method, path};
    use wiremock::{Mock, MockServer, ResponseTemplate};

    const CDM_JSON: &str = r#"[
        {"TCA":"2026-07-20T12:00:00.000000","MIN_RNG":"550.5","PC":"0.0001","SAT_1_ID":"25544","SAT_2_ID":"48274"}
    ]"#;

    fn bucket() -> TokenBucket {
        TokenBucket::new(30, 0.5)
    }

    #[tokio::test]
    async fn login_then_query_uses_cookie() {
        let server = MockServer::start().await;
        Mock::given(method("POST"))
            .and(path("/ajaxauth/login"))
            .respond_with(
                ResponseTemplate::new(200)
                    .insert_header("set-cookie", "chocolatechip=abc123; Path=/"),
            )
            .mount(&server)
            .await;
        Mock::given(method("GET"))
            .and(path("/basicspacedata/query/class/cdm_public/orderby/TCA/format/json"))
            .and(header_exists("cookie"))
            .respond_with(ResponseTemplate::new(200).set_body_string(CDM_JSON))
            .expect(1)
            .mount(&server)
            .await;

        let src = SpaceTrackSource::new(server.uri(), Credentials::new("u", "p"), bucket());
        let raw = src.extract().await.unwrap();
        assert_eq!(raw.len(), 1);
        assert!(raw[0].payload.contains("25544"));
        // .expect(1) verifies on drop that the cookie-guarded query was hit.
    }

    #[tokio::test]
    async fn login_failure_is_auth_error() {
        let server = MockServer::start().await;
        Mock::given(method("POST"))
            .and(path("/ajaxauth/login"))
            .respond_with(ResponseTemplate::new(401))
            .mount(&server)
            .await;
        let src = SpaceTrackSource::new(server.uri(), Credentials::new("u", "bad"), bucket());
        let err = src.extract().await.unwrap_err();
        assert!(matches!(err, EtlError::Auth(_)));
    }

    #[test]
    fn transform_parses_cdm_to_conjunction() {
        let raw = RawRecord {
            source: "spacetrack".into(),
            payload: CDM_JSON.into(),
            fetched_at: Utc::now(),
        };
        let rows = SpaceTrackTransform.transform(&raw).unwrap();
        assert_eq!(rows.len(), 1);
        match &rows[0] {
            Row::Conjunction(c) => {
                assert_eq!(c.primary, NoradId(25544));
                assert_eq!(c.secondary, NoradId(48274));
                assert!((c.miss_distance_km - 0.5505).abs() < 1e-9);
                assert!((c.collision_probability - 0.0001).abs() < 1e-9);
            }
            other => panic!("expected Conjunction, got {other:?}"),
        }
    }
}
```

Tests: `login_then_query_uses_cookie`, `login_failure_is_auth_error`, `transform_parses_cdm_to_conjunction`.

---

## Part IV — The Paginated Arc (TheSpaceDevs)

Retry with capped exponential backoff, content-addressed idempotent writes, and
a cursor-paginated source that follows `next` to the end.

<h3 id="task-etl-core-retry">crates/etl-core/src/retry.rs — retry with backoff (Part IV)</h3>

`with_retry` runs an operation, retrying only transient failures up to
`max_attempts`, backing off exponentially — but a server-supplied `Retry-After`
always wins. Tested under paused time.

```rust,ignore
//! Retry with capped exponential backoff — retrying only what is worth
//! retrying.
//!
//! The policy leans entirely on `EtlError::is_transient()`: permanent errors
//! return immediately, transient ones back off, and a server-supplied
//! `Retry-After` always wins over the computed backoff.

use std::future::Future;

use tokio::time::{Duration, sleep};

use crate::error::EtlError;

#[derive(Clone, Copy, Debug)]
pub struct RetryPolicy {
    pub max_attempts: u32,
    pub base_delay: Duration,
    pub max_delay: Duration,
}

impl Default for RetryPolicy {
    fn default() -> Self {
        Self {
            max_attempts: 4,
            base_delay: Duration::from_millis(200),
            max_delay: Duration::from_secs(10),
        }
    }
}

/// Run `op`, retrying transient failures up to `policy.max_attempts` times.
pub async fn with_retry<F, Fut, T>(policy: RetryPolicy, mut op: F) -> Result<T, EtlError>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T, EtlError>>,
{
    let mut attempt: u32 = 0;
    loop {
        match op().await {
            Ok(value) => return Ok(value),
            Err(err) => {
                attempt += 1;
                if attempt >= policy.max_attempts || !err.is_transient() {
                    return Err(err);
                }
                // A server-requested backoff always wins; otherwise exponential.
                let backoff = err.retry_after().unwrap_or_else(|| {
                    let factor = 2u32.saturating_pow(attempt - 1);
                    (policy.base_delay * factor).min(policy.max_delay)
                });
                sleep(backoff).await;
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::atomic::{AtomicU32, Ordering};
    use std::sync::Arc;

    fn policy() -> RetryPolicy {
        RetryPolicy {
            max_attempts: 4,
            base_delay: Duration::from_millis(100),
            max_delay: Duration::from_secs(10),
        }
    }

    #[tokio::test(start_paused = true)]
    async fn retries_transient_then_succeeds() {
        let calls = Arc::new(AtomicU32::new(0));
        let c = calls.clone();
        let out = with_retry(policy(), || {
            let c = c.clone();
            async move {
                let n = c.fetch_add(1, Ordering::SeqCst);
                if n < 2 {
                    Err(EtlError::Network("reset".into()))
                } else {
                    Ok(42)
                }
            }
        })
        .await
        .unwrap();
        assert_eq!(out, 42);
        assert_eq!(calls.load(Ordering::SeqCst), 3);
    }

    #[tokio::test(start_paused = true)]
    async fn gives_up_on_permanent_immediately() {
        let calls = Arc::new(AtomicU32::new(0));
        let c = calls.clone();
        let err = with_retry(policy(), || {
            let c = c.clone();
            async move {
                c.fetch_add(1, Ordering::SeqCst);
                Err::<(), _>(EtlError::Parse("bad".into()))
            }
        })
        .await
        .unwrap_err();
        assert!(matches!(err, EtlError::Parse(_)));
        assert_eq!(calls.load(Ordering::SeqCst), 1);
    }

    #[tokio::test(start_paused = true)]
    async fn honors_retry_after() {
        let calls = Arc::new(AtomicU32::new(0));
        let c = calls.clone();
        let start = tokio::time::Instant::now();
        let out = with_retry(policy(), || {
            let c = c.clone();
            async move {
                let n = c.fetch_add(1, Ordering::SeqCst);
                if n == 0 {
                    Err(EtlError::RateLimited { retry_after_secs: 5 })
                } else {
                    Ok(())
                }
            }
        })
        .await;
        assert!(out.is_ok());
        // The 5s Retry-After beats the 100ms base backoff.
        assert!(start.elapsed() >= Duration::from_secs(5));
    }
}
```

Tests: `retries_transient_then_succeeds`, `gives_up_on_permanent_immediately`, `honors_retry_after`.

<h3 id="task-etl-core-content">crates/etl-core/src/content.rs — content-addressing & the idempotent sink (Part IV)</h3>

`content_id` derives a stable `sha256(kind ‖ 0x00 ‖ canonical)` id; `Row::content_key`
uses it as a dedupe key (fallible by design). `IdempotentSink` wraps any `Sink`
and forwards only rows it has not seen — the precondition for any retry or
backfill.

```rust,ignore
//! Content-addressing and idempotent writes.
//!
//! A stable id derived from a row's *content* means re-running a pipeline
//! writes each row at most once — the precondition for any retry, backfill, or
//! scheduling story. This is where the `sha2` seam stops being decorative.

use std::collections::HashSet;

use async_trait::async_trait;
use sha2::{Digest, Sha256};
use tokio::sync::Mutex;

use crate::error::EtlError;
use crate::pipeline::{Row, Sink};

/// A deterministic id for a value: `sha256(kind ‖ 0x00 ‖ canonical)`, hex.
///
/// The `kind` tag keeps ids from two different entity types from ever
/// colliding even if their canonical strings coincide.
pub fn content_id(kind: &str, canonical: &str) -> String {
    let mut hasher = Sha256::new();
    hasher.update(kind.as_bytes());
    hasher.update([0u8]);
    hasher.update(canonical.as_bytes());
    format!("{:x}", hasher.finalize())
}

impl Row {
    /// The dedupe key for this row — the content id of its canonical JSON.
    ///
    /// Fallible on purpose: a non-serializable field (e.g. a `NaN` float) must
    /// surface as an error, never silently collapse to a shared key that would
    /// make [`IdempotentSink`] discard distinct rows as duplicates.
    pub fn content_key(&self) -> Result<String, EtlError> {
        // serde_json is deterministic for our structs (fields in declaration
        // order), so equal rows hash equal.
        let canonical = serde_json::to_string(self).map_err(|e| EtlError::Parse(e.to_string()))?;
        Ok(content_id("row", &canonical))
    }
}

/// Wraps any `Sink`, forwarding only rows whose content it has not seen before.
///
/// The `seen` set is in-memory (per run). Seeding it from an existing sink file
/// at construction is the cross-run refinement noted in the course.
pub struct IdempotentSink<S: Sink> {
    inner: S,
    seen: Mutex<HashSet<String>>,
}

impl<S: Sink> IdempotentSink<S> {
    pub fn new(inner: S) -> Self {
        Self {
            inner,
            seen: Mutex::new(HashSet::new()),
        }
    }

    /// Pre-load already-written keys so re-runs across processes stay idempotent.
    pub fn with_seen(inner: S, keys: impl IntoIterator<Item = String>) -> Self {
        Self {
            inner,
            seen: Mutex::new(keys.into_iter().collect()),
        }
    }
}

#[async_trait]
impl<S: Sink> Sink for IdempotentSink<S> {
    async fn load(&self, rows: &[Row]) -> Result<usize, EtlError> {
        let mut fresh = Vec::new();
        {
            let mut seen = self.seen.lock().await;
            for row in rows {
                if seen.insert(row.content_key()?) {
                    fresh.push(row.clone());
                }
            }
        }
        if fresh.is_empty() {
            return Ok(0);
        }
        self.inner.load(&fresh).await
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::domain::Launch;
    use pretty_assertions::assert_eq;
    use std::sync::atomic::{AtomicUsize, Ordering};

    fn launch(id: &str) -> Row {
        Row::Launch(Launch {
            id: id.into(),
            name: "L".into(),
            net: "2026-07-10T12:00:00Z".parse().unwrap(),
            provider: "SpaceX".into(),
            payloads: vec![],
            description: "d".into(),
        })
    }

    /// A counting sink that records how many rows actually reached it.
    struct CountingSink {
        total: AtomicUsize,
    }
    #[async_trait]
    impl Sink for CountingSink {
        async fn load(&self, rows: &[Row]) -> Result<usize, EtlError> {
            self.total.fetch_add(rows.len(), Ordering::SeqCst);
            Ok(rows.len())
        }
    }

    #[test]
    fn same_content_hashes_stable_distinct_content_differs() {
        assert_eq!(content_id("row", "x"), content_id("row", "x"));
        assert_ne!(content_id("row", "x"), content_id("row", "y"));
        assert_ne!(content_id("a", "x"), content_id("b", "x"));
    }

    #[tokio::test]
    async fn rerun_writes_zero_new_rows() {
        let sink = IdempotentSink::new(CountingSink {
            total: AtomicUsize::new(0),
        });
        let rows = vec![launch("a"), launch("b")];
        assert_eq!(sink.load(&rows).await.unwrap(), 2);
        // Same rows again: nothing new reaches the inner sink.
        assert_eq!(sink.load(&rows).await.unwrap(), 0);
    }

    #[test]
    fn content_key_is_stable_and_distinguishes_rows() {
        // Equal rows -> equal keys; a changed field -> a different key. This is
        // the contract IdempotentSink relies on, and it is fallible by design
        // (never unwrap_or_default, which would collapse distinct rows to one).
        let a = launch("a");
        let a2 = launch("a");
        let b = launch("b");
        assert_eq!(a.content_key().unwrap(), a2.content_key().unwrap());
        assert_ne!(a.content_key().unwrap(), b.content_key().unwrap());
    }

    #[tokio::test]
    async fn distinct_rows_all_written() {
        let sink = IdempotentSink::new(CountingSink {
            total: AtomicUsize::new(0),
        });
        let rows = vec![launch("a"), launch("b"), launch("c")];
        assert_eq!(sink.load(&rows).await.unwrap(), 3);
    }
}
```

Tests: `same_content_hashes_stable_distinct_content_differs`, `rerun_writes_zero_new_rows`, `content_key_is_stable_and_distinguishes_rows`, `distinct_rows_all_written`.

<h3 id="task-spacedevs-source">crates/etl-sources/src/spacedevs.rs — the TheSpaceDevs source (Part IV)</h3>

Cursor pagination: `extract` follows the `next` cursor to the end, wraps each
page fetch in `with_retry`, and guards against a cyclic `next` with a visited
set. The transform pulls launches out of deeply nested Launch Library 2 JSON.

```rust,ignore
//! The TheSpaceDevs source (Launch Library 2): cursor pagination over a
//! throttled REST API (Extract), plus the transform of deeply nested launch
//! JSON into `Row::Launch`.

use async_trait::async_trait;
use chrono::{DateTime, Utc};
use etl_core::{EtlError, Launch, RawRecord, Row, Source, Transform};
use serde::Deserialize;

use crate::http::classify_status;
use etl_core::{with_retry, RetryPolicy};

/// Fetches launches, following the `next` cursor to the end.
pub struct SpaceDevsSource {
    pub base_url: String,
    pub policy: RetryPolicy,
}

impl SpaceDevsSource {
    pub fn new(base_url: impl Into<String>) -> Self {
        Self {
            base_url: base_url.into(),
            policy: RetryPolicy::default(),
        }
    }
}

#[async_trait]
impl Source for SpaceDevsSource {
    fn name(&self) -> &str {
        "spacedevs"
    }

    async fn extract(&self) -> Result<Vec<RawRecord>, EtlError> {
        let base = self.base_url.trim_end_matches('/');
        let client = reqwest::Client::new();
        let mut next = Some(format!("{base}/2.2.0/launch/?limit=100&mode=detailed"));
        let mut pages = Vec::new();
        // Guard against a cyclic or self-referential `next` from a buggy server.
        let mut visited = std::collections::HashSet::new();

        while let Some(url) = next {
            if !visited.insert(url.clone()) {
                return Err(EtlError::Parse(format!("pagination cycle at {url}")));
            }
            let body = with_retry(self.policy, || {
                let client = &client;
                let url = url.clone();
                async move {
                    let resp = client
                        .get(&url)
                        .send()
                        .await
                        .map_err(|e| EtlError::Network(e.to_string()))?;
                    // 429 -> RateLimited, 4xx -> permanent, 5xx -> transient.
                    if !resp.status().is_success() {
                        return Err(classify_status(resp.status(), resp.headers()));
                    }
                    resp.text().await.map_err(|e| EtlError::Network(e.to_string()))
                }
            })
            .await?;

            let page: Page =
                serde_json::from_str(&body).map_err(|e| EtlError::Parse(e.to_string()))?;
            next = page.next;
            pages.push(RawRecord {
                source: self.name().to_string(),
                payload: body,
                fetched_at: Utc::now(),
            });
        }
        Ok(pages)
    }
}

#[derive(Deserialize)]
struct Page {
    next: Option<String>,
    #[allow(dead_code)]
    results: Vec<RawLaunch>,
}

#[derive(Deserialize)]
struct RawLaunch {
    id: String,
    name: String,
    net: String,
    launch_service_provider: Option<Provider>,
    mission: Option<Mission>,
}

#[derive(Deserialize)]
struct Provider {
    name: String,
}

#[derive(Deserialize)]
struct Mission {
    name: Option<String>,
    description: Option<String>,
}

/// Parses one Launch Library 2 page into launches.
pub struct SpaceDevsTransform;

impl Transform for SpaceDevsTransform {
    fn transform(&self, raw: &RawRecord) -> Result<Vec<Row>, EtlError> {
        let page: Page =
            serde_json::from_str(&raw.payload).map_err(|e| EtlError::Parse(e.to_string()))?;
        let mut rows = Vec::with_capacity(page.results.len());
        for l in page.results {
            let net = l
                .net
                .parse::<DateTime<Utc>>()
                .map_err(|e| EtlError::Parse(format!("bad net {:?}: {e}", l.net)))?;
            let (payloads, description) = match l.mission {
                Some(m) => (
                    m.name.into_iter().collect::<Vec<String>>(),
                    m.description.unwrap_or_default(),
                ),
                None => (vec![], String::new()),
            };
            rows.push(Row::Launch(Launch {
                id: l.id,
                name: l.name,
                net,
                provider: l.launch_service_provider.map(|p| p.name).unwrap_or_default(),
                payloads,
                description,
            }));
        }
        Ok(rows)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use etl_core::Row;
    use pretty_assertions::assert_eq;
    use wiremock::matchers::{method, path};
    use wiremock::{Mock, MockServer, ResponseTemplate};

    fn page_json(next: Option<&str>) -> String {
        let next_field = match next {
            Some(u) => format!("\"{u}\""),
            None => "null".to_string(),
        };
        format!(
            r#"{{"next": {next_field}, "results": [
                {{"id":"ll2-1","name":"Falcon 9 | Starlink","net":"2026-07-10T12:00:00Z",
                  "launch_service_provider":{{"name":"SpaceX"}},
                  "mission":{{"name":"Starlink G-99","description":"A batch of broadband satellites."}}}}
            ]}}"#
        )
    }

    #[tokio::test]
    async fn follows_next_until_null() {
        let server = MockServer::start().await;
        let page2 = format!("{}/page2", server.uri());
        Mock::given(method("GET"))
            .and(path("/2.2.0/launch/"))
            .respond_with(ResponseTemplate::new(200).set_body_string(page_json(Some(&page2))))
            .mount(&server)
            .await;
        Mock::given(method("GET"))
            .and(path("/page2"))
            .respond_with(ResponseTemplate::new(200).set_body_string(page_json(None)))
            .mount(&server)
            .await;

        let src = SpaceDevsSource::new(server.uri());
        let pages = src.extract().await.unwrap();
        assert_eq!(pages.len(), 2);
    }

    #[tokio::test]
    async fn rejects_pagination_cycle() {
        let server = MockServer::start().await;
        // The first page's `next` points back at the first page — a cycle.
        let self_url = format!("{}/2.2.0/launch/", server.uri());
        Mock::given(method("GET"))
            .and(path("/2.2.0/launch/"))
            .respond_with(ResponseTemplate::new(200).set_body_string(page_json(Some(&self_url))))
            .mount(&server)
            .await;
        let src = SpaceDevsSource::new(server.uri());
        let err = src.extract().await.unwrap_err();
        assert!(matches!(err, EtlError::Parse(_)));
    }

    #[tokio::test]
    async fn http_error_is_network_error() {
        let server = MockServer::start().await;
        Mock::given(method("GET"))
            .respond_with(ResponseTemplate::new(500))
            .mount(&server)
            .await;
        let src = SpaceDevsSource::new(server.uri());
        let err = src.extract().await.unwrap_err();
        assert!(matches!(err, EtlError::Network(_)));
    }

    #[test]
    fn parses_nested_launch_json() {
        let raw = RawRecord {
            source: "spacedevs".into(),
            payload: page_json(None),
            fetched_at: Utc::now(),
        };
        let rows = SpaceDevsTransform.transform(&raw).unwrap();
        assert_eq!(rows.len(), 1);
        match &rows[0] {
            Row::Launch(l) => {
                assert_eq!(l.provider, "SpaceX");
                assert_eq!(l.payloads, vec!["Starlink G-99".to_string()]);
                assert!(l.description.contains("broadband"));
            }
            other => panic!("expected Launch, got {other:?}"),
        }
    }
}
```

Tests: `follows_next_until_null`, `rejects_pagination_cycle`, `http_error_is_network_error`, `parses_nested_launch_json`.

---

## Part V — The Orchestration Arc

The typed task DAG, the executor that runs it (with retry, ledger recording,
freshness gating, and block propagation), and the `panoptes-etl` binary that
wires the concrete sources in — the only crate that names them.

<h3 id="task-etl-orchestrate-cargo">crates/etl-orchestrate/Cargo.toml — the orchestrator manifest (Part V)</h3>

Depends on `etl-core` **only** — the dependency-inversion constraint made
mechanical. `cargo tree -p etl-orchestrate` proves no concrete source is
reachable from here.

```toml
[package]
name = "etl-orchestrate"
version = "0.1.0"
edition = "2024"

# Depends on etl-core ONLY. It schedules `dyn Source` and must never know a
# concrete source exists — that constraint is the dependency-inversion lesson.
[dependencies]
etl-core = { path = "../etl-core" }
chrono = { workspace = true }

[dev-dependencies]
pretty_assertions = { workspace = true }
tokio = { workspace = true }
async-trait = { workspace = true }
```

<h3 id="task-etl-orchestrate-lib">crates/etl-orchestrate/src/lib.rs — the orchestrator crate root (Part V)</h3>

```rust,ignore
//! `etl-orchestrate` — a typed task DAG and the executor that runs it.
//!
//! Depends on `etl-core` only. The executor schedules `dyn Source` /
//! `dyn Transform` / `dyn Sink` and has no knowledge of any concrete source —
//! that separation is enforced by the crate boundary, not by convention.

pub mod executor;
pub mod graph;

pub use executor::{Executor, Pipeline, RunReport};
pub use graph::{Dag, TaskId};
```

<h3 id="task-dag-graph">crates/etl-orchestrate/src/graph.rs — the task DAG (Part V)</h3>

A directed acyclic graph of `TaskId`s with `topological_order` via Kahn's
algorithm over a sorted ready-set, so the order is deterministic and a remaining
node signals a cycle.

```rust,ignore
//! A typed task DAG with deterministic topological execution order.

use std::collections::{BTreeSet, HashMap, HashSet};

use etl_core::EtlError;

/// The identifier of a task in the DAG.
#[derive(Debug, Clone, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct TaskId(pub String);

impl TaskId {
    pub fn new(s: impl Into<String>) -> Self {
        TaskId(s.into())
    }
}

/// A directed acyclic graph of tasks. Edges record prerequisites: `add_dep(b,
/// a)` means "b depends on a", so a runs before b.
#[derive(Default)]
pub struct Dag {
    nodes: HashSet<TaskId>,
    /// task -> its prerequisites.
    prereqs: HashMap<TaskId, Vec<TaskId>>,
}

impl Dag {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn add_task(&mut self, id: impl Into<String>) -> TaskId {
        let t = TaskId(id.into());
        self.nodes.insert(t.clone());
        self.prereqs.entry(t.clone()).or_default();
        t
    }

    /// Record that `task` depends on `depends_on`.
    pub fn add_dep(&mut self, task: &TaskId, depends_on: &TaskId) {
        self.nodes.insert(task.clone());
        self.nodes.insert(depends_on.clone());
        self.prereqs.entry(depends_on.clone()).or_default();
        self.prereqs
            .entry(task.clone())
            .or_default()
            .push(depends_on.clone());
    }

    /// The direct prerequisites of a task (empty if none / unknown).
    pub fn prerequisites(&self, task: &TaskId) -> &[TaskId] {
        self.prereqs.get(task).map_or(&[], |v| v.as_slice())
    }

    /// Kahn's algorithm with a sorted ready-set, so the order is deterministic.
    /// A remaining node means a cycle.
    pub fn topological_order(&self) -> Result<Vec<TaskId>, EtlError> {
        let mut indegree: HashMap<TaskId, usize> = self
            .nodes
            .iter()
            .map(|n| (n.clone(), self.prereqs.get(n).map_or(0, |p| p.len())))
            .collect();

        let mut dependents: HashMap<TaskId, Vec<TaskId>> = HashMap::new();
        for (task, prereqs) in &self.prereqs {
            for p in prereqs {
                dependents.entry(p.clone()).or_default().push(task.clone());
            }
        }

        let mut ready: BTreeSet<TaskId> = indegree
            .iter()
            .filter(|(_, d)| **d == 0)
            .map(|(t, _)| t.clone())
            .collect();

        let mut order = Vec::with_capacity(self.nodes.len());
        while let Some(next) = ready.iter().next().cloned() {
            ready.remove(&next);
            if let Some(children) = dependents.get(&next) {
                for child in children {
                    let d = indegree.get_mut(child).expect("child in indegree");
                    *d -= 1;
                    if *d == 0 {
                        ready.insert(child.clone());
                    }
                }
            }
            order.push(next);
        }

        if order.len() != self.nodes.len() {
            return Err(EtlError::Parse(format!(
                "cycle detected: ordered {} of {} tasks",
                order.len(),
                self.nodes.len()
            )));
        }
        Ok(order)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use pretty_assertions::assert_eq;

    fn ids(v: &[TaskId]) -> Vec<String> {
        v.iter().map(|t| t.0.clone()).collect()
    }

    #[test]
    fn linear_chain_orders_correctly() {
        let mut dag = Dag::new();
        let a = dag.add_task("a");
        let b = dag.add_task("b");
        let c = dag.add_task("c");
        dag.add_dep(&b, &a);
        dag.add_dep(&c, &b);
        assert_eq!(ids(&dag.topological_order().unwrap()), ["a", "b", "c"]);
    }

    #[test]
    fn diamond_orders_deps_before_dependents() {
        let mut dag = Dag::new();
        let a = dag.add_task("a");
        let b = dag.add_task("b");
        let c = dag.add_task("c");
        let d = dag.add_task("d");
        dag.add_dep(&b, &a);
        dag.add_dep(&c, &a);
        dag.add_dep(&d, &b);
        dag.add_dep(&d, &c);
        let order = ids(&dag.topological_order().unwrap());
        // a before everything; d after everything; b,c deterministic (sorted).
        assert_eq!(order, ["a", "b", "c", "d"]);
    }

    #[test]
    fn cycle_is_rejected() {
        let mut dag = Dag::new();
        let a = dag.add_task("a");
        let b = dag.add_task("b");
        dag.add_dep(&a, &b);
        dag.add_dep(&b, &a);
        let err = dag.topological_order().unwrap_err();
        assert!(matches!(err, EtlError::Parse(_)));
    }
}
```

Tests: `linear_chain_orders_correctly`, `diamond_orders_deps_before_dependents`, `cycle_is_rejected`.

<h3 id="task-executor">crates/etl-orchestrate/src/executor.rs — the executor (Part V)</h3>

`Pipeline` bundles a boxed `dyn Source`/`dyn Transform`/`dyn Sink` with a retry
policy and an optional freshness window. `Executor::run` walks the DAG in
topological order, skips still-fresh sources, records every run in the ledger,
and propagates a block downstream when a prerequisite fails. It never names a
concrete source.

```rust,ignore
//! The executor: runs pipelines in topological order, retries transient
//! failures, records every run in the ledger, and skips sources still fresh
//! from a recent successful run.
//!
//! It is built entirely on `etl-core` traits — `dyn Source`, `dyn Transform`,
//! `dyn Sink` — and never names a concrete source.

use std::collections::HashMap;

use chrono::{Duration, Utc};
use etl_core::{
    with_retry, EtlError, Ledger, Outcome, RetryPolicy, RunRecord, Sink, Source, Transform,
};

use crate::graph::{Dag, TaskId};

/// One extract → transform → load chain.
pub struct Pipeline {
    pub source: Box<dyn Source>,
    pub transform: Box<dyn Transform>,
    pub sink: Box<dyn Sink>,
    pub policy: RetryPolicy,
    /// If set, skip the run when the last success is newer than this.
    pub refresh_after: Option<Duration>,
}

impl Pipeline {
    pub fn new(
        source: Box<dyn Source>,
        transform: Box<dyn Transform>,
        sink: Box<dyn Sink>,
    ) -> Self {
        Self {
            source,
            transform,
            sink,
            policy: RetryPolicy::default(),
            refresh_after: None,
        }
    }

    /// Extract (with retry) → transform every raw record → load once.
    pub async fn run_once(&self) -> Result<usize, EtlError> {
        let raws = with_retry(self.policy, || self.source.extract()).await?;
        let mut rows = Vec::new();
        for raw in &raws {
            rows.extend(self.transform.transform(raw)?);
        }
        self.sink.load(&rows).await
    }

    async fn is_fresh(&self, ledger: &Ledger, id: &TaskId) -> Result<bool, EtlError> {
        let Some(refresh) = self.refresh_after else {
            return Ok(false);
        };
        let Some(last) = ledger.last_success(&id.0).await? else {
            return Ok(false);
        };
        Ok(Utc::now() - last.finished_at < refresh)
    }
}

/// The outcome of a whole DAG run.
#[derive(Debug, Default)]
pub struct RunReport {
    pub records: Vec<RunRecord>,
    pub skipped: Vec<TaskId>,
    /// Nodes not run because a prerequisite failed (or was itself blocked).
    pub blocked: Vec<TaskId>,
}

/// Owns the DAG, the pipelines, and the ledger.
pub struct Executor {
    dag: Dag,
    pipelines: HashMap<TaskId, Pipeline>,
    ledger: Ledger,
}

impl Executor {
    pub fn new(ledger: Ledger) -> Self {
        Self {
            dag: Dag::new(),
            pipelines: HashMap::new(),
            ledger,
        }
    }

    pub fn add_pipeline(&mut self, id: &str, pipeline: Pipeline) -> TaskId {
        let t = self.dag.add_task(id);
        self.pipelines.insert(t.clone(), pipeline);
        t
    }

    pub fn add_dep(&mut self, task: &TaskId, depends_on: &TaskId) {
        self.dag.add_dep(task, depends_on);
    }

    /// Run every pipeline in dependency order.
    ///
    /// A node whose prerequisite failed (or was itself blocked) does not run —
    /// the DAG edges gate execution, not just order it. `bad` accumulates both
    /// failed and blocked ids so the block propagates down the graph.
    pub async fn run(&self) -> Result<RunReport, EtlError> {
        let order = self.dag.topological_order()?;
        let mut report = RunReport::default();
        let mut bad: std::collections::HashSet<TaskId> = std::collections::HashSet::new();
        for id in order {
            let Some(pipeline) = self.pipelines.get(&id) else {
                continue;
            };
            if self.dag.prerequisites(&id).iter().any(|p| bad.contains(p)) {
                bad.insert(id.clone());
                report.blocked.push(id);
                continue;
            }
            if pipeline.is_fresh(&self.ledger, &id).await? {
                report.skipped.push(id.clone());
                continue;
            }
            let started_at = Utc::now();
            let result = pipeline.run_once().await;
            let finished_at = Utc::now();
            let record = match result {
                Ok(rows_written) => RunRecord {
                    source: id.0.clone(),
                    started_at,
                    finished_at,
                    outcome: Outcome::Success,
                    watermark: Some(finished_at),
                    rows_written,
                },
                Err(e) => {
                    bad.insert(id.clone());
                    RunRecord {
                        source: id.0.clone(),
                        started_at,
                        finished_at,
                        outcome: Outcome::Failed { reason: e.to_string() },
                        watermark: None,
                        rows_written: 0,
                    }
                }
            };
            self.ledger.append(&record).await?;
            report.records.push(record);
        }
        Ok(report)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use async_trait::async_trait;
    use etl_core::{RawRecord, Row};
    use pretty_assertions::assert_eq;
    use std::sync::atomic::{AtomicUsize, Ordering};
    use std::sync::{Arc, Mutex};

    /// Records the order in which sources are extracted, and counts calls.
    struct StubSource {
        name: String,
        log: Arc<Mutex<Vec<String>>>,
        calls: Arc<AtomicUsize>,
    }
    #[async_trait]
    impl Source for StubSource {
        fn name(&self) -> &str {
            &self.name
        }
        async fn extract(&self) -> Result<Vec<RawRecord>, EtlError> {
            self.calls.fetch_add(1, Ordering::SeqCst);
            self.log.lock().unwrap().push(self.name.clone());
            Ok(vec![RawRecord {
                source: self.name.clone(),
                payload: String::new(),
                fetched_at: Utc::now(),
            }])
        }
    }

    struct EmptyTransform;
    impl Transform for EmptyTransform {
        fn transform(&self, _raw: &RawRecord) -> Result<Vec<Row>, EtlError> {
            Ok(vec![])
        }
    }

    struct NullSink;
    #[async_trait]
    impl Sink for NullSink {
        async fn load(&self, rows: &[Row]) -> Result<usize, EtlError> {
            Ok(rows.len())
        }
    }

    fn temp_ledger(tag: &str) -> Ledger {
        let dir = std::env::temp_dir().join(format!("etl-exec-{tag}-{}", std::process::id()));
        std::fs::create_dir_all(&dir).unwrap();
        let path = dir.join("runs.jsonl");
        let _ = std::fs::remove_file(&path);
        Ledger::new(path)
    }

    fn stub_pipeline(name: &str, log: Arc<Mutex<Vec<String>>>, calls: Arc<AtomicUsize>) -> Pipeline {
        Pipeline::new(
            Box::new(StubSource {
                name: name.into(),
                log,
                calls,
            }),
            Box::new(EmptyTransform),
            Box::new(NullSink),
        )
    }

    #[tokio::test]
    async fn runs_pipelines_in_dependency_order() {
        let log = Arc::new(Mutex::new(Vec::new()));
        let mut exec = Executor::new(temp_ledger("order"));
        let a = exec.add_pipeline(
            "a",
            stub_pipeline("a", log.clone(), Arc::new(AtomicUsize::new(0))),
        );
        let b = exec.add_pipeline(
            "b",
            stub_pipeline("b", log.clone(), Arc::new(AtomicUsize::new(0))),
        );
        exec.add_dep(&b, &a);

        let report = exec.run().await.unwrap();
        assert_eq!(report.records.len(), 2);
        assert_eq!(*log.lock().unwrap(), vec!["a".to_string(), "b".to_string()]);
    }

    #[tokio::test]
    async fn records_a_run_per_node() {
        let mut exec = Executor::new(temp_ledger("perrun"));
        exec.add_pipeline(
            "a",
            stub_pipeline("a", Arc::new(Mutex::new(vec![])), Arc::new(AtomicUsize::new(0))),
        );
        exec.add_pipeline(
            "b",
            stub_pipeline("b", Arc::new(Mutex::new(vec![])), Arc::new(AtomicUsize::new(0))),
        );
        let report = exec.run().await.unwrap();
        assert_eq!(report.records.len(), 2);
        assert!(report.records.iter().all(|r| r.outcome == Outcome::Success));
    }

    /// A source that always fails permanently.
    struct FailingSource {
        calls: Arc<AtomicUsize>,
    }
    #[async_trait]
    impl Source for FailingSource {
        fn name(&self) -> &str {
            "failing"
        }
        async fn extract(&self) -> Result<Vec<RawRecord>, EtlError> {
            self.calls.fetch_add(1, Ordering::SeqCst);
            Err(EtlError::Parse("boom".into()))
        }
    }

    #[tokio::test]
    async fn failed_upstream_blocks_dependent() {
        let mut exec = Executor::new(temp_ledger("block"));
        let a = exec.add_pipeline(
            "a",
            Pipeline::new(
                Box::new(FailingSource { calls: Arc::new(AtomicUsize::new(0)) }),
                Box::new(EmptyTransform),
                Box::new(NullSink),
            ),
        );
        let downstream_calls = Arc::new(AtomicUsize::new(0));
        let b = exec.add_pipeline(
            "b",
            stub_pipeline("b", Arc::new(Mutex::new(vec![])), downstream_calls.clone()),
        );
        exec.add_dep(&b, &a);

        let report = exec.run().await.unwrap();
        // a ran and failed; b was blocked and never extracted.
        assert_eq!(report.records.len(), 1);
        assert!(matches!(report.records[0].outcome, Outcome::Failed { .. }));
        assert_eq!(report.blocked, vec![b]);
        assert_eq!(downstream_calls.load(Ordering::SeqCst), 0);
    }

    #[tokio::test]
    async fn incremental_skips_up_to_date_source() {
        let ledger = temp_ledger("incr");
        // Seed a very recent successful run for "celestrak".
        ledger
            .append(&RunRecord {
                source: "celestrak".into(),
                started_at: Utc::now(),
                finished_at: Utc::now(),
                outcome: Outcome::Success,
                watermark: Some(Utc::now()),
                rows_written: 5,
            })
            .await
            .unwrap();

        let calls = Arc::new(AtomicUsize::new(0));
        let mut pipeline =
            stub_pipeline("celestrak", Arc::new(Mutex::new(vec![])), calls.clone());
        pipeline.refresh_after = Some(Duration::hours(6));

        let mut exec = Executor::new(ledger);
        exec.add_pipeline("celestrak", pipeline);

        let report = exec.run().await.unwrap();
        assert_eq!(report.skipped.len(), 1);
        assert!(report.records.is_empty());
        // The source was never extracted.
        assert_eq!(calls.load(Ordering::SeqCst), 0);
    }
}
```

Tests: `runs_pipelines_in_dependency_order`, `records_a_run_per_node`, `failed_upstream_blocks_dependent`, `incremental_skips_up_to_date_source`.

<h3 id="task-panoptes-etl-cargo">crates/panoptes-etl/Cargo.toml — the binary manifest (Part V)</h3>

The binary — the only crate that names concrete sources and wires them into the
DAG. It depends on all three library crates.

```toml
[package]
name = "panoptes-etl"
version = "0.1.0"
edition = "2024"

# The binary — the ONLY crate that names concrete sources and wires them into
# the DAG. Depends on all three library crates.
[dependencies]
etl-core = { path = "../etl-core" }
etl-sources = { path = "../etl-sources" }
etl-orchestrate = { path = "../etl-orchestrate" }
serde_json = { workspace = true }
clap = { workspace = true }
anyhow = { workspace = true }
tokio = { workspace = true }
chrono = { workspace = true }

[dev-dependencies]
pretty_assertions = { workspace = true }
```

<h3 id="task-cli">crates/panoptes-etl/src/cli.rs — the CLI surface (Part V)</h3>

The clap `Cli`/`Command` definitions and the ledger-status report — the testable
parts of the binary, kept out of `main.rs` so they can be unit-tested.

```rust,ignore
//! CLI surface and the ledger-status report — the testable parts of the binary.

use std::path::PathBuf;

use clap::{Parser, Subcommand};
use etl_core::{EtlError, Ledger};

/// The Panoptes space-data ETL.
#[derive(Parser, Debug)]
#[command(name = "panoptes-etl", version, about)]
pub struct Cli {
    #[command(subcommand)]
    pub command: Command,

    /// Directory for normalized JSONL output.
    #[arg(long, default_value = "data")]
    pub out_dir: PathBuf,

    /// Path to the run ledger.
    #[arg(long, default_value = "data/runs.jsonl")]
    pub ledger: PathBuf,
}

#[derive(Subcommand, Debug, PartialEq, Eq)]
pub enum Command {
    /// Run every pipeline once, skipping sources still fresh.
    Run,
    /// Re-run every pipeline, ignoring freshness (full backfill).
    Backfill,
    /// Embed the prose fields of already-loaded launches into a vector index.
    Embed,
    /// Show the last successful watermark per source.
    Status,
}

/// The sources this binary knows how to run, in DAG order.
pub const SOURCES: &[&str] = &["celestrak", "spacetrack", "spacedevs"];

/// Render a status report: the last successful watermark per known source.
pub async fn status_report(ledger: &Ledger, sources: &[&str]) -> Result<String, EtlError> {
    let mut lines = Vec::new();
    for source in sources {
        let line = match ledger.last_success(source).await? {
            Some(rec) => {
                let wm = rec
                    .watermark
                    .map(|w| w.to_rfc3339())
                    .unwrap_or_else(|| "-".into());
                format!("{source:<12} last success {} ({} rows)", wm, rec.rows_written)
            }
            None => format!("{source:<12} never run"),
        };
        lines.push(line);
    }
    Ok(lines.join("\n"))
}

#[cfg(test)]
mod tests {
    use super::*;
    use etl_core::{Outcome, RunRecord};
    use chrono::Utc;

    #[test]
    fn cli_parses_run_subcommand() {
        let cli = Cli::try_parse_from(["panoptes-etl", "run"]).unwrap();
        assert_eq!(cli.command, Command::Run);
    }

    #[test]
    fn cli_parses_flags_before_subcommand() {
        let cli =
            Cli::try_parse_from(["panoptes-etl", "--out-dir", "/tmp/x", "status"]).unwrap();
        assert_eq!(cli.command, Command::Status);
        assert_eq!(cli.out_dir, PathBuf::from("/tmp/x"));
    }

    #[tokio::test]
    async fn status_reports_last_watermark() {
        let dir = std::env::temp_dir().join(format!("etl-cli-{}", std::process::id()));
        std::fs::create_dir_all(&dir).unwrap();
        let path = dir.join("runs.jsonl");
        let _ = std::fs::remove_file(&path);
        let ledger = Ledger::new(&path);
        ledger
            .append(&RunRecord {
                source: "celestrak".into(),
                started_at: Utc::now(),
                finished_at: Utc::now(),
                outcome: Outcome::Success,
                watermark: Some("2026-07-19T00:00:00Z".parse().unwrap()),
                rows_written: 42,
            })
            .await
            .unwrap();

        let report = status_report(&ledger, SOURCES).await.unwrap();
        assert!(report.contains("celestrak"));
        assert!(report.contains("2026-07-19"));
        assert!(report.contains("42 rows"));
        assert!(report.contains("spacedevs"));
        assert!(report.contains("never run"));
    }
}
```

Tests: `cli_parses_run_subcommand`, `cli_parses_flags_before_subcommand`, `status_reports_last_watermark`.

<h3 id="task-main">crates/panoptes-etl/src/main.rs — the binary entry point (Part V)</h3>

Wires the concrete sources into the DAG and dispatches the subcommands. This is
the only file in the workspace that names `CelestrakSource`, `SpaceTrackSource`,
`SpaceDevsSource`, and `VoyageEmbedder`. The commented capstone block shows where
your NASA source goes — nothing structural changes to admit a fourth source.

```rust,ignore
//! The `panoptes-etl` binary: wires the concrete sources into a DAG and runs
//! it. This is the only crate in the workspace that names a concrete source.

mod cli;

use std::path::Path;

use anyhow::Result;
use chrono::Duration;
use clap::Parser;
use std::path::PathBuf;

use etl_core::{EmbedSink, EmbeddedRow, IdempotentSink, JsonlSink, Ledger, Row, Sink};
use etl_orchestrate::{Executor, Pipeline};
use etl_sources::{
    CelestrakSource, CelestrakTransform, Credentials, SpaceDevsSource, SpaceDevsTransform,
    SpaceTrackSource, SpaceTrackTransform, TokenBucket, VoyageEmbedder,
};

use cli::{status_report, Cli, Command, SOURCES};

const CELESTRAK_BASE: &str = "https://celestrak.org";
const SPACETRACK_BASE: &str = "https://www.space-track.org";
const SPACEDEVS_BASE: &str = "https://ll.thespacedevs.com";

#[tokio::main]
async fn main() -> Result<()> {
    let cli = Cli::parse();
    tokio::fs::create_dir_all(&cli.out_dir).await?;
    let ledger = Ledger::new(&cli.ledger);

    match cli.command {
        Command::Status => {
            println!("{}", status_report(&ledger, SOURCES).await?);
        }
        Command::Run | Command::Backfill => {
            let incremental = cli.command == Command::Run;
            let executor = build_executor(&cli.out_dir, ledger, incremental).await?;
            let report = executor.run().await?;
            for r in &report.records {
                println!("{:<12} {:?} ({} rows)", r.source, r.outcome, r.rows_written);
            }
            for s in &report.skipped {
                println!("{:<12} skipped (fresh)", s.0);
            }
            for b in &report.blocked {
                println!("{:<12} blocked (upstream failed)", b.0);
            }
        }
        Command::Embed => {
            let n = embed_launches(&cli.out_dir).await?;
            println!("embedded {n} new prose fields");
        }
    }
    Ok(())
}

/// Read the rows already written to a JSONL file (empty if it doesn't exist).
async fn read_rows(path: &Path) -> Result<Vec<Row>> {
    match tokio::fs::read_to_string(path).await {
        Ok(text) => Ok(text
            .lines()
            .filter(|l| !l.trim().is_empty())
            .map(serde_json::from_str)
            .collect::<Result<_, _>>()?),
        Err(e) if e.kind() == std::io::ErrorKind::NotFound => Ok(vec![]),
        Err(e) => Err(e.into()),
    }
}

/// A JSONL sink wrapped for idempotency, seeded with the content keys already
/// on disk so a re-run (or `backfill`) never writes a row twice.
async fn idempotent_sink(path: PathBuf) -> Result<IdempotentSink<JsonlSink>> {
    let mut keys = Vec::new();
    for row in read_rows(&path).await? {
        keys.push(row.content_key()?);
    }
    Ok(IdempotentSink::with_seen(JsonlSink::new(path), keys))
}

/// Embed the prose fields of already-loaded launches into a vector index.
/// The embed step consumes the sources' output — in a fuller build it would be
/// a DAG node depending on `spacedevs`. Seeded from the existing index so
/// already-embedded prose is never paid for twice.
async fn embed_launches(out_dir: &Path) -> Result<usize> {
    let rows = read_rows(&out_dir.join("launches.jsonl")).await?;
    if rows.is_empty() {
        anyhow::bail!("no launches to embed — run `panoptes-etl run` first");
    }

    let index_path = out_dir.join("embeddings.jsonl");
    let seen_ids = read_embedded_ids(&index_path).await?;

    let embedder = VoyageEmbedder::from_env("https://api.voyageai.com")?;
    let sink = EmbedSink::with_seen(embedder, index_path, seen_ids);
    Ok(sink.load(&rows).await?)
}

/// The ids already present in an embeddings JSONL file (empty if absent).
async fn read_embedded_ids(path: &Path) -> Result<Vec<String>> {
    match tokio::fs::read_to_string(path).await {
        Ok(text) => Ok(text
            .lines()
            .filter(|l| !l.trim().is_empty())
            .map(|l| serde_json::from_str::<EmbeddedRow>(l).map(|r| r.id))
            .collect::<Result<_, _>>()?),
        Err(e) if e.kind() == std::io::ErrorKind::NotFound => Ok(vec![]),
        Err(e) => Err(e.into()),
    }
}

/// Assemble the DAG. `incremental` sets a freshness window so `run` skips
/// recently-succeeded sources; `backfill` passes `false` to force every source.
async fn build_executor(out_dir: &Path, ledger: Ledger, incremental: bool) -> Result<Executor> {
    let refresh = incremental.then(|| Duration::hours(6));
    let mut executor = Executor::new(ledger);

    let mut celestrak = Pipeline::new(
        Box::new(CelestrakSource::new(CELESTRAK_BASE, "geo")),
        Box::new(CelestrakTransform),
        Box::new(idempotent_sink(out_dir.join("space_objects.jsonl")).await?),
    );
    celestrak.refresh_after = refresh;
    executor.add_pipeline("celestrak", celestrak);

    // Space-Track needs credentials; include it only when they are present.
    match Credentials::from_env() {
        Ok(creds) => {
            let mut spacetrack = Pipeline::new(
                Box::new(SpaceTrackSource::new(
                    SPACETRACK_BASE,
                    creds,
                    TokenBucket::new(30, 0.5),
                )),
                Box::new(SpaceTrackTransform),
                Box::new(idempotent_sink(out_dir.join("conjunctions.jsonl")).await?),
            );
            spacetrack.refresh_after = refresh;
            executor.add_pipeline("spacetrack", spacetrack);
        }
        Err(_) => eprintln!("note: SPACETRACK_USER/PASS not set — skipping spacetrack"),
    }

    let mut spacedevs = Pipeline::new(
        Box::new(SpaceDevsSource::new(SPACEDEVS_BASE)),
        Box::new(SpaceDevsTransform),
        Box::new(idempotent_sink(out_dir.join("launches.jsonl")).await?),
    );
    spacedevs.refresh_after = refresh;
    executor.add_pipeline("spacedevs", spacedevs);

    // CAPSTONE (you drive, unassisted): add your NASA pipeline here.
    // A `NasaSource` reads its key from `NASA_API_KEY` and passes it as a query
    // parameter; wire it with a transform and a `JsonlSink` exactly like the
    // three above. The Source trait is all the orchestrator needs — nothing in
    // this file's structure has to change to admit a fourth source.

    Ok(executor)
}
```

Tests: none in this file — the binary's testable surface lives in `cli.rs`.

> **Capstone (Part V, you drive).** The NASA source has no reference file: the
> commented block in `build_executor` above is the whole answer key for it. You
> write `NasaSource` + its transform in `etl-sources`, then add one `Pipeline`
> here — no structural change, because `Source` is all the orchestrator needs.

---

## Part VI — The Retrieval Arc (RAG)

The embedder boundary, a concrete Voyage-backed embedder, the embed load target,
and the brute-force cosine retriever.

<h3 id="task-embedder-trait">crates/etl-core/src/embed.rs — the Embedder trait (Part VI)</h3>

The embedding boundary — one text-to-vectors method, provider-agnostic and
mockable. Mirrors the first course's `ModelClient`.

```rust,ignore
//! The embedding boundary — a trait mirroring the first course's `ModelClient`.
//!
//! Embeddings are the definitive expensive, idempotent, content-addressed
//! workload: the same text must never be embedded twice. The trait keeps the
//! provider swappable and mockable.

use async_trait::async_trait;

use crate::error::EtlError;

/// Turns text into vectors. One vector per input, same order.
#[async_trait]
pub trait Embedder: Send + Sync {
    async fn embed(&self, texts: &[String]) -> Result<Vec<Vec<f32>>, EtlError>;
}
```

Tests: none — the trait is exercised by `vector.rs` (a counting stub) and `voyage.rs` (the concrete impl).

<h3 id="task-vector-index">crates/etl-core/src/vector.rs — the embed sink & cosine index (Part VI)</h3>

`cosine` similarity, the `EmbeddedRow` record, a brute-force `VectorIndex` with
`top_k`, `Row::prose` (only text fields are ever embedded), and `EmbedSink` — a
load target that embeds prose and appends `EmbeddedRow` JSONL, never embedding
the same text twice.

```rust,ignore
//! Brute-force semantic retrieval: an embed load target and a cosine index.
//!
//! The corpus is thousands of prose fields, not millions, so linear cosine
//! search is the correct tool — no vector database. Only *prose* is embedded;
//! the numeric entities are for structured (SQL/filter) retrieval instead.

use std::collections::HashSet;
use std::path::PathBuf;

use async_trait::async_trait;
use serde::{Deserialize, Serialize};
use tokio::fs::OpenOptions;
use tokio::io::AsyncWriteExt;
use tokio::sync::Mutex;

use crate::content::content_id;
use crate::embed::Embedder;
use crate::error::EtlError;
use crate::pipeline::{Row, Sink};

/// Cosine similarity of two vectors; 0.0 if either is the zero vector.
pub fn cosine(a: &[f32], b: &[f32]) -> f32 {
    let dot: f32 = a.iter().zip(b).map(|(x, y)| x * y).sum();
    let na = a.iter().map(|x| x * x).sum::<f32>().sqrt();
    let nb = b.iter().map(|x| x * x).sum::<f32>().sqrt();
    if na == 0.0 || nb == 0.0 {
        0.0
    } else {
        dot / (na * nb)
    }
}

/// One embedded prose field: a content-addressed id, its text, and its vector.
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct EmbeddedRow {
    pub id: String,
    pub text: String,
    pub vector: Vec<f32>,
}

/// A brute-force cosine index over embedded rows.
#[derive(Default)]
pub struct VectorIndex {
    pub rows: Vec<EmbeddedRow>,
}

impl VectorIndex {
    pub fn new(rows: Vec<EmbeddedRow>) -> Self {
        Self { rows }
    }

    pub fn from_jsonl(text: &str) -> Result<Self, EtlError> {
        let mut rows = Vec::new();
        for line in text.lines().filter(|l| !l.trim().is_empty()) {
            rows.push(serde_json::from_str(line).map_err(|e| EtlError::Parse(e.to_string()))?);
        }
        Ok(Self { rows })
    }

    /// The `k` most similar rows to `query`, highest score first.
    pub fn top_k(&self, query: &[f32], k: usize) -> Vec<(String, f32)> {
        let mut scored: Vec<(String, f32)> = self
            .rows
            .iter()
            .map(|r| (r.id.clone(), cosine(query, &r.vector)))
            .collect();
        scored.sort_by(|a, b| b.1.total_cmp(&a.1));
        scored.truncate(k);
        scored
    }
}

impl Row {
    /// The prose worth embedding, if any. Only text fields — never the numbers.
    pub fn prose(&self) -> Option<String> {
        match self {
            Row::Launch(l) if !l.description.trim().is_empty() => Some(l.description.clone()),
            _ => None,
        }
    }
}

/// A load target that embeds prose fields and appends `EmbeddedRow` JSONL,
/// never embedding the same text twice.
pub struct EmbedSink<E: Embedder> {
    embedder: E,
    path: PathBuf,
    seen: Mutex<HashSet<String>>,
}

impl<E: Embedder> EmbedSink<E> {
    pub fn new(embedder: E, path: impl Into<PathBuf>) -> Self {
        Self {
            embedder,
            path: path.into(),
            seen: Mutex::new(HashSet::new()),
        }
    }

    /// Pre-load already-embedded ids so re-runs across processes never re-embed
    /// the same prose (embedding is expensive — this is the whole point).
    pub fn with_seen(embedder: E, path: impl Into<PathBuf>, ids: impl IntoIterator<Item = String>) -> Self {
        Self {
            embedder,
            path: path.into(),
            seen: Mutex::new(ids.into_iter().collect()),
        }
    }
}

#[async_trait]
impl<E: Embedder> Sink for EmbedSink<E> {
    async fn load(&self, rows: &[Row]) -> Result<usize, EtlError> {
        // Collect the fresh prose to embed, keyed by content id.
        let mut fresh: Vec<(String, String)> = Vec::new();
        {
            let mut seen = self.seen.lock().await;
            for row in rows {
                if let Some(text) = row.prose() {
                    let id = content_id("embed", &text);
                    if seen.insert(id.clone()) {
                        fresh.push((id, text));
                    }
                }
            }
        }
        if fresh.is_empty() {
            return Ok(0);
        }

        let texts: Vec<String> = fresh.iter().map(|(_, t)| t.clone()).collect();
        let vectors = self.embedder.embed(&texts).await?;
        if vectors.len() != texts.len() {
            return Err(EtlError::Parse(format!(
                "embedder returned {} vectors for {} texts",
                vectors.len(),
                texts.len()
            )));
        }

        let mut file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(&self.path)
            .await?;
        let mut buf = String::new();
        for ((id, text), vector) in fresh.into_iter().zip(vectors) {
            let row = EmbeddedRow { id, text, vector };
            buf.push_str(&serde_json::to_string(&row).map_err(|e| EtlError::Parse(e.to_string()))?);
            buf.push('\n');
        }
        let written = buf.lines().count();
        file.write_all(buf.as_bytes()).await?;
        Ok(written)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::domain::{Conjunction, Launch};
    use crate::ids::NoradId;
    use std::sync::atomic::{AtomicUsize, Ordering};
    use std::sync::Arc;

    #[test]
    fn cosine_of_identical_is_one_orthogonal_is_zero() {
        assert!((cosine(&[1.0, 2.0, 3.0], &[1.0, 2.0, 3.0]) - 1.0).abs() < 1e-6);
        assert!(cosine(&[1.0, 0.0], &[0.0, 1.0]).abs() < 1e-6);
    }

    #[test]
    fn top_k_ranks_by_similarity() {
        let index = VectorIndex::new(vec![
            EmbeddedRow { id: "near".into(), text: "".into(), vector: vec![1.0, 0.1] },
            EmbeddedRow { id: "far".into(), text: "".into(), vector: vec![-1.0, 0.0] },
            EmbeddedRow { id: "mid".into(), text: "".into(), vector: vec![0.5, 0.5] },
        ]);
        let hits = index.top_k(&[1.0, 0.0], 2);
        assert_eq!(hits.len(), 2);
        assert_eq!(hits[0].0, "near");
        assert!(hits[0].1 >= hits[1].1);
    }

    /// Counts how many texts were embedded and returns fixed-width vectors.
    struct CountingEmbedder {
        calls: Arc<AtomicUsize>,
    }
    #[async_trait]
    impl Embedder for CountingEmbedder {
        async fn embed(&self, texts: &[String]) -> Result<Vec<Vec<f32>>, EtlError> {
            self.calls.fetch_add(texts.len(), Ordering::SeqCst);
            Ok(texts.iter().map(|t| vec![t.len() as f32, 1.0]).collect())
        }
    }

    fn launch(desc: &str) -> Row {
        Row::Launch(Launch {
            id: "l".into(),
            name: "L".into(),
            net: "2026-07-10T12:00:00Z".parse().unwrap(),
            provider: "SpaceX".into(),
            payloads: vec![],
            description: desc.into(),
        })
    }

    fn temp_path(tag: &str) -> PathBuf {
        let dir = std::env::temp_dir().join(format!("etl-embed-{tag}-{}", std::process::id()));
        std::fs::create_dir_all(&dir).unwrap();
        let path = dir.join("vectors.jsonl");
        let _ = std::fs::remove_file(&path);
        path
    }

    #[tokio::test]
    async fn embed_sink_only_touches_prose() {
        let calls = Arc::new(AtomicUsize::new(0));
        let sink = EmbedSink::new(
            CountingEmbedder { calls: calls.clone() },
            temp_path("prose"),
        );
        let rows = vec![
            launch("A batch of broadband satellites."),
            // A conjunction has no prose — it must not be embedded.
            Row::Conjunction(Conjunction {
                id: "c".into(),
                tca: "2026-07-20T12:00:00Z".parse().unwrap(),
                miss_distance_km: 0.5,
                collision_probability: 0.001,
                primary: NoradId(1),
                secondary: NoradId(2),
            }),
        ];
        let n = sink.load(&rows).await.unwrap();
        assert_eq!(n, 1);
        assert_eq!(calls.load(Ordering::SeqCst), 1);
    }

    #[tokio::test]
    async fn cached_text_not_re_embedded() {
        let calls = Arc::new(AtomicUsize::new(0));
        let sink = EmbedSink::new(
            CountingEmbedder { calls: calls.clone() },
            temp_path("cache"),
        );
        let rows = vec![launch("same text"), launch("same text")];
        let n = sink.load(&rows).await.unwrap();
        assert_eq!(n, 1); // identical prose embedded once
        assert_eq!(calls.load(Ordering::SeqCst), 1);
        // A second load of the same text embeds nothing new.
        assert_eq!(sink.load(&rows).await.unwrap(), 0);
        assert_eq!(calls.load(Ordering::SeqCst), 1);
    }
}
```

Tests: `cosine_of_identical_is_one_orthogonal_is_zero`, `top_k_ranks_by_similarity`, `embed_sink_only_touches_prose`, `cached_text_not_re_embedded`.

<h3 id="task-voyage-embedder">crates/etl-sources/src/voyage.rs — the Voyage embedder (Part VI)</h3>

A concrete `Embedder` backed by the Voyage AI embeddings API — injectable base
URL, bearer key, request/response pair — structurally identical to the first
course's `ModelClient`, and mocked with wiremock.

```rust,ignore
//! A concrete [`Embedder`] backed by the Voyage AI embeddings API.
//!
//! Structurally identical to the first course's Anthropic `ModelClient`: an
//! injectable base URL, a bearer key, a request/response pair — and it is
//! mocked with `wiremock` in tests, never hitting the live API.

use async_trait::async_trait;
use etl_core::{Embedder, EtlError};
use serde::{Deserialize, Serialize};

pub struct VoyageEmbedder {
    pub base_url: String,
    pub api_key: String,
    pub model: String,
}

impl VoyageEmbedder {
    pub fn new(base_url: impl Into<String>, api_key: impl Into<String>) -> Self {
        Self {
            base_url: base_url.into(),
            api_key: api_key.into(),
            model: "voyage-3".to_string(),
        }
    }

    /// Read the key from `VOYAGE_API_KEY`; missing ⇒ auth error.
    pub fn from_env(base_url: impl Into<String>) -> Result<Self, EtlError> {
        let key = std::env::var("VOYAGE_API_KEY")
            .map_err(|_| EtlError::Auth("VOYAGE_API_KEY not set".into()))?;
        Ok(Self::new(base_url, key))
    }
}

#[derive(Serialize)]
struct EmbedRequest<'a> {
    input: &'a [String],
    model: &'a str,
}

#[derive(Deserialize)]
struct EmbedResponse {
    data: Vec<EmbedData>,
}

#[derive(Deserialize)]
struct EmbedData {
    embedding: Vec<f32>,
}

#[async_trait]
impl Embedder for VoyageEmbedder {
    async fn embed(&self, texts: &[String]) -> Result<Vec<Vec<f32>>, EtlError> {
        if texts.is_empty() {
            return Ok(vec![]);
        }
        let resp = reqwest::Client::new()
            .post(format!("{}/v1/embeddings", self.base_url.trim_end_matches('/')))
            .bearer_auth(&self.api_key)
            .json(&EmbedRequest {
                input: texts,
                model: &self.model,
            })
            .send()
            .await
            .map_err(|e| EtlError::Network(e.to_string()))?;
        if !resp.status().is_success() {
            return Err(crate::http::classify_status(resp.status(), resp.headers()));
        }
        let parsed: EmbedResponse =
            resp.json().await.map_err(|e| EtlError::Parse(e.to_string()))?;
        Ok(parsed.data.into_iter().map(|d| d.embedding).collect())
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use pretty_assertions::assert_eq;
    use wiremock::matchers::{header, method, path};
    use wiremock::{Mock, MockServer, ResponseTemplate};

    #[tokio::test]
    async fn embeds_batch_of_texts_with_bearer_auth() {
        let server = MockServer::start().await;
        Mock::given(method("POST"))
            .and(path("/v1/embeddings"))
            .and(header("authorization", "Bearer test-key"))
            .respond_with(ResponseTemplate::new(200).set_body_string(
                r#"{"data":[{"embedding":[0.1,0.2]},{"embedding":[0.3,0.4]}]}"#,
            ))
            .expect(1)
            .mount(&server)
            .await;

        let embedder = VoyageEmbedder::new(server.uri(), "test-key");
        let vectors = embedder
            .embed(&["hello".to_string(), "world".to_string()])
            .await
            .unwrap();
        assert_eq!(vectors, vec![vec![0.1, 0.2], vec![0.3, 0.4]]);
    }

    #[tokio::test]
    async fn empty_input_makes_no_request() {
        // No mock mounted: if embed() called out, it would error.
        let embedder = VoyageEmbedder::new("http://127.0.0.1:1", "k");
        assert_eq!(embedder.embed(&[]).await.unwrap(), Vec::<Vec<f32>>::new());
    }
}
```

Tests: `embeds_batch_of_texts_with_bearer_auth`, `empty_input_makes_no_request`.

---

## Verifying the whole thing

Run these from the workspace root. All three must pass exactly as shown.

**1. The full test suite — 65 tests, all passing:**

```console
$ cargo test
```

The per-crate breakdown:

| Crate | Tests |
|---|---|
| `etl-core` | 29 |
| `etl-sources` | 26 |
| `etl-orchestrate` | 7 |
| `panoptes-etl` | 3 |
| **Total** | **65** |

Every live API (CelesTrak, Space-Track, TheSpaceDevs, Voyage) is mocked with
`wiremock`; no test touches the network. The time-based tests (`limiter`,
`retry`) run under paused tokio time, so they are deterministic.

**2. Clippy, clean across all targets:**

```console
$ cargo clippy --all-targets
```

Produces no warnings.

**3. The dependency-inversion proof — `etl-orchestrate` depends on `etl-core` only:**

```console
$ cargo tree -p etl-orchestrate
```

Among the workspace crates, the only one that appears is `etl-core` (alongside
`chrono` and their external transitive deps). `etl-sources` and `panoptes-etl`
are **absent** — the orchestrator schedules `dyn Source` and cannot name a
concrete source. Only the `panoptes-etl` binary names them. That is the
dependency arrow pointing toward `etl-core`, enforced by the crate graph rather
than by discipline.

If all three commands come back the way this page describes them, your build
matches the reference exactly.
