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
| **WHEN has a consumer** | error | A `WHEN` on a node with no payload — a plain node, or one carrying only `TILE`/`ENTRYPOINTS`/`VARIANT` — is never evaluated. |
| **Requirements are only meaningful when offered** | warning | A `TOGGLE`, an any-of group or a `NOT` on a node nothing can offer (a variant, or a node with no identity) is inert: the closure walk never reads them. |
| **Malformed WHEN** | error | A condition that does not parse fails open (always applies); caught statically. |
| **Pointless node** | warning | No payload, no `TILE`, no `ENTRYPOINTS`, no `OVER`: not a node, a mistake. |

## 15.2 The edge

| Rule | Severity | Detail |
|------|----------|--------|
| **Refs resolve** | error / warning | Every bare ref in `OVER` MUST name a node in the graph. An absent any-of *member* is a warning — the group is a requirement, and one present member can satisfy it. |
| **NOT target exists** | warning | A `NOT` naming a missing node has no effect. |
| **Acyclic** | error | The graph reachable through bare `OVER` refs MUST be a DAG. (The runtime still completes by breaking the back-edge.) |
| **Group satisfiable** | error | An any-of group with no member in the graph can never be satisfied. A repeated member is warned. |
| **Not both composed and excluded** | error | A node MUST NOT compose a ref and `NOT` the same ref. |
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

## 15.4 Variants and entrypoints

| Rule | Severity | Detail |
|------|----------|--------|
| **Entry has a HOST** | error | Every `ENTRYPOINTS` entry MUST declare `HOST` (refused at lower). |
| **Entry is never conditional** | error | A `WHEN` on an entry is refused; `RECOMMENDED` on an entry is refused (it is a node facet). |
| **Distinct entry labels** | error | Two entries of one list with the same `LABEL` are indistinguishable in the picker. |
| **A variant can run** | error | A `VARIANT` on a node with no effective entrypoints, or whose effective entrypoints are a runner's, has nothing to run. |
| **Inheritance is unambiguous** | warning | Two nodes at the same nearest distance beneath declare *different* entrypoint lists (or different faces): `OVER` order decides, and the author should mean it. |
| **A runner serves its platform** | warning | No runner on this machine serves the entry's `HOST`. |
| **PATH case-exact** | error | An entry's `PATH` MUST case-exactly match a real file in the locally-present mount (a helpful validator suggests the casing, or the nested path). Skipped for `%var%` paths and un-hydrated content. Every entry of every variant is checked. |
| **A variant has a face** | warning | A variant with no tile beneath it appears under no card. |

## 15.5 Tiles

| Rule | Severity | Detail |
|------|----------|--------|
| **Tile has a UID** | error | A `TILE` MUST declare a non-empty `UID` (it keys saves, settings, the content root and the card). |
| **One UID, one main** | warning | A `UID` has several tiles with no same-UID tile beneath them: two main faces, and the card shows both at top level. Legitimate for two independent installs; a tile above another is a child face and never a lint. |
| **Same face twice** | lint | A tile equal to one beneath it adds nothing; the variant would inherit it. |

## 15.6 Runners

| Rule | Severity | Detail |
|------|----------|--------|
| **Declares GUEST platforms** | warning | A runner entry with an empty `GUEST` is an edge to nowhere. |
| **Prefix runner routes through drive_c** | warning | A `PREFIX_GENERATE` runner whose `CONTENT_ROOT` has no `drive_c`. |

## 15.7 Grafts and conflicts (recommended)

| Rule | Severity | Detail |
|------|----------|--------|
| **Canonical purity** | lint | A node that is the pristine game SHOULD carry only `LAYERS`, `TILE` and, when every variant runs the same way, `ENTRYPOINTS`. |
| **Conflict report** | *recommended* | Over a **resolved mount** (never a whole library): two grafts providing the same target path with different content CIDs, overlapping patch ranges on one file, or the same key with different values. Surfaced to the instance for a precedence/winner decision; identical bytes and appended lines never conflict. |
| **Unordered write conflict** | *recommended* | Two closure nodes with no dependency relation writing the same path or registry value: which wins is unspecified (I9). Fix: make one `OVER` the other. |

## 15.8 What a validator should *also* do (recommended)

- **CID-or-file presence** — warn if a layer has neither a present local file nor a `SOURCE.CID`.
- **Cover resolvability** — warn if a tile's `COVER.PATH` is missing locally and has no CID.
- **Chain reachability** — for each entry, attempt full chain resolution and warn if the platform can't reach the
  machine platform with the installed runners.
- **Token sanity** — warn on `%TOKEN%`s that are neither built-ins nor declared by any `VARS` entry in the mount.
- **Orphan notice** — note (do not error) a node nothing is `OVER` that is not a variant: the normal state while
  authoring, and the normal state of a graft.

## 15.9 Scope

A validator MAY validate the whole graph (publisher/CI) or one package's nodes (the launch gate), consulting the
rest of the graph for cross-references.

Next: [Run modes & the CLI surface](16-cli-and-run-modes.md).
