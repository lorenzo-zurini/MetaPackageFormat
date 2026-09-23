# 12 · Resolution: selection ≠ closure

Resolving a launch means turning the graph into an **ordered list of nodes** — the *mount* — from the user's
**choices**. Two sets, two rules, and everything the format promises about mods follows from keeping them apart:

- **selected** — what the user *chose*: the launchable picked from the card, one member per any-of group, the
  grafts ticked (and those the author pre-ticked with `TOGGLE: "on"`).
- **closure** — what the selected nodes are *made of*: everything reachable from them through plain `OVER`
  entries, transitively. This is what mounts.

| question | answered against |
|----------|------------------|
| what mounts | the **closure** of the selected set |
| which grafts are offered / applicable | the **selected** set — never the closure |
| what runs | an entrypoint of a **selected** node — never one under it |

A plain `OVER` entry is therefore *never a choice and never a branch point*. `1.16.5 OVER [1.16.4]` puts 1.16.4's
bytes under you; a mod `OVER [1.16.4]` is a sibling branch off a node you did not choose — not a mod for you — and
1.16.4's entrypoint is not offered. *Nothing travels along the chain.*

## 12.1 What resolution produces

`ResolveNodeOrder(graph, launchNodeId, toggles) → [nodeId, …]` — the launchable's own closure, topologically
ordered **requirements before dependants**, the launch node **last** (highest overlay priority). `toggles` is
`nodeKey → bool`: the user's explicit choices, keyed by the node's key (its CID; a label resolves too). Any
reference missing from the graph is reported.

`ResolveGraftOrder(graph, launchNodeId, toggles, baseOrder, precedence) → [nodeId, …]` — the nodes that mount
**above** that closure: every selected, applicable graft with its own substance beneath it, in instance
precedence. The launch mounts `baseOrder` then this, in that order.

## 12.2 The closure

```
function ResolveNodeOrder(graph, launchId, toggles):
    enabled = { launchId }; frontier = [ launchId ]; pending = []
    loop:
        while frontier not empty:
            node = graph[frontier.popFront()]
            plain = [ e for e in node.OVER if e is a ref ]              # any-of groups are DEFERRED (below)
            pending += [ e for e in node.OVER if e is a group ]
            for pid in stableSort(plain, explicitlyOn(toggles) first):  # an explicit choice is kept before a
                consider(pid)                                           # conflicting DEFAULT-on sibling
        if pending empty: break
        group = pending.popFront()
        if some member already in enabled: continue                     # kept through ANY other route wins
        for m in explicitly-toggled-on members, then the rest in list order:
            if consider(m): break                                       # the first the gate KEEPS; then walk it
        else: report the group as missing                               # never a mount with NO member, silently
    function consider(pid) -> kept:
        p = graph[pid]; if missing or unlowerable: report; return false
        if p.TOGGLE present and not (toggles[pid] if present else p.TOGGLE == "on"): return false
        if p excludes an enabled node, or an enabled node excludes p: return false   # NOT: first-kept wins
        enabled.add(pid); frontier.pushBack(pid); return true
    return postOrderDFS(launchId, following OVER in list order, restricted to enabled)   # cycles broken, warned
```

- **`TOGGLE` inside the closure** is an optional module: present ⇒ toggleable, value = the author's default; the
  user's toggles are authoritative in both directions. An off node is not descended into, so its private
  requirements drop with it (the *hierarchy gate*).
- **An any-of group** is a choice. It is resolved only after every plain requirement has been walked, so a member
  already kept through another route satisfies it without a second pick; with nothing kept, an explicit toggle
  picks, else the first present member — deterministic, and never two. A member the gate *refuses* (toggled off,
  excluded by a kept node) is not the pick: the next is tried, and a group no member satisfies is reported as
  missing. On a *launchable* this is the version selector the picker shows.
- **`NOT`** is one-sided in the file and symmetric in effect: a candidate is dropped if it excludes a kept node
  **or** a kept node excludes it; explicitly toggled-on candidates are considered first, so an explicit choice
  beats a conflicting default.
- **The gating map must be the same on every walk.** Any pass that re-walks a closure MUST use the user's
  toggles, never a map reconstructed from a default-gated walk (which never visits an off-by-default node).

## 12.3 Order = priority

The emitted order is the overlay stacking order: earlier = lower priority, later = higher; a node is always
emitted after everything it is `OVER`, so **a dependant wins over its requirements**; the launchable sits on top of
its own closure; grafts sit above that. Among a node's own entries, list order is the tie-break (later = higher).
Within one node the engine applies kinds in phases — VFS mount → binary patches → post-VFS file edits → registry
— so a pluripotent node needs no internal order.

## 12.4 Grafts — the offered set

A **graft** is a node of the launchable's title (same UID, own or inherited — [ch. 3 §3.2](03-roles.md)) that is
**not part of the launchable's own composition** (not reachable from it through `OVER`) and is `OVER` something.
Launchables and runners are never grafts (they are variants); substance is never a graft (it has no title).

A graft is **applicable** iff every requirement holds against the **selected set**:

