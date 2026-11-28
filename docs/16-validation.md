# 16 · Validation

A validator checks a library before anything ships. **Errors** are things that cannot be right: a package with an
error SHOULD NOT be published, and the resolution errors of §16.2 refuse a launch. **Warnings** are things that are almost always a
mistake but can be intended. The format's failure mode is silence (a layer that quietly applies nothing), so most
checks exist to make a silent failure loud.

*(VidyaGod: `--validate-nodes` runs everything below over the library; `--audit-packages` additionally resolves and
lowers every launchable as a launch would, without mounting.)*

## 16.1 The node and its layers

Errors:

- a node file whose canonical form is over 256 KiB (§2.3), named by its file, with the split that brings it under;
- a node field other than `CID LABEL POS COMMENT VARIANT SECTION LAYERS` (a node-level `RECOMMENDED` names where it
  belongs: the entry, or a part's `INTO`); a field of the wrong type;
  `LAYERS` missing or not a list; an empty `VARIANT` or `SECTION`;
- a layer with no type key or with two; a field its type does not take;
- `WHEN`, `COMMENT`, `LABEL`, `SECTION` of the wrong type (`LABEL` and `SECTION` non-empty);
- a content file name that leaves the package (`..`, an absolute path); `SOURCE` not a string, `SIZE` not a number;
- a `TARGET` that is not a `FILES` address (content, `EDIT`, `NODE`);
- `NODE` not a CID string; `TAKE` not a list of addresses and `[address, name]` pairs; two `TAKE` selections landing on
  one address; two selections where one holds the other (chapter 05 §5.2);
- `EDIT` without ops or without a `TARGET` file; an op without `MODE`; an op field that is not a scalar; a `Delete` op
  with any field but `MODE`, `COMMENT` and `WHEN`; `IF_EXISTS` that is not a boolean, or `true` on a binary op; a `LineFlag`
  whose `KEY` is missing, blank or more than one line, or without `VALUE`, or whose literal `VALUE` is neither `1`
  nor `0`; `SECTION` on an op that is not a `ConfigWrite`, or one that is not a header line (a name in brackets, no
  bracket inside); `ABSENT` on an op that is not a `ConfigWrite`, `JsonSet` or `XmlSet`;
- a binary op with neither `ANCHOR` nor `OFFSET` (but `ReplaceAll`, which takes neither); `Replace` or `ReplaceAll`
  without `REPLACE`; `ReplaceAll` without `EXPECT`, or with a `REPLACE` of another length; `Poke` or `Or` without
  `VALUE`; `Cave` without `PAYLOAD`; `Or` with `EXPECT`;
- `REG` not a map of key paths; a key path not starting at a hive, or with an empty key name; a key that maps to neither
  values, `{}` nor `null`; a value whose data is an object or a list; `KEY` empty or not a string; `ARCH` missing, empty
  or not a list of `"32"`/`"64"`; a `Wow6432Node` key;
- `VARS` not `{KEY: declaration}`; `DEFAULT`, `COMMENT`, `WHEN`, `UI.SECTION` not strings; `UI` not an object; `EVAL`
  not a boolean; a `UI.CONTROL` that is not one of chapter 08's; a `key` option whose literal `DEFAULT` is not a key
  name (chapter 08 §8.3); a `MAP` that is not an object of strings, a `MAP` table with a `UI`, an `ELSE` that is not a
  string or not on a table;
- `ENV` or `DLL` values that are neither strings nor `null`;
- `EXEC` not a non-empty `{label: entry}`; an empty label; an entry that is not an object, or carrying a `LABEL` field;
  `RECOMMENDED` not a boolean; `HOST`, `EXE`, `WORKDIR`, `CONTENT_ROOT` not strings; `GUEST`, `ARGS` not string lists;
  `PREFIX_GENERATE`, `UNIFIED_RUNTIME` not booleans; an `INTO` layer's `RECOMMENDED` not a boolean;
- `GUEST_ROOTS` keys that are not `%Anchor%`s, values that are not paths, anchors spelled from each other in a
  circle, an anchor on a drive `DRIVES` does not lay out, an anchor neither on a drive nor spelled from a `%variable%`
  (`n64/x`, `/opt/x`); `DRIVES` values that are not paths;
- `TILE` fields of the wrong type; a `COVER` that is not `{FILE, SOURCE, SIZE}`, has no `FILE`, or whose file leaves the package;
- `KEEP` not a map; an address outside `FILES` and `REG`; a value that is not `true`, `false` or an options object; a
  kept `REG/HKLM/Software…` key without `ARCH`, an `ARCH` that is not a list of distinct `"32"`/`"64"`, or one on a
  `FILES` address; a wildcard outside the last segment of a `FILES` address, or on a folder;
- `INTO` not a non-empty list of CIDs; an `INTO` layer after any layer that is not one (`INTO` layers open the list);
- `ANY` and `NOT` (gone from the vocabulary: `INTO` and conditions replace them).

- the same node in two installed package folders (a node has one home, chapter 02 §2.7).

Warnings: a node with no layers.

## 16.2 References and resolution

Errors:

- a `NODE` layer naming a node that is not in the library;
- a cycle of `NODE` layers;
- a variant whose resolution meets a cycle, a missing node, or a part it contains going `INTO` what is not beneath it;
- an `INTO` naming a runner, or a node that is neither an end (`VARIANT`) nor a part;
- a `WHEN` naming a runner's CID (a package names a runner's capability, never its CID).

