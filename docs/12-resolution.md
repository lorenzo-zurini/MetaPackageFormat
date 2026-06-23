# 12 · Dependency resolution

Resolving a launchable (or a runner) means turning its `PARENTS` graph into an **ordered list of nodes** — the *closure*
— after deciding which optional nodes are on and which mutually-exclusive ones win. The order is the overlay priority
(chapter 13). This chapter specifies the algorithm exactly; an implementation must reproduce its observable results.

## 12.1 What resolution produces

`ResolveNodeOrder(graph, launchNodeId, toggles) → [nodeId, …]`

- Input: the graph, the node to resolve, and a `toggles` map (`nodeId → bool`) of explicit user choices for optional
  nodes.
- Output: the topologically-ordered list of enabled nodes — **parents before children**, so the launch node is **last**
  (highest overlay priority). Runners are *not* excluded by this function (a runner's build is resolved by calling it on
  the runner node); the launch pipeline filters runner nodes out of the *content* closure separately.
- Also reports any `PARENTS` id missing from the graph (for diagnostics).

## 12.2 The two phases

Resolution is a breadth-first *enable* pass followed by a depth-first *order* pass.

```
function ResolveNodeOrder(graph, launchId, toggles):
    if launchId not in graph: report missing; return []

    # ---- Phase 1: determine the ENABLED set (BFS over PARENTS) ----
    enabled = { launchId }
    kept    = { launchId }                         # for symmetric EXCLUDE (first-kept wins)
    frontier = [ launchId ]
    while frontier not empty:
        cur = frontier.popFront()
        node = graph[cur]
        # consider EXPLICITLY-toggled-on parents first, so an explicit choice wins an EXCLUDE
        parents = stableSort(node.PARENTS, key = explicitlyOn(toggles) first)
        for pid in parents:
            if pid in enabled: continue
            p = graph[pid]; if p is null: report missing; continue
            if p.OPTIONAL:
                on = toggles[pid] if present else p.DEFAULT
                if not on: continue                # an off optional ⇒ skip (and its subtree, see §12.5)
            if conflicts(p, kept): continue        # EXCLUDE (§12.4)
            enabled.add(pid); kept.add(pid); frontier.pushBack(pid)

    # ---- Phase 2: topologically order the enabled subgraph (post-order DFS) ----
    order = []; visited = {}; onStack = {}
    function emit(id):
        if id in visited: return
        if id in onStack: warn("cycle through " + id); return   # break back-edge (invariant I2)
        onStack.add(id)
        for pid in graph[id].PARENTS:
            if pid in enabled: emit(pid)           # parents first
        onStack.remove(id); visited.add(id)
        order.append(id)
    emit(launchId)
    return order
```

(VidyaGod: `ManifestModel::ResolveNodeOrder`.)

## 12.3 `OPTIONAL` and `DEFAULT` — toggles

A node referenced as a parent is a **hard dependency** unless it declares `OPTIONAL: true`, which makes it a **toggle**:

- It is **on** iff the user's `toggles` map says so; absent a choice, it falls back to its own `DEFAULT` (default `true`).
- An **off** optional node is simply not enabled — it does not enter the closure, so its layers don't mount and its
  exec/persistence don't apply.

`OPTIONAL`/`DEFAULT` live on the *node*, not on the `PARENTS` edge — so the same optional node referenced by two parents
has one consistent toggle state. Optional content is how DLC, mods and feature flags are modelled:

```json
{ "NODE_ID": "morrowind_tribunal", "ROLE": "content", "OPTIONAL": true, "DEFAULT": false,
  "PARENTS": ["morrowind_tribunal_data"], "LAYERS": [ … ] }
```

The implementation surfaces the set of reachable `OPTIONAL` nodes as the user-facing toggle list (VidyaGod:
`OptionalNodes`). Runner nodes are excluded from that list.

## 12.4 `EXCLUDE` — mutual exclusion (pick-one)

`EXCLUDE` lists node ids this node cannot coexist with. It is **symmetric** by intent — both nodes should list each other;
a validator warns if only one does. When resolution would enable two mutually-exclusive nodes, **first-kept wins**: the
one already in the `kept` set blocks the later one.

Two subtleties make this behave well:

- **Explicit choice beats a default.** In phase 1, a node's parents are stable-sorted so that *explicitly toggled-on*
  parents are considered before others. So if option A is default-on and the user explicitly turns on its excluded
  sibling B, B is considered first and kept, and A's default is dropped — the user's choice wins rather than being
  silently overridden by A's default.
- **Symmetric check.** `conflicts(p, kept)` is true if `p` excludes any kept node **or** any kept node excludes `p`.

Use `EXCLUDE` for "choose exactly one" sets (renderer backends, mutually incompatible mods, edition-specific patches).

## 12.5 The hierarchy gate

Resolution only **descends into nodes it keeps**: an optional node that is turned off is never visited, so *its* unique
parents are not pulled in either. A subtree that exists only to support an off optional node is naturally dropped with it.
Conversely, a node reachable through *another* kept path stays (it isn't orphaned just because one route to it was
disabled). This "you get a node's ancestors only if you kept the node" rule is the **hierarchy gate**, and it falls out of
the BFS pushing onto the frontier only nodes it enabled.

## 12.6 Order = priority

The emitted order is the overlay stacking order: **earlier = lower priority, later = higher**, with the launch node last
and highest. Within one node, `LAYERS` array order breaks ties (later layer wins). Concretely:

- A base content node listed early; a mod node listed later → the mod's files override the base's. *That is the load
  order* — no separate mod-ordering construct exists.
- The launchable's own layers sit on top of all parents — the place for variant-specific overrides.
- `PARENTS` **list order** is the tie-break among a node's own parents: list a parent later to give it higher priority.

(Edit layers add two more tiers — base edits below the user's writable layer, OVERRIDE edits above everything — see
chapters 6 and 13.)

## 12.7 Cycles (invariant I2)

`PARENTS` MUST be acyclic. If a cycle exists, resolution still completes: the depth-first order pass detects the back-edge
(a node already on the recursion stack), reports it, and skips that edge so emission terminates. A validator MUST report
cycles as errors (chapter 15); the runtime tolerates them defensively rather than hanging.

## 12.8 Modules (legacy term)

Generation-1 manifests used a `MODULES` array (with `REQUIRED`/`DEFAULT`/`EXCLUDE`/`PARENTCOMPONENT`) on a variant to
select components. The node graph subsumes this: a "module" is just an `OPTIONAL` content node, `REQUIRED` is a
non-optional parent, `EXCLUDE` is unchanged, and the `PARENTCOMPONENT` chain is ordinary `PARENTS`. New packages use
nodes; the term "module" survives only in older material and in some UI labels.

Next: [The runtime model](13-runtime-model.md).
