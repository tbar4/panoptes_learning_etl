# Concept: Modeling a Pipeline as a Typed DAG

> **Kind:** Concept. **Seam installed this arc:** the task graph — pipelines become nodes, dependencies become edges, and execution order becomes a computed property of the graph rather than a hand-written sequence.

## The idea

You now have three concrete pipelines — CelesTrak, Space-Track, TheSpaceDevs — each a `Source` + `Transform` + `Sink`. Left to yourself, you would run them by writing them out in an order: extract this, then that, then the third. That works right up until one pipeline needs another's output first, or you add a fourth, or you want an embedding step that must run *after* the launches land. The hand-written order becomes a thing you maintain by hand, and hand-maintained orderings rot.

The fix is to stop writing the order down and start **computing** it. Model the pipelines as a **directed acyclic graph** (DAG):

- Each pipeline is a **node** (a `TaskId`).
- "B needs A's output first" is a directed **edge** — a prerequisite.
- "Acyclic" is the promise that the prerequisites never loop back on themselves. If they did, there would be no valid order to run them in, and that is a bug you want caught *before* anything runs, not a deadlock you discover at 3am.

Given that graph, a **topological sort** produces a linear run order in which every node appears after all of its prerequisites. You declare the dependencies; the algorithm derives the order. Add a fourth pipeline with one `add_dep` call and the order recomputes itself.

```text
        ┌─────────┐
        │  sift   │            a topological order is any linearization
        └────┬────┘            where every arrow points "backwards" in
          ┌──┴───┐             the list — a prerequisite is always to
          ▼      ▼             the left of the task that needs it:
     ┌────────┐ ┌────────┐
     │ grease │ │ whisk  │        sift → grease → whisk → bake
     └────┬───┘ └───┬────┘
          └────┬────┘
               ▼
          ┌────────┐
          │  bake  │
          └────────┘
```

## Kahn's algorithm, and why the shipped code uses a *sorted* ready-set

The topological sort in `etl-orchestrate` is **Kahn's algorithm**. The mental model is small:

1. Compute each node's **indegree** — how many prerequisites it has that have not run yet.
2. Every node with indegree `0` is **ready** (nothing is blocking it). Put those in a ready-set.
3. Pop a ready node, append it to the output order, and decrement the indegree of everything that depended on it. Any of those that just hit `0` is now ready — add it.
4. Repeat until the ready-set is empty.

There is a design decision hiding in step 3: *which* ready node do you pop when several are ready at once? In the diamond above, once `sift` runs, both `grease` and `whisk` become ready simultaneously. A plain queue or a `HashSet` would pop them in whatever order the hash happened to land — which can differ run to run, and differ between machines. That is a nightmare to test and a nightmare to debug: a pipeline order that changes for no visible reason.

The shipped code makes the tie-break **deterministic** by keeping the ready-set in a `BTreeSet`, which is always sorted. When several tasks are ready, the alphabetically-first one runs next. Same graph, same order, every single time — on your laptop, in CI, on a colleague's machine.

> This example is pure `std` — `HashMap`, `HashSet`, `BTreeSet`, `Vec` — so it runs on the Rust playground. It is the *same algorithm* the shipped `Dag::topological_order` uses, on a cake-baking domain so it cannot be mistaken for the answer key.