Warnings:

- an `INTO` layer naming no node in the library (a part for a version you do not have);
- a `WHEN` naming a node CID that is not in the library (it reads false);
- a part that is offered but whose `INTO` is not beneath it;
- a re-mention that raises a node's own layers over values written after its first place (chapter 05 §5.4: "raise").

## 16.3 Content

Errors:

- a `ZIP` with a compressed entry;
- a `DELTA` without `BASE_SOURCES`, or naming a file that is not in its launch (no `ZIP`, `FILE` or `DELTA` there carries that
  `SOURCE`; chapter 04 §4.1);
- a `DELTA` whose named bases, concatenated, are not the size its header says it was built over;
- a **covered** `EDIT`: beneath a later layer that replaces its file (chapter 06 §6.1);
- a `Delete` whose `TARGET` reaches what the user keeps: an owned address, anything inside a kept folder, a folder
  holding a kept address, a file a kept pattern matches (chapter 06 §6.2);
- two layers providing paths that differ only in case (chapter 04 §4.4);
- an entry's `EXE` that matches a content file only case-insensitively.

Warnings: a `DIR` that is not a runtime directory (it will not publish); an entry's `EXE` not found in the content;
a binary op without `EXPECT`.

## 16.4 Variants, tiles and grafts

Errors:

- a `VARIANT` with no effective entry to run, or presenting no tile;
- a card (a tile `UID`) with **two or more** recommended rows, or with several rows and **none** recommended (chapter 09
  §9.4);
- a graft with an entry that presents no tile, of its own or taken (it appears on no card, chapter 09 §9.4);
- a part that is also a version (a variant whose list begins with `INTO`) with no end in the library it goes into.

Warnings: `RECOMMENDED` on the `INTO` layer of a graft that has an entry (a row is never ticked); a recommended row
whose tile has no `TITLE`; a tile no variant presents.

## 16.5 Runners and paths

Errors:

- a drive letter in a package (outside a runner's `GUEST_ROOTS` and `DRIVES`);
- an anchor that does not start its path;
- a path written from an anchor when a more specific anchor contains it (`%Windows%/syswow64/x` for `%SysDir32%/x`).

Warnings:

- a runner entry with no `GUEST`; a prefix-generating runner without `GUEST_ROOTS`/`DRIVES`;
- an entry with no `HOST`; an entry whose platform no runner chain on this machine serves (for `custom`, no engine in
  its own package);
- runners that disagree on an anchor's guest path (packages are checked against the first).

## 16.6 Variables and conditions

Errors: a malformed `WHEN`; a layer `WHEN` reading a variable declared only in a gated layer; a reference to a variable
that is neither built in, an anchor, nor declared; a `%KEY:map:TABLE%` naming a table no `VARS` declares.

Warnings: an option (`UI`) referenced nowhere; two `key` options of one node and `SECTION` bound to the same key by
default.

## 16.7 Ownership

Errors: a `KEEP` of a whole registry hive; two `FILES` addresses of one node stored under the same name (their `NAME`s,
else their last segments, compared without case), since only one of them could be kept.

Warnings: a `KEEP` of a whole namespace (`FILES`); a `KEEP` `NAME` reserved for the instance's own state; a launch
option written into a user-owned file without a take-back (chapter 13 §13.4).

## 16.8 Scope

A validator checks one library (local packages, received stubs and runners together, since references cross
packages), optionally limited to some packages. Checks against content (case, compression, delta bases, covered
edits) need the content present; on a dehydrated package they are skipped, not passed.

Next: [Run modes, instances & the CLI](17-cli-and-run-modes.md).
