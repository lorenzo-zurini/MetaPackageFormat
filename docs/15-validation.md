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
| **PARENTS resolve** | error | Every `NODE_ID` in `PARENTS` MUST exist in the graph. A reference to a missing node is an error. |
| **Acyclic PARENTS** | error | The `PARENTS` graph reachable from a node MUST be acyclic (invariant I2). A cycle is an error. (The runtime still completes by breaking the back-edge, but the package is malformed.) |
| **NODE_ID uniqueness** | (enforced at index) | Duplicate ids are dropped first-seen-wins at index time (chapter 4) with a diagnostic; a validator MAY additionally report cross-repo collisions. |

## 15.2 Layer rules

| Rule | Severity | Detail |
|------|----------|--------|
| **VFS layer has a path** | error | Every `VFSZipLayer`/`VFSDirLayer`/`VFSFileLayer` MUST declare a local path (`PATH` or `SOURCE.PATH`). A VFS layer with no path is an error. |
| **STORE zips only** | error | A locally-present `VFSZipLayer` whose archive contains any DEFLATE-compressed entry is an error — it will not mount. Fix: re-create with `zip -0`. (Checked where the zip is present; remote-only layers are checked after fetch.) |
| **Dir layers can't publish** | warning | A `VFSDirLayer` is an unzipped authoring intermediary; warn that it must be converted to a STORE zip before publishing (the content network seeds files/zips, not directories). It still test-runs locally. |
| **Runner layers are ignored** | warning | A `runner` node that carries its *own* VFS layer is warned: runner layers are never mounted (the build comes from `PARENTS` content nodes — chapter 10 §10.5). Fix: move the layer to a content node and add it to the runner's `PARENTS`. |

## 15.3 Selection rules

| Rule | Severity | Detail |
|------|----------|--------|
| **EXCLUDE symmetry** | warning | `EXCLUDE` is meant to be symmetric; if node A excludes B but B does not exclude A, warn. |
| **EXCLUDE target exists** | warning | An `EXCLUDE` entry referencing a missing node is warned (not an error — it simply has no effect). |

## 15.4 Launchable rules

| Rule | Severity | Detail |
|------|----------|--------|
| **Has a host platform** | warning | A `launchable` with no `PLATFORM.HOST` is warned (it can't be routed to a runner). |
| **A runner serves its platform** | warning | If no runner node serves the launchable's `PLATFORM.HOST` on this machine (directly or — for a chaining-aware validator — via a chain), warn. |
| **CONTENTPATH case-exact** | error | The `EXEC.CONTENTPATH` MUST case-exactly match a real file in the launchable's locally-present content. A case-only mismatch (`MW4Mercs.exe` vs `MW4mercs.exe`) is an error — the case-sensitive mount would never find it (silent crash). Skipped for `%var%`-bearing paths and un-hydrated content. A helpful validator suggests the correct casing, or the correct nested path if the basename exists under a top folder (a zip that nests content shifts the path). |
| **No cross-layer case collisions** | error | Two different layers in the merged content view contributing paths that differ only in case (base `MAPS/foo`, patch `maps/foo`) is an error — both exist on the case-sensitive mount and a lookup can hit the wrong one. (Collisions *within one layer* are the upstream content's own and are ignored.) |

## 15.5 Runner rules

| Rule | Severity | Detail |
|------|----------|--------|
| **Declares GUEST platforms** | warning | A `runner` with an empty `PLATFORM.GUEST` is warned (it's an edge to nowhere — it can run nothing). |
| **Prefix runner routes through drive_c** | warning | A `PREFIX_GENERATE` runner whose `CONTENT_ROOT` contains no `drive_c` is warned — content won't land inside the `C:` drive where the Windows program expects it. |

## 15.6 What a validator should *also* do (recommended)

These aren't in the reference's core validator but a thorough one SHOULD consider them:

- **CID-or-file presence** — warn if a VFS layer has neither a present local file nor a `SOURCE.CID` (it's unrunnable and
  unshippable).
- **Cover resolvability** — warn if `META.COVER.PATH` is missing locally and has no CID.
- **Chain reachability** — for each launchable, attempt full chain resolution (chapter 11) and warn if the platform can't
  reach the machine platform with the installed runners.
- **Token sanity** — warn on `%TOKEN%`s that aren't built-ins and aren't declared by any `CustomVar` in the closure
  (likely a typo that will survive substitution as a literal `%TOKEN%`).
- **Unknown TYPE** — warn on a `LAYERS` entry whose `TYPE` the implementation doesn't recognize.

## 15.7 Scope

A validator MAY validate the whole graph (for a publisher/CI) or just one package's nodes (for the launch gate), while
still consulting the rest of the graph for cross-references (a launchable's runner availability, its `PARENTS`). Scoped
validation lets a launch verify only what it's about to run without auditing the entire catalog.

Next: [Run modes & the CLI surface](16-cli-and-run-modes.md).
