# 12 · Resolution

Resolving turns a node, an instance and a set of ticked grafts into a **plan**: the ordered content and edits, and
the folded registry, DLLs, environment, variables, entries and keeps. Every runner in the chain is resolved the same
way, from its own node.

## 12.1 Phases

1. **Variables for `WHEN`.** Collect what a layer `WHEN` may read: the instance's values and per-launch overrides, the
   built-ins (`%UID%`, `%PackageUID%`, the screen size, the runner's anchors), and the `DEFAULT`s of ungated `VARS`
   layers reached through ungated `NODE` layers, folded in list order (the ticked grafts' included, since they
   follow the variant). This phase reads the lists, not the expansion, so nothing is circular.
2. **Expansion.** Walk `LAYERS` (the root node's, then each ticked graft's, in order). Skip a layer whose `WHEN` does
   not hold. Expand a `NODE` layer in place, through its `TAKE` and `TARGET` (chapter 05). Check `ANY` and `NOT`
   (§12.5).
3. **Fold** (§12.2), then resolve every variable fully over the folded declarations (chapter 08 §8.5).

## 12.2 The fold

Walking the expansion bottom to top:

- **Per address, the latest writer wins.** A file, a registry value, an environment variable, a DLL policy, a
  variable declaration, an entry field.
- **`null` deletes**: a value or, for a `REG` key, the key and everything beneath it; for `ENV`, the name; for `DLL`,
  the policy; in an entry, the field.
- **An `EDIT` applies to the value beneath it** at its position, and a `DELTA` reconstructs from the content beneath
  it at its target (chapters 04 and 06).
- **`KEEP` folds per address** like everything else (chapter 13).
- **Entries fold per label and field**, in first-declared order (chapter 09 §9.2).

Order is position. A container decides where its parents go; a node never rises above the node that contains it
(I3); the variant's own layers sit above everything it contains; ticked grafts sit above the variant, in graft order.

An implementation SHOULD be able to show, per address, the winner and the layers it beat (the conflict view), and
SHOULD warn when a held or moved mention (chapter 05 §5.4) decided a winner.

*(VidyaGod: `Fold::Resolve`, `Fold::ResolveVars` in `fold.cpp`. The reference definition is the Python
`tools/gen6/resolve.py`; `tools/gen6/fold_gate.py` holds the C++ to it plan for plan.)*

## 12.3 What a resolution produces

| Part | Contents |
|------|----------|
| sequence | content and `EDIT` layers in fold order, each with its placed target, its package folder and its `TAKE` view |
| registry | `REG` values and created keys, per architecture |
| DLL, ENV, KEEP | the folded maps |
| declarations | the folded `VARS` |
| entries | the folded `EXEC` entries |
| order | the occurrences, in expansion order |
| events | held and moved mentions, cycles, missing nodes, unmet `ANY`s, hit `NOT`s |

A malformed layer (no single type key) is an error for the whole resolution: nothing is launched from it.

## 12.4 Variants and rows

Picking a row selects one variant node and one of its tiled entries (chapter 09 §9.4). The variant's node is the
root of the resolution. Nothing beneath it is a choice: playing a version runs that version, its options are its own
variables, and what it contains is simply part of it.

## 12.5 Checks and grafts

**`ANY [cid, …]`** is satisfied when an occurrence of one of its members is already in the expansion at this point
(beneath it). It never mounts, moves or fetches anything. **`NOT cid`** is satisfied when that node does not occur in
the final expansion. An unsatisfied check makes the resolution's root not applicable: a variant is blocked (listed,
with the reason) and a graft is not applied.

**A graft is a node whose list begins with `ANY`.** The leading `ANY` names what it applies onto; anyone may publish
one, and nothing it applies to names it (I9).

- **Offered.** Grafts are indexed under every member of their leading `ANY`. After resolving a row, every graft
  indexed under a node that occurs in it is offered. A graft that a node of the row itself contains (with a `NODE`
  layer) is part of the row, not an option.