| requirement | holds when |
|-------------|------------|
| a plain ref to a node **with identity** | that node is *selected* — the launchable, or a selected graft |
| a plain ref to a node **without identity** (substance) | always — substance is satisfied by *mounting*, it is not a choice |
| an any-of group | some member holds by the rules above |
| `{ "NOT": x }` | `x` is *not* selected — and, symmetrically, no *selected* node's `NOT` names this graft |

**Selection follows identity.** Selecting a node selects the nodes it takes its identity from: a tile-carrying
launchable selects only itself (what is under it is made-of); a tile-less launchable graft (Forge `OVER [1.16.5]`,
SKSE `OVER [[640, 659]]`) selects what it inherits through — a plain entry as-is, a group by choice. So picking
Forge is "1.16.5 with Forge", and 1.16.5's mods are offered alongside Forge's.

**Fixpoint with retraction.** Ticking a graft can make another applicable (`tex` → `tex-hd OVER [tex]`) and can
trip another's `NOT`; the selected set is the fixpoint of "selected = launchable ∪ { ticked grafts applicable
against the *other* selected nodes }", re-derived in candidate order until it settles — so a graft that a later
selection excludes *leaves*, and whatever stood on it leaves with it (never `hd` mounted next to the node that
excludes its base). Between two ticked grafts that exclude each other the first in candidate order wins; a UI
unticks the loser at tick time. An implementation reports, per graft, *applicable*, *selected*, and — when
blocked — the first unsatisfied requirement (the UI shows "needs X") or the excluder ("excluded by Y").

**Scope.** Candidates are the hydrated, localised library only; a received CATALOG stub never grafts.

## 12.5 The graft order

Selected applicable grafts mount above the base closure in **instance precedence** (higher = later = wins at a
conflict), ties by label then key, so an untouched instance is reproducible *and* survives a re-mint (a CID
changes with every edit; a label does not). Each graft's own closure is emitted beneath it
(its substance, its private ancestors); nodes already in the base mount or already emitted are never repeated.
A graft's identity-bearing requirements are selected by construction, so descending into them never pulls a
second copy of the game.

## 12.6 Conflicts

Conflicts are **detected, never declared by pairs**. Two grafts conflict when they provide the **same target path
with different content** (layers), **overlapping ranges on one file** (patches), or **the same key with a different
value** (file edits, registry, DLL overrides). Identical bytes never conflict; an appended line never conflicts;
`VARS` override by closure order *by design*. A graft overlapping the canonical is not a conflict — that is the
point. Archives are opaque (authors are encouraged to unpack). Resolution is the instance's: **per-mod
precedence** in bulk, **per-file winners** as sparse exceptions.

## 12.7 The instance

The **instance** is the loadout — local configuration, not a node: the selected set (keyed by node CID; author
defaults from `TOGGLE`), the entrypoint `(node, label)`, precedence, winners, variable overrides, its own saves.
Permutations are instances. A mod update is a new CID: a graft `OVER` the old CID becomes unsatisfied and unticks
until a node names the new one — re-selection is the honest cost of CID-keyed compatibility.

## 12.8 Cycles

`OVER` MUST be acyclic. If a cycle exists, resolution still completes: the order pass detects the back-edge,
reports it, and skips that edge. A validator MUST report cycles as errors ([ch. 15](15-validation.md)).

## 12.9 Worked examples

**Wipeout XL.** Two launchables `OVER` one pristine; the widescreen fix is one graft `OVER [["sp", "mp"]]` with
`TOGGLE: "on"`. Pick *Single Player* → selected `{sp}` → the graft is applicable (sp ∈ its group) and pre-ticked →
mount: pristine → sp → widescreen (its patches over the composed exe, its ini edit post-VFS, its 93 knobs in the
sheet). Pick *Multiplayer* → same graft, same rule. The 200fps mode is a second graft, `TOGGLE` absent: offered,
unticked; ticked by the user it mounts above, its variable node pulled in beneath as substance.

**Minecraft.** Pick *1.16.5* → the delta chain mounts beneath (bytes); *Play* runs 1.16.5's own entry; a mod
`OVER [1.12.2]` is not offered. Pick *Forge 36.2* (`OVER [1.16.5]`, no tile) → selects Forge and 1.16.5 → *Biomes
O' Plenty* `OVER [1.16.5, forge-36]` becomes applicable; run *Forge* or 1.16.5's *Play*.

**Skyrim.** Pick *SKSE* (`OVER [[640, 659]]`) → choose 640 → `{skse, 640}`. Offered: `tex OVER [[640, 659]]` ✓,
`quest OVER [640, skse, NOT old-quest]` ✓, `old-quest` ✓, `tex-hd OVER [tex]` (needs tex), `ab-patch OVER [modA,
modB, 640]` (needs modA, modB). Tick tex → tex-hd offered. Tick old-quest → quest unticks (its `NOT`); tick quest →
old-quest unticks. plugins.txt is the composed file: each ticked mod's `FILEEDITS` *AppendLine* in precedence
order — no generator.

Next: [The runtime model](13-runtime-model.md).
