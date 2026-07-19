# Concept: Dependency Inversion — the Executor Knows Only the Trait

> **Kind:** Concept. **The property this arc proves:** the orchestrator schedules `dyn Source` / `dyn Transform` / `dyn Sink` and depends on `etl-core` *only*. It has never heard of CelesTrak. The crate boundary is the proof, and `cargo tree` is how you read it.

## The idea

Here is the arrangement you would reach for if you were not being careful. The executor needs to run the CelesTrak pipeline, so it imports `CelestrakSource`. It needs Space-Track too, so it imports `SpaceTrackSource`. Now `etl-orchestrate` depends on `etl-sources`, and every time you add a source you edit the orchestrator. The scheduler — the piece that should be the most stable code in the system — churns every time a *leaf* changes. That is a dependency pointing the wrong way: the high-level policy (how to schedule) depends on the low-level detail (which sources exist).

**Dependency inversion** flips it. Both the executor and the concrete sources depend on an *abstraction* — the `Source` trait, which lives in `etl-core` — and neither depends on the other. The executor holds a `Box<dyn Source>` and calls `.extract()` through the vtable. It does not know, and cannot know, whether the box contains a `CelestrakSource` or a NASA source you will write next week. The arrows both point *inward*, at the trait:

```text
   BEFORE (dependency points down at the detail):

     etl-orchestrate ──depends on──▶ etl-sources ──▶ CelestrakSource
        (policy)                       (detail)
     every new source edits the orchestrator ✗


   AFTER (both depend on the abstraction in the middle):

     etl-orchestrate ──▶ Source trait (etl-core) ◀── etl-sources
        (policy)            (abstraction)              (detail)
     a new source touches neither the trait nor the orchestrator ✓
```

The executor's own doc comment states the contract outright: it "schedules `dyn Source` / `dyn Transform` / `dyn Sink` and has no knowledge of any concrete source — that separation is enforced by the crate boundary, not by convention."

## The crate boundary *is* the enforcement

"By convention" would mean: we all agree not to import concrete sources into the orchestrator, and we hope nobody does. That is not a guarantee — it is a coding-review habit that fails the first time someone is in a hurry.

The Panoptes workspace makes it a *compile-time* guarantee instead. `etl-orchestrate`'s `Cargo.toml` simply does not list `etl-sources` as a dependency:

```toml
[package]
name = "etl-orchestrate"
edition = "2024"

# Depends on etl-core ONLY. It schedules `dyn Source` and must never know a
# concrete source exists — that constraint is the dependency-inversion lesson.
[dependencies]
etl-core = { path = "../etl-core" }
chrono = { workspace = true }
```

Because the dependency is not declared, `CelestrakSource` is *not in scope* inside `etl-orchestrate`. If you tried to write `use etl_sources::CelestrakSource;` in `executor.rs`, it would not compile — unresolved import, missing crate. The wrong dependency is not discouraged; it is *impossible*. The layering is load-bearing structure, checked by `cargo` on every build.

## Reading the proof with `cargo tree`

You do not have to take the manifest's word for it. `cargo tree -p etl-orchestrate` prints the crate's entire dependency closure — everything it pulls in, transitively. If `etl-sources` were anywhere in that tree, it would appear. Here is the real output, trimmed to the two direct dependencies and their headline sub-deps:

```text
$ cargo tree -p etl-orchestrate
etl-orchestrate v0.1.0
├── chrono v0.4.45
│   └── ... (time math for the freshness window)
└── etl-core v0.1.0
    ├── async-trait v0.1.91 (proc-macro)
    ├── chrono v0.4.45
    ├── serde v1.0.229
    ├── serde_json v1.0.150
    ├── sha2 v0.10.9
    ├── thiserror v2.0.19
    └── tokio v1.53.0
[dev-dependencies]
├── async-trait
├── pretty_assertions
└── tokio
```

Two direct dependencies: `chrono` (for the freshness math) and `etl-core` (for the traits, `Ledger`, `RunRecord`, `with_retry`). **`etl-sources` is not present — not directly, not transitively.** The orchestrator's world is the abstraction layer and nothing below it. That absence is the whole lesson, and it is a one-command check you can put in CI: grep the tree for `etl-sources` and fail the build if it ever shows up.

<div class="callout">
<span class="callout-label">Where the concrete sources actually get named</span>
Somebody has to say "the CelesTrak source is a <code>CelestrakSource</code>." That somebody is the <strong>binary</strong>, <code>panoptes-etl</code> — the top of the dependency graph, which depends on both <code>etl-orchestrate</code> and <code>etl-sources</code> and wires them together in <code>main.rs</code>. The binary is the <em>only</em> crate in the workspace that names a concrete source. Detail-knowledge is concentrated at the single point that assembles the program, and kept out of every reusable layer beneath it.
</div>

