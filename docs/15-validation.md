# 15 · Validation

A validator checks a node graph for correctness before it is published or launched. This chapter enumerates every rule.
**Errors** SHOULD block (publishing, and ideally launching the affected node); **warnings** advise but don't block. The
reference implementation runs this over the whole graph (or a single package's nodes) on demand and at the launch gate.
(VidyaGod: `ManifestModel::ValidateNodeGraph`.)

A validator iterates every node (optionally scoped to a subset, while still consulting the rest of the graph for
cross-references) and applies the rules below.

## 15.1 Graph integrity (errors)

| Rule | Severity | Detail |
|------|----------|--------|
| **PARENTS resolve** | error | Every `LABEL` in `PARENTS` MUST exist in the graph. A reference to a missing node is an error. |
| **Acyclic PARENTS** | error | The `PARENTS` graph reachable from a node MUST be acyclic (invariant I2). A cycle is an error. (The runtime still completes by breaking the back-edge, but the package is malformed.) |
| **LABEL uniqueness** | (enforced at index) | Duplicate ids are dropped first-seen-wins at index time (chapter 4) with a diagnostic; a validator MAY additionally report cross-repo collisions. |

## 15.2 Payload rules

| Rule | Severity | Detail |
|------|----------|--------|
| **Known TYPE** | error | A node's `TYPE` MUST be one the implementation knows. An unknown type is an error, not something to ignore — a payload nobody applies is a package that quietly does less than it says. |
| **Content node has a path** | error | Every `Content` node MUST declare a local path (`PATH` or `SOURCE.PATH`) and a known `FORM`. |
| **STORE zips only** | error | A locally-present `FORM: "zip"` whose archive contains any DEFLATE-compressed entry is an error — it will not mount. Fix: re-create with `zip -0`. (Checked where the zip is present; remote-only content is checked after fetch.) |
| **Dir content can't publish** | warning | `FORM: "dir"` is an unzipped authoring intermediary; warn that it must be converted to a STORE zip before publishing (the content network seeds files/zips, not directories). It still test-runs locally. |
| **A delta has something below it** | error | `FORM: "delta"` reconstructs against the composed view at its own `TARGET` when it declares no `BASE_TARGETS`. A delta that is the LOWEST layer composing bytes at its target has nothing to reconstruct against: the runtime skips it and the package launches with that content absent. Checked on the ASSEMBLED mount plan, where the question is exact — the plan holds every layer of every node in the closure, so "is there a byte view below me" is decidable, and only `zip`/`delta` layers make one. (It is *not* decidable from a node graph, which is why this is enforced where the plan is built; see the row below.) |
| **A declared base mounts something** | error | When a `delta` names its bases explicitly (`BASE_TARGETS`, always an ordered list), **every** named target MUST be one that an EARLIER layer in the resolved plan composes bytes at — earlier because the runtime registers each target's composed view as it walks the plan, and only `zip`/`delta` layers produce one (a `dir` or `file` layer mounts fine and is not a base). A base naming anything else is not a launch failure: the runtime finds no bytes to reconstruct against, skips the layer, and the game starts with that content simply absent. The *implicit* base (no `BASE_TARGETS`) is the row above, checked the same way and at the same point — the two are one rule with the base list defaulting to the layer's own target. **Where it belongs:** the answer depends on plan ORDER and layer TYPE, neither of which a node-graph walk sees, so this is checked when the mount plan is ASSEMBLED rather than by a graph validator. (VidyaGod: `VfsMount::BuildLayerSpec`, so a real launch reports it too; `--audit-packages` builds every plan and captures the same diagnostic. `ManifestModel::ValidateNodeGraph` does **not** implement it.) |
| **A relative FILE** | error | A `FileEdit`/`BinaryPatch` `FILE` MUST be relative to its pass's base. An absolute path (including a `%RuntimePath%/`-prefixed one) escapes the base and the edit lands where nothing reads it. (VidyaGod checks `BinaryPatch` here; a `FileEdit` is currently caught at apply time, which is after the launch has begun.) |

## 15.3 Selection rules

| Rule | Severity | Detail |
|------|----------|--------|
| **EXCLUDE symmetry** | warning | `EXCLUDE` is meant to be symmetric; if node A excludes B but B does not exclude A, warn. |
| **EXCLUDE target exists** | warning | An `EXCLUDE` entry referencing a missing node is warned (not an error — it simply has no effect). |
| **WHEN has a consumer** | error | A `WHEN` on a `DeclareExec` or `DeclareLibraryItem` is never evaluated — those payloads become the node's identity at index time. Accepting it would mean a node that looks conditional and is not. Point the author at `TOGGLE`. |
| **Unordered write conflict** | *recommended* | Two nodes with **no dependency relation** that write the same file path or the same registry value. Which one wins is unspecified (invariant **I9**), so the package's behaviour is undefined. Fix: make one a parent of the other. **This rule MUST be evaluated over a RESOLVED CLOSURE**, never over a whole library — two nodes that never meet in any closure are not in conflict, and a library-wide scan reports dozens of collisions that cannot happen, every one of them wrong. It is a SHOULD rather than a MUST because that closure-scoped analysis is substantial; the reference implementation does not yet provide it. |

## 15.4 Launchable rules

| Rule | Severity | Detail |
|------|----------|--------|
| **Has a host platform** | warning | A launchable with no `HOST` is warned (it can't be routed to a runner). |
| **A runner serves its platform** | warning | If no runner serves the launchable's `HOST` on this machine (directly or — for a chaining-aware validator — via a chain), warn. |
| **PATH case-exact** | error | A launchable's `PATH` MUST case-exactly match a real file in its locally-present content. A case-only mismatch (`MW4Mercs.exe` vs `MW4mercs.exe`) is an error — the case-sensitive mount would never find it (silent crash). Skipped for `%var%`-bearing paths and un-hydrated content. A helpful validator suggests the correct casing, or the correct nested path if the basename exists under a top folder (a zip that nests content shifts the path). |
| **No cross-layer case collisions** | error | Two different `Content` nodes in the merged view contributing paths that differ only in case (base `MAPS/foo`, patch `maps/foo`) is an error — both exist on the case-sensitive mount and a lookup can hit the wrong one. (Collisions *within one archive* are the upstream content's own and are ignored.) |
| **Unambiguous tile** | error | A launchable MUST reach **at most one** `DeclareLibraryItem` through `PARENTS`. Two reachable tiles means nothing can decide which game it belongs to ([ch. 3 §3.4](03-roles.md)). |

## 15.4a Library tile rules

| Rule | Severity | Detail |
|------|----------|--------|
| **Tile has a UID** | error | A `DeclareLibraryItem` MUST declare a non-empty `UID`. The UID keys saved state, settings and the content root inside a prefix; a tile without one silently shares another tile's state or lands its content at the wrong path. This was unexpressible before the tile was its own node — now it is one field on one node and MUST be checked. |

## 15.5 Runner rules

| Rule | Severity | Detail |
|------|----------|--------|
| **Declares GUEST platforms** | warning | A runner (`DeclareExec` with `GUEST`) whose `GUEST` list is empty once resolved is an edge to nowhere — it can run nothing. (The older "a runner's own VFS layers are ignored" rule is GONE: a node is one layer of one TYPE, so a runner *declaration* cannot also carry content, and the footgun it guarded is unrepresentable.) |
| **Prefix runner routes through drive_c** | warning | A `PREFIX_GENERATE` runner whose `CONTENT_ROOT` contains no `drive_c` is warned — content won't land inside the `C:` drive where the Windows program expects it. |

## 15.6 What a validator should *also* do (recommended)

These aren't in the reference's core validator but a thorough one SHOULD consider them:

- **CID-or-file presence** — warn if a `Content` node has neither a present local file nor a `SOURCE.CID` (it's
  unrunnable and unshippable).
- **Cover resolvability** — warn if a tile's `COVER.PATH` is missing locally and has no CID.
- **Chain reachability** — for each launchable, attempt full chain resolution (chapter 11) and warn if the platform can't
  reach the machine platform with the installed runners.
- **Token sanity** — warn on `%TOKEN%`s that aren't built-ins and aren't declared by any `CustomVar` in the closure
  (likely a typo that will survive substitution as a literal `%TOKEN%`).
- **Orphan notice** — note (do not error) a node that nothing depends on and that is not a launchable. That is the
  *normal* state while authoring — you capture, then wire, then declare the exec last — so it is information, never a
  reason to rewrite the graph "to fix it".

## 15.7 Scope

A validator MAY validate the whole graph (for a publisher/CI) or just one package's nodes (for the launch gate), while
still consulting the rest of the graph for cross-references (a launchable's runner availability, its `PARENTS`). Scoped
validation lets a launch verify only what it's about to run without auditing the entire catalog.

Next: [Run modes & the CLI surface](16-cli-and-run-modes.md).