```rust
use std::collections::{BTreeSet, HashMap, HashSet};

/// A tiny task graph over string ids. Edges record prerequisites:
/// `add_dep("frost", "bake")` means "frost depends on bake" — bake runs first.
#[derive(Default)]
struct Dag {
    nodes: HashSet<String>,
    prereqs: HashMap<String, Vec<String>>,
}

impl Dag {
    fn add_task(&mut self, id: &str) {
        self.nodes.insert(id.to_string());
        self.prereqs.entry(id.to_string()).or_default();
    }

    fn add_dep(&mut self, task: &str, depends_on: &str) {
        self.nodes.insert(task.to_string());
        self.nodes.insert(depends_on.to_string());
        self.prereqs.entry(depends_on.to_string()).or_default();
        self.prereqs
            .entry(task.to_string())
            .or_default()
            .push(depends_on.to_string());
    }

    /// Kahn's algorithm with a *sorted* ready-set (BTreeSet), so ties break
    /// alphabetically and the order is identical on every run.
    fn topological_order(&self) -> Result<Vec<String>, String> {
        // indegree = number of prerequisites still unmet.
        let mut indegree: HashMap<String, usize> = self
            .nodes
            .iter()
            .map(|n| (n.clone(), self.prereqs.get(n).map_or(0, |p| p.len())))
            .collect();

        // Reverse index: prerequisite -> the tasks waiting on it.
        let mut dependents: HashMap<String, Vec<String>> = HashMap::new();
        for (task, prereqs) in &self.prereqs {
            for p in prereqs {
                dependents.entry(p.clone()).or_default().push(task.clone());
            }
        }

        // Everything with no prerequisites is ready now.
        let mut ready: BTreeSet<String> = indegree
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

        // Anything left unordered sits in a cycle: its indegree never hit zero.
        if order.len() != self.nodes.len() {
            return Err(format!(
                "cycle detected: ordered {} of {} tasks",
                order.len(),
                self.nodes.len()
            ));
        }
        Ok(order)
    }
}

fn main() {
    // A diamond: `sift` feeds both `whisk` and `grease`; `bake` needs both.
    let mut dag = Dag::default();
    dag.add_task("sift");
    dag.add_task("whisk");
    dag.add_task("grease");
    dag.add_task("bake");
    dag.add_dep("whisk", "sift");
    dag.add_dep("grease", "sift");
    dag.add_dep("bake", "whisk");
    dag.add_dep("bake", "grease");

    let order = dag.topological_order().unwrap();
    println!("{order:?}");

    // Run it again — the sorted ready-set means the answer never wobbles.
    let again = dag.topological_order().unwrap();
    println!("stable: {}", order == again);
}
```

```text
["sift", "grease", "whisk", "bake"]
stable: true
```

Read the output carefully. `sift` is first because it is the only node with no prerequisites. Then `grease` beats `whisk` — not because it was added first (it was not), but because `"grease" < "whisk"` alphabetically and the `BTreeSet` always hands back its smallest element. `bake` is last because its indegree only reaches `0` after *both* `grease` and `whisk` have run. And the second call produces the byte-identical order: determinism here is not luck, it is the data structure.

<div class="callout">
<span class="callout-label">Edge direction is a decision you make once</span>
The code stores <code>prereqs</code>: <em>task → the things it depends on</em>. Kahn's algorithm needs the reverse — <em>prerequisite → the things waiting on it</em> — to know whose indegree to decrement, so <code>topological_order</code> builds a <code>dependents</code> index at the top. Pick one direction as the stored truth and derive the other; storing both and keeping them in sync by hand is how graphs drift into inconsistency.
</div>

## Cycle detection falls out for free

Notice that `topological_order` never explicitly searches for a cycle. It does not need to. Kahn's algorithm can only ever emit a node once that node's indegree has reached `0` — once all its prerequisites have already been emitted. If some nodes depend on each other in a loop, none of them can ever reach indegree `0`, so none of them are ever emitted. The loop runs dry, and the output order is *shorter* than the node count.

That length mismatch **is** the cycle detector: `if order.len() != self.nodes.len()`. The shipped code turns it into an `EtlError::Parse` (a cycle is, in effect, a malformed pipeline definition); the toy returns a `String` error, but the check is identical:

```rust
# use std::collections::{BTreeSet, HashMap, HashSet};
# #[derive(Default)]
# struct Dag { nodes: HashSet<String>, prereqs: HashMap<String, Vec<String>> }
# impl Dag {
#     fn add_dep(&mut self, task: &str, depends_on: &str) {
#         self.nodes.insert(task.to_string());
#         self.nodes.insert(depends_on.to_string());
#         self.prereqs.entry(depends_on.to_string()).or_default();
#         self.prereqs.entry(task.to_string()).or_default().push(depends_on.to_string());
#     }
#     fn topological_order(&self) -> Result<Vec<String>, String> {
#         let mut indegree: HashMap<String, usize> =
#             self.nodes.iter().map(|n| (n.clone(), self.prereqs.get(n).map_or(0, |p| p.len()))).collect();
#         let mut dependents: HashMap<String, Vec<String>> = HashMap::new();
#         for (task, prereqs) in &self.prereqs {
#             for p in prereqs { dependents.entry(p.clone()).or_default().push(task.clone()); }
#         }
#         let mut ready: BTreeSet<String> =
#             indegree.iter().filter(|(_, d)| **d == 0).map(|(t, _)| t.clone()).collect();
#         let mut order = Vec::new();
#         while let Some(next) = ready.iter().next().cloned() {
#             ready.remove(&next);
#             if let Some(children) = dependents.get(&next) {
#                 for child in children {
#                     let d = indegree.get_mut(child).expect("child in indegree");
#                     *d -= 1;
#                     if *d == 0 { ready.insert(child.clone()); }
#                 }
#             }
#             order.push(next);
#         }
#         if order.len() != self.nodes.len() {
#             return Err(format!("cycle detected: ordered {} of {} tasks", order.len(), self.nodes.len()));
#         }
#         Ok(order)
#     }
# }
fn main() {
    // a depends on b, and b depends on a: neither can ever be "ready".
    let mut dag = Dag::default();
    dag.add_dep("a", "b");
    dag.add_dep("b", "a");
    match dag.topological_order() {
        Ok(order) => println!("ordered: {order:?}"),
        Err(e) => println!("rejected: {e}"),
    }
}
```

```text
rejected: cycle detected: ordered 0 of 2 tasks
```

Zero of two: `a` waits on `b`, `b` waits on `a`, the ready-set starts *empty*, the `while` loop never executes, and the count check fires immediately. A malformed pipeline graph is rejected before a single `extract` is attempted — the "acyclic" in DAG is enforced, not assumed.

<div class="callout trap">
<span class="callout-label">A self-edge is a cycle too</span>
<code>add_dep("report", "report")</code> — a task that depends on itself — is a one-node cycle. Its indegree starts at 1 and nothing will ever decrement it, so it is never emitted and the length check rejects the whole graph. This is easy to introduce by accident when task ids are computed from strings; the count-mismatch guard catches it the same way it catches a two-node loop, with no special case.
</div>

## Why this is the seam that unlocks the arc

The three earlier arcs each installed a seam — the `Source`/`Transform`/`Sink` traits, the run ledger, the retry middleware. The DAG is the seam that lets those pieces be *composed without a human choosing the order*. Once "what depends on what" is data in a graph rather than lines in a `main` function, three things become free that were previously manual:

- **Order** is computed, so adding a node cannot silently run it too early.
- **Cycles** are rejected up front, so an impossible pipeline never half-runs.
- **Determinism** means a run order you can write a test against — which is exactly what the next build page does.

The executor builds directly on top of this: it takes the `topological_order()` and walks it, but with two twists a bare sort does not have — it can *skip* a node that is still fresh, and it can *gate* a node whose prerequisite failed. Both of those need the order first. This is the foundation they stand on.

## Questions to lock

1. What do nodes and edges represent in the pipeline DAG, and what does "acyclic" guarantee that makes a topological order possible at all?
2. In Kahn's algorithm, what is a node's *indegree*, and what event lets a node move into the ready-set?
3. Why does the shipped code use a `BTreeSet` for the ready-set instead of a plain queue or `HashSet` — what property would be lost otherwise?
4. How does `topological_order` detect a cycle *without* ever explicitly searching for one? What single comparison is the whole detector?
5. The code stores `prereqs` (task → its prerequisites) but builds a `dependents` index inside the sort. Why does Kahn's algorithm need the reverse mapping?

The graded quiz for this arc is at the end of the part, on the [Concept-Check: Orchestration](./quiz.md) page.
