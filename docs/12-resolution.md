# 12 · Resolution: facts fold, choices don't

Resolving a launch turns the graph into an **ordered list of nodes** — the *mount* — from the user's **choices**.
Two sets, one rule:

- **selected** — what the user *chose*: the variant picked from the card, and the grafts ticked (including those
  the author pre-ticked with `TOGGLE: "on"`).
- **closure** — what a node is *made of*: everything reachable from it through the bare refs of `OVER`,
  transitively. This is what mounts.

| question | answered against |
|----------|------------------|
| what mounts | the **closure** of the selected variant, then of each selected graft |
| which grafts are offered / applicable | the **selected** set — never the closure |
| what runs | the effective entry of a **selected** node — the variant, or a ticked graft carrying one |
| what a node is part of | the nearest face **beneath** it |

**In `OVER`, a bare ref composes; a group or a `NOT` requires.** The closure walk follows bare refs and nothing
else: no gates, no groups, no exclusions, no toggles. Requirements — any-of groups, `NOT`s, and a bare ref that
names a *variant* — are evaluated in exactly one place: when a node is **offered** as a graft, against the
selected set. This is what makes "facts fold, choices don't" literally true: a variant's closure is a pure
function of the graph, and every choice is a variant or a graft.

## 12.1 What resolution produces

`Closure(graph, nodeId) → [nodeId, …]` — every node reachable from `nodeId` through bare `OVER` refs, topologically
ordered **requirements before dependants**, the node itself **last** (highest overlay priority). A ref missing from
the graph is reported. Cycles are reported and broken (I2).

`Selected(graph, variantId, ticks) → set` — the fixpoint of §12.3.

`Mount(graph, variantId, ticks, precedence) → [nodeId, …]` — `Closure(variant)`, then for each selected graft in
instance precedence, `Closure(graft)` minus what is already mounted, graft last.

```
function Closure(graph, id):
    order = []; visited = {}
    function emit(n):
        if n in visited: return
        visited.add(n)
        for r in bareRefs(graph[n].OVER): emit(r)      # composition only; a missing r is reported
        order.append(n)
    emit(id)
    return order                                       # requirements before dependants, id last
```

`TOGGLE` has no meaning inside a closure. A node reached through a bare ref is mounted, full stop; the pristine
cannot know its optional pieces, so an optional piece is never *inside* a closure — it is a graft over the
variants it applies to (§12.3). A validator warns about a `TOGGLE`, a group or a `NOT` on a node that nothing
offers.

## 12.2 Order = priority

The emitted order is the overlay stacking order: earlier = lower priority, later = higher; a node is always
emitted after everything it is `OVER`, so **a dependant wins over its requirements**; the variant sits on top of its
own closure; grafts sit above that. Among a node's own refs, list order is the tie-break (later = higher). Within
one node the engine applies kinds in phases — VFS mount → binary patches → post-VFS file edits → registry — so a
pluripotent node needs no internal order.

## 12.3 Grafts — the offered set

A **graft**, relative to a selected variant, is a node that has the variant's face's identity (the same UID, own or
inherited — [ch. 3 §3.2](03-roles.md)), is **not** a variant, and is **not in the selected variant's closure**.
Variants are never grafts (they are picked, not ticked); substance is never a graft (it has no title). A node is
a graft for one variant and made-of for another as the graph dictates: *Widescreen* is offered on *Single Player*
and simply part of *Widescreen Edition*, which is `OVER` it.

A graft is **applicable** iff every requirement in its `OVER` holds against the **selected set**:

| entry | it is | holds when |
|-------|-------|------------|
| a bare ref to a **variant** | a requirement | that variant is the selection — a mod `OVER [640]` is not for you on 659 |
| a bare ref to **anything else** | composition | always — the graft is *made of* it and brings it beneath itself when ticked: tex-hd brings tex, a fix brings its library. What it brings counts as selected from then on (for `NOT`s, and for other grafts), and **what it brings is judged too**: a graft made of a graft `OVER [640]` requires 640 — requirements are transitive over composition |
| an any-of group `[a, b]` | a requirement | some member is selected |
| `{ "NOT": x }` | a requirement | `x` is *not* selected — and, symmetrically, no selected node's `NOT` names this graft or anything it brings |

**Selecting a variant selects exactly that node.** Nothing beneath it is selected, however much it inherits:
playing 1.16.5 offers mods `OVER [1.16.5]`, never mods `OVER [1.16.4]`, even though 1.16.4 is in the closure and
1.16.5 inherits its entry from it. This is the boundary that keeps a version chain from being a mod chain.

**A graft may carry an entry.** A mod loader (Forge `OVER [1.16.5]`, SKSE `OVER [[640, 659]]`) is ticked like any
graft; the variant stays selected, so its mods stay offered; and the picker gains the graft's entry as a way to
run ("Forge" beside "Play"). There is no such thing as a launchable graft that "selects what it inherits through":
what you picked is what is selected.