- **Applied.** Ticked grafts are expanded after the variant, in the instance's graft order. Each is judged against
  the row with the grafts before it applied, so a graft may need another graft (its `ANY` names it). It applies when
  it is offered there and applying it leaves no requirement newly unmet (its `NOT` hits nothing, no `ANY` fails).
  One that does not apply is dropped from the launch and reported.
- **Default ticks.** A fresh instance ticks the offered grafts whose `RECOMMENDED` lists the row's tile, in rounds, so
  a graft always follows the graft it needs.
- **Default order.** A graft comes after every offered graft its `ANY` or `NOT` names; of those ready, the one naming
  the most recently listed goes first, then by `LABEL`, then CID. Related grafts stay together (mutually exclusive
  soundtracks, then patches in the order they build on each other).
- **Exclusive grafts.** Ticking a graft unticks the ticked grafts it cannot apply beside (either side's `NOT`),
  judged in list order. Mutually exclusive grafts behave as a choice of one.
- **Reordering** the graft list is the user's. A reorder that would put a graft before a graft its `ANY` needs makes
  it not apply; an implementation SHOULD refuse or flag such a reorder, naming both.
- **`SECTION`** (a node field) places a graft in the launch window's collapsible graft tree; the tree read depth
  first is the order grafts apply in.
- **An entry on a graft** (a mod loader such as SKSE or Forge) is one more way to run the row: the variant stays
  selected and its other grafts stay offered. A graft whose entry carries a tile has its own card (chapter 09 §9.4).
- **Graft declarations count in phase 1**, so a graft's own options gate its own layers.

Write the leading `ANY` against the lowest node the graft needs, usually a pristine, which never changes: a graft
written against a version is not offered on the next one.

## 12.6 The instance

The **instance** is the user's loadout for a family: the selected variant, the ticked grafts and their order, the
pinned runner chain, variable values (including drawn `POOL` values) and the owned state (chapter 13). It is local
data, never a node, and it is keyed by node CID; `LABEL` is never a key.

The cost of content addressing, stated once: re-minting a graft changes its CID, so a saved graft list that names a
CID the library no longer has was saved against an older package; the implementation falls back to the tile's
defaults. A user may keep several instances of one family (chapter 17 §17.3).

## 12.7 Cycles

A node inside itself, directly or through others, is a cycle. Resolution completes: the back edge is reported and
skipped. A validator MUST report it.

## 12.8 Worked examples

**Wipeout XL.** The pristine holds the game and the `Play` entry with tile `17260`. *Single Player* and *Multiplayer*
are variants containing it. The FLAC soundtrack, the widescreen fix and the 200 fps mode are grafts whose list begins
`{"ANY": [sp, mp]}`, the first two with `RECOMMENDED: ["17260"]`. Pick *Single Player*: the pristine and the variant
expand, the three grafts are offered, two are ticked. The launch folds the pristine, the variant, the soundtrack, the
widescreen fix; tick the 200 fps mode and it folds above them.

**Warcraft III.** Versions are a chain of nodes, each containing the one before and adding a delta. Every version
from 1.21b is a variant presenting both the Reign of Chaos tile (`802`) and The Frozen Throne tile (`803`,
`PARENTUID 802`); one entry per tile. A layer `{"REG": …, "WHEN": "%UID%==803"}` writes TFT's registry only when
TFT is launched. Both tiles share `%PackageUID%` `802`: one install folder, one set of saves.

**Age of Empires II.** *UserPatch 1.5* is a graft whose `ANY` names *The Conquerors, patch 1.0e*, recommended under
The Conquerors' tile (`13006`): on that row it is offered and pre-ticked. *Forgotten Empires 2.2* contains UserPatch
with a `NODE` layer, so on Forgotten Empires' rows UserPatch is part of the row, not an option. UserPatch's installer
checkboxes are `bool` variables whose binary ops `Or` bits into one shared byte.

Next: [Ownership: `KEEP`](13-ownership.md).
