# 12 · Resolution

Resolving turns a node, an instance and a set of ticked parts (grafts) into a **plan**: the ordered content and edits, and
the composed registry, DLLs, environment, variables, entries and keeps. Every runner in the chain is resolved the same
way, from its own node.

## 12.1 Phases

1. **Variables for `WHEN`.** Collect what a layer `WHEN` may read: the instance's values and per-launch overrides, the
   built-ins (`%UID%`, `%PackageUID%`, the screen size, the runner's anchors), and the `DEFAULT`s of ungated `VARS`
   layers reached through ungated `NODE` layers, in list order (the ticked parts' included, since they follow the
   variant). This phase reads the lists, not the composition, so nothing is circular. **Membership**: a `WHEN` naming a
   node CID is decided on the final composition, by a fixpoint: compose with every such condition guessed, read which
   nodes the composition holds, decide the conditions again, until nothing changes. A set of conditions that never
   settles (a node in only when it is out) is refused with the nodes it names (chapter 08 §8.6). An implementation MAY
   bound the rounds (VidyaGod: 64) and refuse beyond them.
2. **Composition** (§12.2): walk `LAYERS` (the root node's, then each applied part's, §12.5). Skip a layer whose
   `WHEN` does not hold; a part whose `INTO` layer's `WHEN` does not hold is not applied at all. A `NODE` layer brings
   its node's composition in place, through its `TAKE` and `TARGET` (chapter 05). Check each `INTO` (§12.5).
3. **Variables**: resolve every variable fully over the composed declarations (chapter 08 §8.5).

## 12.2 Composition

The sequence is built item by item, as chapter 05 §5.4 defines (provided values move, carried ones never come back,
an edit applies once). Then, walking it bottom to top:

- **Per address, the latest writer wins**, and it writes the address **whole** (I10): a file, a registry value, an
  environment variable, a DLL policy, a variable declaration, an entry, or a part of an entry taken by its deeper
  address (`EXEC/Play/TILE/COVER`, placed over the entry already there).
- **`null` deletes**: a value or, for a `REG` key, the key and everything beneath it; for `ENV`, the name; for `DLL`,
  the policy.
- **An `EDIT` applies to the value beneath it** at its position, and a `DELTA` reconstructs from the files it names by
  `SOURCE`, wherever composition places them (chapters 04 and 06).
- **`KEEP`** composes per address, and the most specific address wins whatever the order (chapter 13).
- **Entries** keep the order in which each label was first written (chapter 09 §9.2).
- **A re-provided copy** (a moved value whose old place was kept, chapter 05 §5.4) leaves the addresses it excepts to
  the layers below it: the mount hides them in that layer.

Order is position. A container decides where its parents go; a node never rises above the node that contains it
(I3); the variant's own layers sit above everything it contains; applied parts sit above the variant, each after what
it goes into (§12.5).

An implementation SHOULD be able to show, per address, the winner and the layers it beat (the conflict view), and
MUST report a re-mention that raised a node's own layers over newer values (`raise`, chapter 05 §5.4).

*(VidyaGod: `Fold::Resolve`, `Fold::ResolveVars` in `fold.cpp`. The reference definition is the Python
`tools/gen6/resolve.py`; `tools/gen6/fold_gate.py` holds the C++ to it plan for plan.)*

## 12.3 What a resolution produces

| Part | Contents |
|------|----------|
| sequence | content and `EDIT` layers in composition order, each with its placed target, its package folder, its `TAKE` view and, for a re-provided copy, the addresses it leaves below (`except`) |
| registry | `REG` values and created keys, per architecture |
| DLL, ENV, KEEP | the composed maps |
| declarations | the composed `VARS` |
| entries | the composed `EXEC` entries, by label |
| order | the occurrences, in composition order |
| events | `raise` (a re-mention overriding newer values), cycles, missing nodes, an `INTO` with nothing it goes into beneath it (`into-unmet`), a part whose `INTO` condition is false (`into-off`) |
| parts | the parts applied, in the order composition read them |
| error | a malformed layer, or conditions that never settle |

A malformed layer (no single type key) is an error for the whole resolution: nothing is launched from it.

## 12.4 Variants and rows

Picking a row selects one variant node and one of its tiled entries (chapter 09 §9.4). The variant's node is the
root of the resolution. Nothing beneath it is a choice: playing a version runs that version, its options are its own
variables, and what it contains is simply part of it.

## 12.5 Parts (grafts)

A **part** (a mod; "graft" is the same word) is a node whose list opens with one or more `INTO` layers (chapter 23
§23.1). Each `INTO` layer lists alternatives; the part goes into all of its layers. Nothing a part goes into names it
(I9), and anyone may publish one.

- **The hard link.** A part is offered on the end being launched (the root) when its `INTO` names it, or once a part it
  names is applied: every `INTO` layer must have a member that is the end or an applied part. A node that merely occurs
  beneath the end is never a reason to offer one (an end containing an end is another end). A part a node of the row
  contains (with a `NODE` layer) is part of the row, not an option.
- **Checked where it occurs.** An `INTO` is satisfied when one of its members is in the composition beneath it. A part
  a node contains whose `INTO` is not satisfied blocks that row (listed, with the reason); a part selected at launch
  that is not offered is not applied, and is reported.
- **Applied as a set.** The selection has no order. Parts apply in rounds: each round applies every selected part
  offered with the parts before it applied (a part always after what it goes into), parts of one round in CID order.
  A part whose `INTO` condition is false on the selection, or that would leave an `INTO` unmet, is dropped and
  reported.
- **Default ticks.** A fresh instance ticks the offered parts whose `INTO` layer says `"RECOMMENDED": true`. A part
  with an entry of its own is never ticked: it is a row (below).
- **Listed** after what each goes into, then by `LABEL`, then CID: a presentation, never an order.
- **Exclusive parts.** A condition on a part's `INTO` (`<cid> == false`) keeps it out where the other is; ticking either
  side unticks the other, so exclusive parts behave as a choice of one.
- **Merging.** Parts that write the same unit merge by its primitive (chapter 23 §23.4); a conflict is shown, never
  settled by a position.
- **`SECTION`** (a node field) places a part in the launch window's collapsible tree.
- **A part with an entry** is a row: with its own tile, a card of its own (Forgotten Empires); without one, it takes a
  tile and is a row under that card (a mod loader such as SKSE). Launching it applies it over the end it goes into,
  with that end's pre-ticked parts, and runs its own entry (chapter 09 §9.4).
- **Part declarations count in phase 1**, so a part's own options gate its own layers.
- **A part's footprint** (2026-11-27): an `INTO` layer may carry a `TAKE` of FILES addresses: what of the part's files
  go into the end, and where. A file or folder it names stays where it is (a footprint has no `TARGET`: unlike a `NODE`'s
  `TAKE` it does not left-strip), a pair moves or renames one, `"FILES"` takes every file where it is, and what it does
  not name is not mounted; two selections where one holds the other are refused, as on a `NODE` layer (chapter 05
  §5.2): a file moved elsewhere is selected by its pair alone. It applies wherever the part is composed (applied at launch, or contained in a pack), in `INTO`-layer order
  (the first innermost), inside its container's view. It never touches the part's facts: its options, DLL overrides,
  entries and kept folders are its own (a fact address in an `INTO`'s `TAKE` is refused). ReShade's `d3d11.dll` taken as
  `dxgi.dll` chain-loads beside ENB's. Conflicts (chapter 23 §23.4) are read where the files land, at every place; a
  footprint that takes none of the part's files is a validator warning.

## 12.6 The instance

The **instance** is the user's loadout for a family: the selected variant, the ticked parts (a set), the
pinned runner chain, variable values (including drawn `POOL` values) and the owned state (chapter 13). It is local
data, never a node, and it is keyed by node CID; `LABEL` is never a key.

The cost of content addressing, stated once: re-minting a part changes its CID, so a saved selection that names a
CID the library no longer has was saved against an older package; the implementation falls back to the tile's
defaults (`catalog repoint` renames the CID in the instances it carries a re-mint to). A user may keep several instances of one family (chapter 17 §17.3).

## 12.7 Cycles

A node inside itself, directly or through others, is a cycle. Resolution completes: the back edge is reported and
skipped. A validator MUST report it.

## 12.8 Worked examples

**Wipeout XL.** The pristine holds the game and the `Play` entry with tile `17260`. *Single Player* and *Multiplayer*
are variants containing it. The FLAC soundtrack, the widescreen fix and the 200 fps mode are parts whose list begins
`{"INTO": [sp, mp]}`, the first two with `"RECOMMENDED": true` on it. Pick *Single Player*: the pristine and the
variant compose, the three grafts are offered, two are ticked. The launch composes the pristine, the variant, the
soundtrack, the widescreen fix; tick the 200 fps mode and it composes beside them (it writes byte ranges the
widescreen fix does not).

**Warcraft III.** Versions are a chain of nodes, each containing the one before and adding a delta. Every version
from 1.21b is a variant presenting both the Reign of Chaos tile (`802`) and The Frozen Throne tile (`803`,
`PARENTUID 802`); one entry per tile. A layer `{"REG": …, "WHEN": "%UID%==803"}` writes TFT's registry only when
TFT is launched. Both tiles share `%PackageUID%` `802`: one install folder, one set of saves.

**Age of Empires II.** *The Conquerors 1.0e* is a part that goes `INTO` *The Conquerors 1.0c*; *UserPatch 1.5* goes
`INTO` 1.0e, its `INTO` saying `"RECOMMENDED": true`: on those rows it is offered and pre-ticked. *Forgotten Empires*
goes `INTO` UserPatch and has its own entry and tile: its own card, each of its rows the chain 1.0c → 1.0e → UserPatch →
Forgotten Empires, never ticked on another row. UserPatch's installer
checkboxes are `bool` variables whose binary ops `Or` bits into one shared byte.

Next: [Ownership: `KEEP`](13-ownership.md).
