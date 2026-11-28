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

## 5.1 Containment in place

Resolving a node walks its list; a `NODE` layer (whose `WHEN` holds) brings, at its position, the contained node's
composition (§5.4). Everything the contained node writes is therefore **beneath** whatever follows the `NODE` layer in
the container, and above whatever precedes it. A container chooses where its parents go:

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
  `"EXEC"`, `"ENV"`, `"DLL"`, `"REG/HKCU\\Software\\X"` likewise. A fact namespace the selections do not name is
  not taken. Facts are global and are never re-rooted by `TARGET`.
- **An entry is a tree** (chapter 09 §9.2): `"EXEC/Play"` takes the entry, `"EXEC/Play/TILE"` its tile,
  `"EXEC/Play/TILE/COVER"` the cover alone, each placed at the same address in the container.
- A pair can rename a fact too (`["EXEC/Play", "EXEC/Demo"]`, `["VARS/a", "VARS/b"]`), but never a registry key:
  `REG` is taken by key, where it is (chapter 07), and a validator refuses a `REG` pair. Facts land at their own
  address: `"EXEC/Reign of Chaos/TILE"` and `"EXEC/The Frozen Throne/TILE"` in one `TAKE` are two places.
- **Two selections that land on the same address** (the same leaf name from two folders, a rename onto a taken
  leaf; compared without case for `FILES` and `REG`) are an error: the winner would depend on match order. So are
  **two selections where one holds the other** (`"FILES/a/"` and `"FILES/a/b.dll"`, `"EXEC/Play"` and
  `"EXEC/Play/TILE"`): the inner one would be taken twice. Selections in different namespaces never overlap.
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

## 5.4 Composition

A resolution is a **composition by history**: each node's list, in order, laid over what came before it. Later wins,
with one exception that makes history safe: **a value a layer only carries never overrides a value already in the
composition.**

**Occurrences.** An occurrence is one way a node is reached: (its CID, the `TAKE`s and `TARGET`s it is reached
through). Each occurrence is composed once, from its own list: every layer whose `WHEN` holds, in order; a `NODE` layer
contributes the contained occurrence's composition. Reaching the same occurrence twice (two parents sharing an
ancestor: a diamond) composes it once. Every WC3 variant reaching the pristine through two version chains mounts it
once.

**Provided and carried.** Within one item of a list:

- an occurrence's **own** layers (every layer of its list that is not `NODE` or `INTO`) are **provided** by it;
- a `NODE` layer **with a `TAKE`** provides everything it takes: the container chose those values here;
- everything else a `NODE` layer brings (the contained node's own containment) is **carried**.

**Identity.** A content layer is identified by what it is: (its type, its `SOURCE` CID, its placed target, the view it
is taken through). One file placed twice at one place is one value. Every other layer (an `EDIT`, a fact) is
identified by its writer: the occurrence whose list holds it, and its index there. An edit therefore applies once
per composition, however many paths reach it.

**Laying an item over the composition**, value by value, in the item's order:

1. a value not yet in the composition is appended;
2. a **carried** value already in the composition is skipped: a stale copy of history never comes back (a version
   that contains the original never pulls the original back over the patches composed after it);
3. a **provided** value already in the composition **moves** to this place (later wins), with one exception: the
   addresses that *carried* values after it in the same item rewrite (an edit of a file it places, the same fact
   address). Those stay with the value's old place, which is kept; the moved copy leaves those addresses to the
   layers below it. A node re-mentioned to be raised over newer files keeps its tuning:

```jsonc
// dgv: dgVoodoo's files, and cfg (an EDIT of dgVoodoo.conf). a: the game, then dgv. b: a, then b's own files.
{ "LABEL": "c", "LAYERS": [ { "NODE": "b" }, { "NODE": "dgv" } ] }
// → game, dgv, cfg's edit, b-over, dgv again — this copy leaving dgVoodoo.conf (edited) to the one below
```

A re-mention that moves a value over something written **after its old place** changes what wins; an implementation
MUST report it ("raise"), and a validator warns (chapter 16), since a re-mention that overrides a newer value may not be
meant. Diamonds and re-mentions that change nothing are silent.

Only a node's **own** layers move. What it contains is carried, and stays where it was: re-mentioning a node whose own
zip lies beneath the patch it contains (`v2 = [ZIP base, NODE patch]`, then `[NODE v2, NODE v2]`) raises `base` over
`patch`. That is a raise like any other, reported; mention a node again only to raise it. An item's own composition is
never undone by containing it: a node whose list raised a library (`c` above) keeps that raise inside any node that
contains it.

**Containment is a boundary** (invariant I3): a value never moves out of the node that contains it, and a mention
inside another node never pulls a value out of the list that placed it. A node inside itself (a cycle) is reported
and its back edge skipped; resolution completes.

Composition is defined over layers, `EDIT` targets and fact addresses. What lies inside an archive is never the
resolver's to see: two content layers at one place stack in composition order and the later one's files win, file by
file, in the mount.

*(VidyaGod: `Fold::Resolve`'s `Composer` in `fold.cpp`; the reference: `tools/gen6/resolve.py` `_compose`/`_combine`.)*

## 5.5 What containment replaces

| Gen 5 | Gen 6 |
|-------|-------|
| `OVER: ["parent"]` | `{"NODE": "parent"}` at the position the node chooses |
| placement variables (`%DXVK_TARGET%`, `%MONO_TARGET%`, `%DirectPlayPath%`, …) | `TARGET` on the `NODE` layer |
| `SUBMOUNTS` relocating part of an archive | a holder node for the archive, contained with a `TAKE` pair |
| "requires" (an any-of group, `NOT`) | `INTO` layers and conditions naming node CIDs (chapter 12 §12.5, chapter 23) |

| Gen 8 | Gen 9 |
|-------|-------|
| a repeated occurrence held at its first position, or moved within one list | provided values move, carried ones never come back (§5.4) |
| `DecidingMentions` (held/move warnings) | the `raise` report: a re-mention overriding a newer value |
| entries folded field by field | every address written whole; `TAKE` reaches any part of an entry (chapter 09 §9.2) |

Next: [`EDIT` layers](06-edit-layers.md).
