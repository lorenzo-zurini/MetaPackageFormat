# 15 · Validation

A validator checks a node graph for correctness before it is published or launched. **Errors** SHOULD block
(publishing, and ideally launching the affected node); **warnings** advise. The reference implementation runs this
over the whole graph (or one package's nodes) on demand and at the launch gate. (VidyaGod:
`ManifestModel::ValidateNodeGraph`.)

## 15.1 The node itself

| Rule | Severity | Detail |
|------|----------|--------|
| **Known vocabulary** | error | Every top-level key MUST be one the format defines. An unknown key — a typo'd section name, a legacy `TYPE` — is refused at lower, and the node is indexed with the reason attached so validation can name it and resolution refuses to route through it. |
| **Non-empty sections** | error | A present payload section MUST have at least one entry. `"VARS": []` is a node that says it contributes something and contributes nothing. |
| **Well-typed payload** | error | What lowering emits is well-typed by construction; a field of the wrong JSON type is a refusal naming the field, never a crash three frames later. |
| **TOGGLE value** | error | `TOGGLE` is `"on"` or `"off"`; anything else is refused, never guessed. |
| **WHEN has a consumer** | error | A `WHEN` on a node with no payload — a plain node, or one carrying only `TILE`/`ENTRYPOINTS` — is never evaluated. Point the author at `TOGGLE`. |
| **Malformed WHEN** | error | A condition that does not parse fails open (always applies); caught statically. |
| **Pointless node** | warning | No payload, no `TILE`, no `ENTRYPOINTS`, no `OVER`: not a node, a mistake. |

## 15.2 The edge

| Rule | Severity | Detail |
|------|----------|--------|
| **Refs resolve** | error / warning | Every plain ref in `OVER` MUST name a node in the graph. An absent any-of *member* is a warning — the group is a choice, and one present member resolves it. |
| **NOT target exists** | warning | A `NOT` naming a missing node has no effect. |
| **Acyclic** | error | The graph reachable through positive `OVER` refs MUST be a DAG. (The runtime still completes by breaking the back-edge.) |
| **Group satisfiable** | error | An any-of group with no member in the graph can never be satisfied. A repeated member is warned. |
| **Not both required and excluded** | error | A node MUST NOT `OVER` a ref and `NOT` the same ref. |
| **Shape** | error | A group MUST be non-empty; an object entry MUST be exactly `{"NOT": ref}`; refs are non-empty strings. |

## 15.3 Content

| Rule | Severity | Detail |
|------|----------|--------|
| **A layer has a path** | error | Every `LAYERS` entry MUST declare a local path (`PATH` or `SOURCE.PATH`) and a known `FORM`. |
| **STORE zips only** | error | A locally-present `FORM: "zip"` with any DEFLATE entry will not mount. Re-create with `zip -0`. |
| **Dir content can't publish** | warning | `FORM: "dir"` is an authoring intermediary; convert to a STORE zip before publishing. |
| **A delta has something below it / a declared base mounts something** | error | Checked on the ASSEMBLED mount plan (VidyaGod: `VfsMount::BuildLayerSpec`, and `--audit-packages`), where plan order and layer type are known; a graph validator cannot decide it. |
| **A relative FILE** | error | A `PATCHES`/`FILEEDITS` `FILE` MUST be relative to its pass's base. |
| **Patch structure** | error/warning | `MODE` ∈ Replace/Cave/Poke; `OFFSET` or `ANCHOR`; the mode's payload field; `EXPECT` guard recommended. |
| **No cross-layer case collisions** | error | Two layers in one mount contributing paths that differ only in case. |

## 15.4 Launchables and entrypoints

| Rule | Severity | Detail |
|------|----------|--------|
| **Entry has a HOST** | error | Every `ENTRYPOINTS` entry MUST declare `HOST` (refused at lower). |
| **Entry is never conditional** | error | A `WHEN` on an entry is refused. |
| **Distinct entry labels** | error | Two entries of one node with the same `LABEL` are indistinguishable in the picker. |
| **A runner serves its platform** | warning | No runner on this machine serves the entry's `HOST`. |
| **PATH case-exact** | error | A launchable entry's `PATH` MUST case-exactly match a real file in the locally-present mount (a helpful validator suggests the casing, or the nested path). Skipped for `%var%` paths and un-hydrated content. Every entry is checked, not just the default. |
| **Has an identity** | warning | A launchable that carries no `TILE` and reaches none through `OVER` appears under no card. A runner legitimately has none. |

## 15.5 Tiles

| Rule | Severity | Detail |
|------|----------|--------|
| **Tile has a UID** | error | A `TILE` MUST declare a non-empty `UID` (it keys saves, settings, the content root and the card). |
| **One UID, one main** | warning | Among the launchables of a `UID`, more than one is `OVER` no other of that UID: the card shows each at top level. Legitimate for two independent installs; a different tile `OVER` the main is an expansion and never a lint. |

## 15.6 Runners

| Rule | Severity | Detail |
|------|----------|--------|
| **Declares GUEST platforms** | warning | A runner entry with an empty `GUEST` is an edge to nowhere. |
| **Prefix runner routes through drive_c** | warning | A `PREFIX_GENERATE` runner whose `CONTENT_ROOT` has no `drive_c`. |

## 15.7 Grafts and conflicts (recommended)

| Rule | Severity | Detail |
|------|----------|--------|
| **Canonical purity** | lint | A node that is the pristine game SHOULD carry only `LAYERS`, `ENTRYPOINTS`, `TILE`. |
| **Conflict report** | *recommended* | Over a **resolved mount** (never a whole library): two grafts providing the same target path with different content CIDs, overlapping patch ranges on one file, or the same key with different values. Surfaced to the instance for a precedence/winner decision; identical bytes and appended lines never conflict. |
| **Unordered write conflict** | *recommended* | Two closure nodes with no dependency relation writing the same path or registry value: which wins is unspecified (I9). Fix: make one `OVER` the other. |

## 15.8 What a validator should *also* do (recommended)

- **CID-or-file presence** — warn if a layer has neither a present local file nor a `SOURCE.CID`.
- **Cover resolvability** — warn if a tile's `COVER.PATH` is missing locally and has no CID.
- **Chain reachability** — for each entry, attempt full chain resolution and warn if the platform can't reach the
  machine platform with the installed runners.
- **Token sanity** — warn on `%TOKEN%`s that are neither built-ins nor declared by any `VARS` entry in the mount.
- **Orphan notice** — note (do not error) a node nothing is `OVER` that is not launchable: the normal state while
  authoring, and the normal state of a graft.

## 15.9 Scope

A validator MAY validate the whole graph (publisher/CI) or one package's nodes (the launch gate), consulting the
rest of the graph for cross-references.

Next: [Run modes & the CLI surface](16-cli-and-run-modes.md).