## A runnable toy: a scheduler that never learns a concrete type

The trait objects in the real code pull in `async-trait` and `tokio`, so they will not run on the playground. Here is the *same shape* reduced to synchronous `std` — a job scheduler that owns a list of type-erased jobs and runs them in order, never naming `Backup` or `Report`:

```rust
// The scheduler's whole world: one trait. It never learns a concrete type.
trait Job {
    fn name(&self) -> &str;
    fn run(&self) -> usize; // returns "units of work done"
}

// A scheduler that owns a list of type-erased jobs and runs them in order.
// Nothing here mentions Backup, Report, or any concrete job — only `dyn Job`.
struct Scheduler {
    jobs: Vec<Box<dyn Job>>,
}

impl Scheduler {
    fn new() -> Self {
        Scheduler { jobs: Vec::new() }
    }
    fn add(&mut self, job: Box<dyn Job>) {
        self.jobs.push(job);
    }
    fn run_all(&self) -> usize {
        let mut total = 0;
        for job in &self.jobs {
            let n = job.run();
            println!("{:<8} did {n} units", job.name());
            total += n;
        }
        total
    }
}

// Concrete jobs — the "leaf" layer. These would live in a *different* crate
// that depends on the scheduler's, never the other way around.
struct Backup;
impl Job for Backup {
    fn name(&self) -> &str { "backup" }
    fn run(&self) -> usize { 12 }
}

struct Report;
impl Job for Report {
    fn name(&self) -> &str { "report" }
    fn run(&self) -> usize { 3 }
}

fn main() {
    let mut sched = Scheduler::new();
    // A fourth, fifth, hundredth job type would need zero scheduler changes:
    // if it implements Job, `add` accepts it.
    sched.add(Box::new(Backup));
    sched.add(Box::new(Report));
    let total = sched.run_all();
    println!("total: {total}");
}
```

```text
backup   did 12 units
report   did 3 units
total: 15
```

`Scheduler` names `Job` and nothing else. `Backup` and `Report` sit at the leaf layer — in the real workspace they are a *different crate*, `etl-sources`, and the dependency goes leaf → trait, never scheduler → leaf. Swap `Report` for a `NasaSource` and `Scheduler` does not change one character. That is the payoff you will feel in the capstone: a fourth source drops into the DAG with zero orchestrator edits, precisely because the orchestrator was built to know only the trait.

<div class="callout trap">
<span class="callout-label">Object safety is the fine print on all of this</span>
The whole scheme rests on <code>Box&lt;dyn Source&gt;</code> being a legal type — i.e. on <code>Source</code> being <strong>dyn-compatible</strong>. The extraction arc showed how one generic method (<code>fn extract_as&lt;T&gt;(&amp;self)</code>) silently makes a trait un-boxable and the error lands far away, at the <code>Box&lt;dyn …&gt;</code> line. Dependency inversion and object safety are the same discipline viewed from two angles: you can only invert the dependency onto a trait that a vtable can actually hold, which is exactly why <code>Source</code>, <code>Transform</code>, and <code>Sink</code> keep their plain, generic-free signatures.
</div>

## Why this is the real payoff of the whole course

Every arc has been quietly setting this up. Part II defined the traits and made them dyn-compatible. Parts III and IV built three concrete implementations behind those traits. This arc collects the reward: because the executor depends only on the abstraction, it schedules all three sources uniformly, and it will schedule the fourth — the NASA source you write in the capstone — without knowing it exists. The system got a new capability by *adding* a leaf, not by *editing* the core. That is the difference between a codebase that ossifies as it grows and one that stays soft where it needs to. The crate graph, not good intentions, is what keeps it that way.

## Questions to lock

1. In the "before" arrangement, which way does the dependency arrow point, and why does that make the orchestrator churn every time a source is added?
2. What is the abstraction that both the executor and the concrete sources depend on, and in which crate does it live?
3. Why is "we agree not to import concrete sources into the orchestrator" weaker than what the workspace actually does — and what makes the wrong import *impossible* rather than merely discouraged?
4. What would `cargo tree -p etl-orchestrate` show if the layering were broken, and what does its *absence* from the tree prove?
5. Which crate is the only one allowed to name a concrete source, and why is concentrating that knowledge at the top of the graph the right call?

The graded quiz for this arc is at the end of the part, on the [Concept-Check: Orchestration](./quiz.md) page.