**Fixpoint with retraction.** Ticking a graft brings what it is made of and can trip another's `NOT`; the
selected set is the fixpoint of "selected = variant ∪ { ticked grafts applicable against the *other* selected
nodes } ∪ what those bring", re-derived in candidate order until it settles — so a graft a later selection excludes
*leaves*, and whatever it brought leaves with it. Between two ticked grafts that exclude each other the first in
candidate order wins; a UI unticks the loser at tick time. An implementation reports, per graft, *applicable*,
*selected*, and — when blocked — the first unsatisfied requirement ("needs version X") or the excluder ("excluded
by Y").

**Pre-ticked.** `TOGGLE: "on"` on a graft means the author ships it ticked (the FLAC soundtrack, the widescreen
fix); `TOGGLE: "off"` or absent means offered, unticked. The user's tick is authoritative in both directions.

**Grafts on runners.** A node `OVER` a runner that is not a variant is offered when that runner is in the chain;
the runner plays the part of the face. Ticked, it mounts above the runner's build.

**Scope.** Candidates are nodes of the local library; a received, un-installed stub is browsed, never grafted.

## 12.4 The graft order

Selected applicable grafts mount above the variant's closure in **instance precedence** (higher = later = wins at
a conflict), ties by key so an untouched instance is reproducible. Each graft's own closure is emitted beneath it
(its substance, its private ancestors); nodes already in the mount are never repeated. A graft's identity-bearing
requirements are selected by construction, so descending into them never pulls a second copy of the game.

## 12.5 Conflicts

Conflicts are **detected, never declared by pairs**. Two grafts conflict when they provide the **same target path
with different content** (layers), **overlapping ranges on one file** (patches), or **the same key with a different
value** (file edits, registry, DLL overrides). Identical bytes never conflict; an appended line never conflicts;
`VARS` override by closure order *by design*. A graft overlapping the canonical is not a conflict — that is the
point. Archives are opaque (authors are encouraged to unpack). Resolution is the instance's: **per-mod
precedence** in bulk, **per-file winners** as sparse exceptions.

## 12.6 The instance

The **instance** is the loadout — local configuration, not a node: the selected variant, the ticks, the entry
`(node, label)`, precedence, winners, variable overrides, its own saves. Permutations are instances. Every piece
of instance state is keyed by **node CID**, and `LABEL` is never a key. The honest cost of content addressing,
stated once: **a re-mint of a node resets its tick and precedence to the author's defaults**, and a mod update
(a new CID) is unticked until re-ticked; a graft `OVER` the old CID is unsatisfied until a node names the new one.

## 12.7 Cycles

Bare `OVER` refs MUST be acyclic. If a cycle exists, resolution still completes: the closure walk detects the
back-edge, reports it, and skips that edge. A validator MUST report cycles as errors ([ch. 15](15-validation.md)).

## 12.8 Worked examples

**Wipeout XL.** One tile on the pristine; *Single Player* and *Multiplayer* are variants over it. The FLAC
soundtrack, the widescreen fix and the 200fps mode are three grafts `OVER [["sp", "mp"]]`, the first two with
`TOGGLE: "on"`. Pick *Single Player* → closure `{pristine, sp}` → offered: soundtrack ✓ ticked, widescreen ✓
ticked, 200fps offered → mount: pristine → sp → soundtrack → widescreen (its patches over the composed exe, its
ini edit post-VFS, its 93 knobs in the sheet). Tick 200fps → it mounts above, its variable node pulled in beneath
as substance. Pick *Multiplayer* → same grafts, same rule.

**Minecraft.** Pick *1.16.5* → the delta chain mounts beneath (bytes); *Play* is 1.16.5's effective entry,
inherited from the last version that declared one; a mod `OVER [1.12.2]` is not offered. Tick *Forge 36.2*
(`OVER [1.16.5]`, an entry, no `VARIANT`) → 1.16.5 stays selected, Forge is selected, *Biomes O' Plenty*
`OVER [1.16.5, forge-36]` becomes applicable; run *Forge* or *Play*.

**Skyrim.** Pick *1.6.640* → `{640}`. Offered: `skse OVER [[640, 659]]`, `tex OVER [[640, 659]]`, `tex-hd OVER
[tex]` (made of tex: ticking it brings tex), `quest OVER [640, skse, NOT old-quest]` (made of skse: ticking it
brings skse, and *SKSE* appears as a way to run), `ab-patch OVER [modA, modB, 640]` (brings both), `old-quest`.
A mod `OVER [659]` is not offered: 659 is a variant and it is not the selection. Tick old-quest → quest unticks
(its `NOT`); tick quest → old-quest unticks. plugins.txt is the composed file: each ticked mod's `FILEEDITS`
*AppendLine* in precedence order — no generator.

**Age of Empires II.** One card, three faces: the Age of Kings tile on the pristine; The Conquerors' tile on its
base zip `OVER` the pristine; Forgotten Empires' `OVER` that. Pick *The Conquerors* (a variant over its face) →
its closure holds the AoK pristine and the AoK registry but not AoK's no-CD patch (that is part of the *Age of
Kings* variant, a sibling). An AoK mod `OVER [aok-variant]` is not offered here: `aok-variant` is not selected.

Next: [The runtime model](13-runtime-model.md).
