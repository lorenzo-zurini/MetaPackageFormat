# 05 · `NODE` layers: containment, `TAKE`, `TARGET`

A `NODE` layer says "this node is made of that one, here". It is how a version builds on the previous one, how a game
uses a library, and how a runner assembles its build. There is no other parent relation.

```jsonc
{ "NODE": "bafkrei…dgvoodoo_2_81_3",
  "TAKE": ["FILES/MS/x86/D3D8.dll", "FILES/dgVoodoo.conf", "DLL/d3d8"],
  "TARGET": "FILES/%GameDir%" }
```

| Field | Meaning |
|-------|---------|
| `NODE` | the contained node's CID |
| `TAKE` | what to take of it: a list of selections (§5.2). Absent: everything |
| `TARGET` | the `FILES` address its files are placed under (§5.3). Absent: where the container's own files are |

## 5.1 Expansion in place

Resolving a node walks its list; a `NODE` layer (whose `WHEN` holds) is replaced, at its position, by the contained
node's own layers, expanded the same way. Everything the contained node writes is therefore **beneath** whatever
follows the `NODE` layer in the container, and above whatever precedes it. A container chooses where its parents go:

```jsonc
{ "LABEL": "Wipeout XL enhanced", "LAYERS": [
    { "NODE": "bafkrei…wipeout_pristine" },          // the game
    { "NODE": "bafkrei…dgvoodoo", "TARGET": "FILES/%GameDir%" },  // a wrapper, above the game's files
    { "EDIT": [ … ], "TARGET": "FILES/%GameDir%/dgVoodoo.conf" } ] } // configured, above the wrapper
```

A `NODE` layer whose CID is not in the library is reported as missing, and a validator refuses it.

## 5.2 `TAKE`: selecting part of a node

`TAKE` is a list. Each selection is either an address string or an `[address, new name]` pair. An address that no
selection matches is not taken.

**Files are left-stripped**: a taken item lands under the `TARGET` by its own name, not its full path.

| Selection | Takes | Lands at |
|-----------|-------|----------|
| `"FILES/a/b/c.dll"` | that file | `<TARGET>/c.dll` |
| `"FILES/a/b"` | that folder | `<TARGET>/b/…` |
| `"FILES/a/b/"` | that folder's **contents** | `<TARGET>/…` |
| `"FILES"` | every file | `<TARGET>/…` (the paths as they were) |
| `["FILES/dinput8.dll", "FILES/%asi_proxy%.dll"]` | that file, renamed | `<TARGET>/<the new name>` |
| `["FILES/a/b", "FILES/c"]` | that folder, renamed | `<TARGET>/c/…` |

- A pair renames a file or a folder, never "contents". The new name MAY omit the namespace
  (`["FILES/x.dll", "y.dll"]`); it is the source's.
- **Facts** are taken by namespace or by address: `"VARS"` takes every variable, `"VARS/lm_lib_var"` one;
  `"EXEC"`, `"ENV"`, `"DLL"`, `"REG/HKCU/Software/X"` likewise. A fact namespace the selections do not name is not
  taken. Facts are global and are never re-rooted by `TARGET`.
- A pair can rename a fact too (`["REG/HKCU/Software/A", "REG/HKCU/Software/B"]`).
- **Two selections that land on the same address** (the same leaf name from two folders, a rename onto a taken
  leaf; compared without case for `FILES` and `REG`) are an error: the winner would depend on match order. A whole
  namespace or a folder's contents may overlap other selections; those merge like any layers.
- Taking part of a delta-built node reconstructs the delta's tree first, then selects from it.

Selecting is how a game uses one file of a large library, how an engine takes only a game's data folder, and how a
package uses a holder node to place one file out of an archive:

```jsonc
// doukutsu-rs takes the English release's data folder and nothing else
{ "NODE": "bafkrei…cave_story_english", "TAKE": ["FILES/%GameDir%/data"], "TARGET": "FILES/%GameDir%" }
```

## 5.3 `TARGET`: placement

`TARGET` names the `FILES` address the contained node's files are placed under; it MUST be in `FILES`. Placement
composes:

- A **relative** path inside the contained node lands under the `TARGET` (and under its container's placement, and
  so on outwards).
- An **anchored** path (one starting with `%GameDir%`, `%SysDir32%`, `%RunnerMount%`, … or another `%variable%`)
  stays where it is: a library that writes `%SysDir32%/x.dll` puts it in the system folder wherever it is contained.
- Path-valued fields that travel with files are placed with them: `EDIT` and `KEEP` addresses, an entry's `EXE` and
  `WORKDIR`.

A library written with relative paths can thus be contained under any folder a game chooses:
`{"NODE": "…dxvk", "TARGET": "FILES/files/lib64/wine/dxvk"}`.

## 5.4 Occurrences

An **occurrence** is one expansion: (the node's CID, the `TAKE`s and `TARGET`s it is reached through). Reaching the
same occurrence twice, which happens whenever two parents share an ancestor (a diamond), yields **one** occurrence,
at its **first** position. Every WC3 variant reaching the pristine through two version chains mounts it once.

A later mention moves an occurrence only when it is **in the same list** that placed it: a node may list a library
early and again later to raise it. A mention inside another node never pulls an occurrence out of the list that
placed it, and **a node never leaves the node that contains it** (invariant I3).

An implementation SHOULD warn when a mention was held back or moved *and* the position decided a winner (a fact
changes, or two layers with overlapping targets fold in the other order). Diamonds and redundant mentions that change
nothing are silent.

A node inside itself (a cycle) is reported and its back edge skipped; resolution completes.

*(VidyaGod: `Fold::Resolve` expansion and `DecidingMentions` in `fold.cpp`.)*

## 5.5 What containment replaces

| Gen 5 | Gen 6 |
|-------|-------|
| `OVER: ["parent"]` | `{"NODE": "parent"}` at the position the node chooses |
| placement variables (`%DXVK_TARGET%`, `%MONO_TARGET%`, `%DirectPlayPath%`, …) | `TARGET` on the `NODE` layer |
| `SUBMOUNTS` relocating part of an archive | a holder node for the archive, contained with a `TAKE` pair |
| "requires" (an any-of group, `NOT`) | `ANY` and `NOT` layers (chapter 12 §12.5) |

Next: [`EDIT` layers](06-edit-layers.md).
