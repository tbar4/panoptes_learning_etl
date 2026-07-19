# Build: The Task Graph + Topological Execution

> **Maps to:** Task 4.1. **Kind:** Build — spec and test names only. No implementation.

## Objective

Create the `etl-orchestrate` crate and its first module: a typed task graph, `Dag`, with a deterministic topological sort. This is the pure-data foundation the executor stands on — no async, no I/O, no traits from `etl-core` beyond the error type. By the end, `topological_order()` turns a set of tasks and dependencies into a stable run order, and rejects any graph that contains a cycle.

## Scaffold

**Create the crate:** `crates/etl-orchestrate/Cargo.toml`, `src/lib.rs`, `src/graph.rs`. Add `"crates/etl-orchestrate"` to the workspace `members`.

**`crates/etl-orchestrate/Cargo.toml`** — this is the crate where the dependency-inversion lesson is enforced by the manifest, so the dependency list is deliberately short:

```toml
[dependencies]
etl-core = { path = "../etl-core" }   # for EtlError only, in this module
chrono = { workspace = true }         # the executor module needs it next

[dev-dependencies]
pretty_assertions = { workspace = true }
tokio = { workspace = true }
async-trait = { workspace = true }
```

> There is **no** `etl-sources` dependency, and there never will be. That absence is the point of the whole arc — see [Concept: Dependency Inversion](./concept-dependency-inversion.md). Prove it after this build with `cargo tree -p etl-orchestrate`.

**Expected result:** `cargo test -p etl-orchestrate graph` → the three graph tests green.

## The spec (givens)

- **`TaskId(pub String)`** — a newtype over the task's name. Derive `Clone, PartialEq, Eq, Hash, PartialOrd, Ord` — the `Ord` is what lets the ready-set sort. A `TaskId::new(impl Into<String>)` constructor.
- **`Dag`** — holds `nodes: HashSet<TaskId>` and `prereqs: HashMap<TaskId, Vec<TaskId>>` (task → its prerequisites). Derive `Default`.
  - `add_task(&mut self, id: impl Into<String>) -> TaskId` — register a node with an empty prereq list, return its id.
  - `add_dep(&mut self, task: &TaskId, depends_on: &TaskId)` — record that `task` depends on `depends_on`. Must insert *both* ids as nodes (so an edge can introduce a node that was never `add_task`-ed) and push `depends_on` onto `task`'s prereq list.
  - `prerequisites(&self, task: &TaskId) -> &[TaskId]` — the direct prereqs, or an empty slice for an unknown task.
  - `topological_order(&self) -> Result<Vec<TaskId>, EtlError>` — Kahn's algorithm with a **`BTreeSet` ready-set** so ties break deterministically (alphabetically). A cycle (ordered count ≠ node count) returns `EtlError::Parse("cycle detected: …")`.

The algorithm is spelled out step by step, with a runnable `std`-only mirror, on [Concept: Modeling a Pipeline as a Typed DAG](./concept-dag.md). This build is that concept made real over `TaskId` and `EtlError`.

## Concepts exercised

- Kahn's topological sort; indegree bookkeeping and the reverse `dependents` index.
- Deterministic iteration with `BTreeSet` (vs the run-to-run wobble of `HashSet`).
- Cycle detection as a length-mismatch invariant, not a separate search.
- A newtype (`TaskId`) whose derived `Ord` is what makes the sort stable.

## The build loop (you drive)

1. **`TaskId` + `Dag` skeleton.** Define the newtype with its derives and the two fields; write `add_task` / `add_dep` / `prerequisites`. Nothing to run yet.
2. **`linear_chain_orders_correctly`.** Three tasks `a → b → c`; assert the order is exactly `["a", "b", "c"]`. Implement enough of `topological_order` to pass it.
3. **`diamond_orders_deps_before_dependents`.** `a` feeds `b` and `c`; `d` needs both. **Predict first:** with a sorted ready-set, do you get `["a", "b", "c", "d"]` or `["a", "c", "b", "d"]`? Decide from the tie-break rule *before* running.
4. **`cycle_is_rejected`.** `a` depends on `b` and `b` depends on `a`; assert `topological_order()` is `Err(EtlError::Parse(_))`. Confirm the length check is what fires.
5. **Run green, prove the boundary:** `cargo tree -p etl-orchestrate | grep etl-sources` returns nothing. Commit.

<div class="callout predict">
<span class="callout-label">Predict first</span>
Before you run the diamond test: the ready-set is a <code>BTreeSet&lt;TaskId&gt;</code>, and <code>TaskId</code> derives <code>Ord</code> from its inner <code>String</code>. When <code>b</code> and <code>c</code> are both ready, which comes out of <code>ready.iter().next()</code> first? Write the expected <code>Vec</code> down, then let the test confirm it. If you had reached for a <code>Vec</code> or <code>HashSet</code> instead, what could go wrong across repeated runs?
</div>

## Done when

`cargo test -p etl-orchestrate graph` is green and `cargo tree -p etl-orchestrate` shows only `etl-core` and `chrono` — no concrete-source crate anywhere in the closure. You now have a deterministic, cycle-safe run order. The executor is next: it walks this order and adds the two behaviors a bare sort lacks — freshness-skip and failure-gating.
