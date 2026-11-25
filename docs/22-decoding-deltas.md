# 22 · Decoding deltas (DRAFT — not normative)

> **Status: design draft, 2026-11-25.** Nothing here is implemented in the engine. It records the decisions taken and
> the evidence measured while prototyping (`~/Storage/tmp/vacuum`). Chapters 04 (content layers) and 15 (packages and
> sharing) stay authoritative until this chapter is promoted. Where this chapter says MUST, it means "MUST once adopted".

Today a version is a `.vgdelta` over the **bytes** of the version beneath it. A game that keeps its data in compressed
archives (MPQ, CASC, zip/`.pak`, Unity bundles, BSA) changes a few files per update, but the archive's compressed bytes
change wholesale, so the delta stores them again. This chapter splits content down to its pieces, decodes them where
that is exact, and deltas **pieces against pieces**.

## 22.1 Goals and non-goals

- **Smaller histories.** Measured: Warcraft III's 8 versions take 2.10× the latest build today, 1.54× with decoded
  pieces (22.10).
- **The anchor stays shipped bytes.** The first version of anything is stored exactly as shipped: no decoding is needed
  to play it.
- **A dumb reader.** The runtime copies ranges, re-encodes units, and copies ranges. Everything clever happens at mint.
- **No caches.** A read re-encodes only the units it touches (22.7). Nothing is rebuilt whole, ever.
- **Byte-for-byte.** Every rebuilt file is verified identical to the shipped one at mint. A piece that cannot be rebuilt
  exactly stays stored as shipped.
- **Append-only.** A new version never re-mints an old one. No reverse deltas.
- Non-goal: decoding for its own sake. The packer decodes a piece only where it pays (22.6).

## 22.2 Terms

| Term | Meaning |
|------|---------|
| **anchor** | a directory, logically; on disk and in transport a canonical STORE zip of it (22.3) |
| **piece** | a unit of content a delta reasons about: a whole file, or an inner file of a described archive (`War3.mpq:Units\HumanUnitFunc.txt`) |
| **stored form** | a piece's bytes as shipped (compressed, decrypted) |
| **decoded form** | a piece's content after its codec chain |
| **unit** | the smallest independently re-encodable chunk of a piece: an MPQ sector, a BLTE chunk, a deflate block (22.7) |
| **recipe** | how a decoded unit re-encodes to its exact shipped bytes: a codec and its parameters |
| **skeleton** | every byte of a described archive that is not a piece: headers, tables, sector tables, gaps |
| **unit map** | where each unit of a piece sits in the shipped file, its length, and its recipe; part of the skeleton |
| **description** | a declarative, bidirectional layout of a container format (22.5) |
| **registry** | the closed set of named functions a description may call: hashes, ciphers, codecs (22.5.3) |
| **layer** | one encoding step wrapped around content (a container, a codec, a cipher); real files nest them (22.11) |
| **decisions** | what a layer's encoder chose (literal runs, matches, block headers), in NEUTRAL form (22.11.2) |
| **family** | a parametric reference encoder in the registry (LZ77 hash-chain parser, …); **knobs** are its parameters |
| **corrections** | where a layer's real decisions differ from its family's proposal under the fitted knobs (22.11.3) |
| **read program** | the flat COPY / ADD / GENERATE program a reader compiles from all layers of a piece (22.12) |

## 22.3 The anchor

- The logical anchor is a **directory**. Deltas target directories, and the runtime mounts directories.
- On disk the anchor stays a **STORE zip**; directories are **virtual**, built by VidyaGodFS from the anchor and the
  chain of tree deltas at runtime. (User decision 2026-11-25: "still zips on disk, the folders are virtual".)
- The zip is a pure function of the directory: one **canonical serialization**, defined by this spec (members sorted by
  path bytes, fixed header fields, no timestamps, no extra fields but zip64 where required, no data descriptors).
  Anyone zipping the same directory gets the same bytes, so the same CID. Zip stops being a format we special-case and
  becomes the way a directory travels.
- **Manifest:** none of its own. The anchor's listing is its zip's central directory, and a tree delta carries its
  target's listing (22.4). The node stays an anchor CID plus a delta chain: a manifest cannot live in a node, which is
  capped at 256 KiB.
- Open: canonical-zip details for empty directories, symlinks, executable bits, case conflicts (the case-conflict
  resolver applies).

## 22.4 Tree deltas

A tree delta rebuilds a directory from the directory beneath it. For each path of the target it says one of:

| Kind | Rebuild |
|------|---------|
| **same** | the bytes of a base piece, by reference (any path, any earlier version in the chain: reverts, renames and moves are free) |
| **patch** | a binary delta of this piece against a base piece |
| **new** | new bytes |

- Every path is checked against the target's listing, which the tree delta carries.
- Pieces are **paired** with their predecessor by name. Archive inner files use `container path:inner name`
  (`War3.mpq:Units\HumanUnitFunc.txt`, agreed 2026-11-25).
- A piece that cannot be paired by name (CASC entries are addressed by hash and have no name) is delta'd against the
  **whole previous version**, exactly as today's deltas are, so a tree delta is never worse than today.
- A tree delta is ONE `.vgdelta` over a multi-base (`BASE_SOURCES`): skeleton first, then changed pieces, then new ones.
- References are by location (base CID + path, or base CID + range), never by a global content hash, so every reference
  stays verifiable through the CID chain. Content hashes are an authoring cache only.

## 22.5 Descriptions

### 22.5.1 Principle

A container format is described as **data**: a JSON document listing its structures, fields and placements. The same
description is used in both directions:

- **read** a shipped archive into a tree (structures, arrays, blobs), and
- **write** the tree back to bytes, recomputing every derived field.

The acceptance test for a description is a byte-identical round trip of real archives from the library.

### 22.5.2 Field forms

| Form | Meaning |
|------|---------|
| `"u32"` | a value read from the bytes (data) |
| `"u32 =EXPR"` | a **computed** value: verified on read, recomputed on write, never copied |
| `{"TYPE", "AT", "ARRAY", "AT_EACH", "IF", "SIZE", "FILL", "CRYPT"}` | placement, repetition, condition, size, encryption |
| `{"LET": EXPR}` | a value derived from what was read, not stored; provisional during the read |
| `{"LET": EXPR, "KEEP": true}` | a fact recovered from the bytes (e.g. a base encryption key): data, kept in the tree |

- Expressions are a closed, side-effect-free subset of arithmetic and comparison over fields, plus `at`, `end`, `len`
  and the registry's functions. There are no loops and no user-defined functions.
- A free name resolves to a field or a `LET` of the enclosing structures, never to anything else.
- An included format refers to its own top by a lexical name, never `root` (the map format includes the MPQ format).
- Encrypted blobs decrypt lazily: their key may depend on the whole archive (MPQ keys by file name).
- **Placement is policy, not description.** "keep" reproduces the original layout; "pack" lays a changed archive out
  fresh, which is what a mod merge needs.

### 22.5.3 The registry

The only code a description may call: format hashes (`mpq.hash`), ciphers (`mpq.decrypt`/`encrypt`), recovery helpers
(`mpq.detect_key`), checksums (`crc32`), and codecs.

A codec is **round-trip** (it has an exact re-encode for a recipe) or **decode-only** (it can be read through, never
rebuilt: e.g. Oodle, whose encoder is proprietary).

A procedural oddity (name-keyed encryption, a seeded obfuscation) becomes a new named registry function, or, until
the registry has one, a tier-3 function carried in the package (22.5.5).

### 22.5.4 Prototype evidence

Descriptions written: `mpq.json` (MPQ v0, with encryption, offset-dependent keys and sector tables), `zip.json` (PKZIP
with zip64 and data descriptors), `w3x.json` (the Warcraft III map wrapper, which includes `mpq.json`).

- **Round trips, byte-identical:**
  - war3.mpq (436 MB, 136,674 computed fields, 0 mismatches);
  - War3x.mpq, War3xLocal.mpq and War3Patch.mpq;
  - a `.w3x` map (100% described);
  - the 1.16 GB WC3 package zip (100%).
- **Write direction:** a file replaced inside War3Patch.mpq **and** inside the encrypted, offset-keyed War3xLocal.mpq,
  with the archive laid out fresh: it reads back correctly, and every other file is identical.

### 22.5.5 Expressiveness: four tiers (USER 2026-11-25: "expressive enough to express every conceivable archive format")

A closed list of declarative constructs cannot express every format: some are genuinely procedural (custom ciphers,
executable packers, self-describing scripts). The language therefore has four tiers. All four are DATA, and one dumb
reader interprets them all ("smart data, dumb reader").

| Tier | What | Covers |
|------|------|--------|
| 1. **Declarative core** | fields, placement, arrays, conditions, computed fields, alignment, ordering (22.5.2) | most container layouts |
| 2. **Codec families + knobs** | a registry family named with its parameters (22.11): LZ77 hash-chain parsers, Huffman, range/arithmetic coders, LZW, BWT, block ciphers, hashes | nearly every codec ever shipped, including "unknown" in-house ones |
| 3. **Carried functions** | a tiny sandboxed, deterministic, total (fuel-bounded), memory-bounded bytecode function stored in the package: bytes in, bytes out, no I/O | the truly procedural remainder; promoted into the registry once common |
| 4. **Corrections** | where the spec under-describes an encoder, the differences are stored (22.11.3) | an INCOMPLETE spec is still bit-exact; it only costs bytes |

- Authoring is case by case, by reverse engineering, at mint (USER 2026-11-25). The fitter (22.11.3) is the authoring
  tool: its corrections show exactly where a spec is incomplete. A quirk found there becomes a knob, and its
  corrections drop to zero.
- The acceptance test does not change: re-serialize an unchanged archive byte-identical.
- Open: tier 3's machine (a WASM subset or our own VM), and its fuel and memory limits.

## 22.6 Pieces: stored or decoded, chosen per piece

- The packer computes both forms of each piece and keeps the one whose delta is smaller, but uses the decoded form
  only where every unit re-encodes exactly (22.7).
- Exactness is decided per unit, at mint, by re-encoding and comparing. It is never assumed.
- A piece and its base are always compared in the **same form**: decoded against decoded, stored against stored.
  Mixing them matches nothing.

## 22.7 Units, recipes and the runtime

### 22.7.1 Recipes are read from the stream

- **zlib:** the stream header gives the window (CMF's CINFO) and the level band (FLG's FLEVEL); the exact level is
  confirmed within the band. Measured:
  - WC3 1.21b: level 6, window 32 KiB.
  - WC3 1.27a+: level 6, window sized to the 4 KiB sector (512 B – 4 KiB).
  - CASC BLTE: level 6 (or 9).
  - KCD2 `.pak` members: raw deflate, level 6, memLevel 9.

  100% exact across every sample tested (148k MPQ sectors, every CASC entry, 22 KCD2 members).
- **LZMA (Unity):** exact with the old LZMA SDK encoder (not 23.01). Settings: normal mode, fast bytes 64, bt4, 4 hash
  bytes, dictionary from the stream's props, `lc3 lp0 pb2`, end marker. 6/6 blocks exact.
- **Storm Huffman + ADPCM (WAV):** ADPCM is lossy, so these pieces stay stored.

### 22.7.2 Units and checkpoints

- **Chunked formats** (MPQ sectors, BLTE chunks, Unity LZ4 blocks, zip members of a few KB): each unit re-encodes
  independently.
- **Single streams** (a deflate member, a `.gz`): the skeleton's unit map holds an **encoder checkpoint** per deflate
  block: (bit offset in the shipped stream, offset in the decoded content).
  - A fresh encoder primed with the 32 KiB of decoded content before the checkpoint reproduces the rest bit-exactly.
    Measured: 803/803 block boundaries across 22 KCD2 members.
  - A checkpoint costs two integers; the skeleton grows about 0.05%.
- **LZMA single streams:** proven with token replay (22.11.1). A checkpoint is the coding state only: probabilities,
  four repeat distances, the state, and the range coder's registers. That is 16 KB for `lc3 lp0 pb2`, with no match
  finder and no prices. Checkpoints are DERIVED by one replay pass and live in RAM; they are never stored.
  - Measured on Unity blocks (16 MiB decoded each): 1,600 random 4 KiB and 64 KiB reads of the shipped bytes were all
    bit-exact. p50 2.5–2.9 ms, p99 ≤ 6.8 ms; a whole-block replay runs at 240–370 MB/s decoded.
  - **RAM (measured, 6 blocks):** space checkpoints by SHIPPED bytes, not decoded ones. These blocks compress 16:1, so
    a checkpoint every 64 KiB of decoded data takes 4× the shipped size. Every 64 KiB SHIPPED, each checkpoint
    zlib-compressed on its own (7–8.4 KB):
    - RAM: 11–14% of the shipped size of the LZMA pieces being read.
    - Reads: p50 2.9–3.2 ms, p99 ≤ 8.4 ms.

    | Option | RAM (% of shipped) | Reads |
    |---|---|---|
    | Every 16 KiB shipped | — | no faster: a 64 KiB read replays its own ~1 MiB of decoded data |
    | Every 256 KiB shipped | 6% uncompressed | p50 7–7.7 ms |
    | Keyframes (compressed against the previous checkpoint, 3.0–3.7 KB) | about 7% | — |
  - Deriving the checkpoints is one sequential replay of the stream (about 50 ms for a 16 MiB block), done the first
    time any part of it is read.

### 22.7.3 Reading (USER, 2026-11-25: "no cache"; "the skeleton map is the cache")

A read of bytes X..Y of a shipped archive:

1. Look up, in the unit map, the units covering X..Y.
2. Re-encode exactly those units from the decoded content with their recipes; skeleton bytes are copies.
3. Return the bytes. No unit larger than the mint-time bound (e.g. 256 KiB of decoded input) is ever re-encoded per read.

The RAM block cache holds recently produced units. There is no disk cache and no first-run reconstruction.

Derived state (LZMA checkpoints, a piece's compiled read program, 22.12) lives in RAM. It is built when a piece is
first touched, bounded, and evicted like cache blocks. It is never written to disk and never transported.

### 22.7.4 The skeleton lives in the delta (agreed 2026-11-25)

The skeleton (tables, headers, sector tables, the unit map, recovered facts such as keys) is stored in the tree delta,
not recomputed at runtime from the description. It is small (WC3: 0.7–44 MiB per version, most of it sector tables)
and reading it is a copy. Delta'd against the previous version's skeleton.

## 22.8 Mods (later)

With archives described, a mod overrides single inner files.

- The runtime serves a merged archive: the untouched compressed members copied as they are, the replaced files stored
  uncompressed, and a rewritten index ("pack" placement).
- No compressor is needed.
- Conflicts are per inner file.

## 22.9 Open questions

- The binary layout of a tree delta (listing, kinds, piece references) and the node keys that name it.
- Tier-3 carried functions: the machine and its limits (22.5.5).
- The neutral decision format, and how a description names a layer's framing over it (22.11.2).
- Copy-class layers resolved per read without a resident composed program (map the read range down layer by layer
  through each layer's skeleton index): designed, not yet measured (22.12).
- Token replay for Oodle-class formats: it needs only their bitstream writer (the open decoders define it), not the
  proprietary encoder's search. Unproven.
- Pairing unnamed pieces better than "whole previous version" (CASC root/encoding files give names).
- CASC: newer `data.###` archives open with empty 30-byte entries and have free space between entries. A splitter
  accepts an entry only when a `BLTE` blob follows its header, otherwise it resyncs. (A splitter that trusted a misread
  size field once swallowed 690 MiB.)
- Engine code: the description interpreter in C++ (a tiny, total, deterministic evaluator, fuzzed), and codecs from
  StormLib (MIT), zlib and the LZMA SDK.
- The PUBLISHED node key (decided 2026-11-25, specced separately in ch. 02/16 by the update-UX work) supplies "newer vs
  older" for the version tree.

## 22.10 Measurements (2026-11-25)

Library-wide (75 Steam games with more than one build, raw bytes):

| Format | All builds ÷ latest build |
|--------|---------------------------|
| today | 1.45× |
| whole-file dedup + per-file deltas, no decoding | 1.30× |

- Cross-depot duplication is large: Skyrim SE 2.34× → 1.45×; XIII 1.00× → 0.20×.
- In 6 Unity games, the upper bound of the decoding win is ~25% of history churn: TUNIC 94%, Disco Elysium 61%,
  Outer Wilds 0%.

Warcraft III, 8 versions, every cost a real `.vgdelta` from today's generator:

| Version | Today | Pieces stored | **Pieces decoded** |
|---------|-------|---------------|--------------------|
| 1.27a | 351 MB | 284 MiB | **64 MiB** |
| 1.28.5 | 363 MB | 270 MiB | 267 MiB |
| 1.29.1 | 339 MB | 273 MiB | 254 MiB |
| 1.29.2 | 14 MB | 8.8 MiB | 8.6 MiB |
| 1.30.1 (MPQ → CASC) | 1,632 MB | 1,628 MiB | **1,067 MiB** |
| 1.30.4 | 179 MB | 168 MiB | 157 MiB |
| 1.31.1 | 440 MB | 422 MiB | **323 MiB** |
| **all ÷ latest (2.06 GiB)** | **2.10×** | 1.98× | **1.54×** |

Today's `.vgdelta` writer itself left ~20% on the table against `zstd --patch-from` on the same input (333 vs 422 MiB
for WC3 1.31.1's stored pieces). That gap is independent of decoding.

Nested layers (128 KiB of real WC3 files, version 2 = 20 small rewrites and one insertion):

| Stack | Today (delta of the shipped files) | Decoded: content delta + per-layer data |
|-------|-------------------------------------|-----------------------------------------|
| 20 layers, every encoder treated as unknown (raw decisions) | 198 KiB | 110 KiB |
| 20 layers, neutral decisions | 198 KiB | 98 KiB |
| 20 layers, recipe first | 198 KiB | 5.2 KiB |
| 21 layers, unknown lazy LZ above LZW, decisions stored | 218 KiB | 51 KiB |
| 21 layers, the same, fitted (knobs + corrections) | 218 KiB | 6.6 KiB |
| 21 layers, the same, with the TOO_FAR quirk as a knob | 218 KiB | **3.0 KiB** |

Every stack rebuilt bit-identical from content + per-layer data. A 16-layer copy-class stack compiled into one read
program (22.12): every random read and a full sequential read were bit-exact, with no layer run at read time.

## 22.11 Layers: recipes, decisions, families

Real files nest encodings: a zip member holding a Unity bundle of LZ4 blocks, an MPQ sector zlib-compressed and then
encrypted. A piece is peeled top-down into its content plus, per layer, what it takes to re-encode that layer exactly.

### 22.11.1 What each layer stores

Each layer stores the cheapest of these that reproduces it, decided at mint by rebuilding and comparing:

| Form | Stored | When |
|------|--------|------|
| **recipe** | a codec and its parameters, read from the stream where possible (22.7.1) | the encoder is identified (stock zlib, liblzma, bzip2, …) |
| **family + knobs + corrections** | knob values and the differences (22.11.3) | an unidentified encoder of a known kind |
| **decisions** | the encoder's choices in neutral form (22.11.2) | nothing better fits |

- **Token replay:** with decisions (or knobs) in hand, re-encoding runs only the entropy coder: no search, no prices.
  It is bit-exact for any encoder: deflate 5/5 KCD2 members at 271–307 MB/s, LZMA 6/6 Unity blocks at 209–336 MB/s.
- Literals are never stored in decisions. They are the layer below's bytes; a decision stores only run lengths.

### 22.11.2 Neutral form (USER 2026-11-25: "all this should be encoded in the data")

- Decisions are stored as a neutral token list: literal-run lengths and (length, distance) matches, plus verbatim
  block headers where a format has them. Never in the encoder's own framing.
- The framing (LZSS flag bytes grouping eight items, LZ4's nibble token, a game's control bytes) is a deterministic
  function of the token list and is DESCRIBED, not stored.
- Why: framing aligns tokens in groups, so one insertion shifts every group after it. Measured on LZSS: v2 delta 14.4 KiB
  in the encoder's framing, 2.3 KiB in neutral form, 0 after fitting.

### 22.11.3 Families, fitting, corrections

- Nearly every LZ encoder ever shipped is a hash-chain parser that differs from the others only in a few knobs:
  - hash length, minimum and maximum match, window, chain depth;
  - lazy or greedy, insertion policy, end-of-buffer limits;
  - quirks such as zlib's TOO_FAR (a 3-byte match more than 4 KiB back is dropped).
- The registry holds the family as ONE reference parser.
- **Fitting (mint):** knobs the stream reveals are read from it: minimum and maximum length, a window that covers the
  largest distance. The rest are searched.
- **Corrections:** the reference parser runs in lockstep with the real decisions, follows them, and records only where
  its proposal differs. Correct by construction for ANY encoder; zero corrections means the encoder is now a recipe.
- **Measured:**
  - An independent lazy LZ with a planted TOO_FAR quirk: its knobs were found exactly (hash 3, min 3, max 130, window
    8191, chain 8, lazy, insert-all). Its corrections, 5.8 KB against 77.6 KB of neutral decisions, were exactly the
    quirk. With the quirk as a knob: 0.
  - LZSS and LZ4: 0 corrections.
- **Chaotic layers.** LZW, bzip2, adaptive Huffman and adaptive coders re-randomize everything above them, so a layer
  above one re-parses brand-new bytes every version. As decisions that costs its whole parse (51 KiB of the 21-layer
  stack). Fitted, the knobs are the same in both versions, so it costs only corrections (0 here).
- The same scheme extends to every other family: code-length tables for Huffman, model parameters for arithmetic
  coders, code width and reset policy for LZW.

## 22.12 Flattening: one read program per piece

USER 2026-11-25: "the sum of the format data of the layers produces the exact recipe to decode the data in one go".

- Every byte a layer emits is either **copied** from the layer below or **generated** by the layer itself.
  - Copied: container members, stored data, the literal runs of byte-aligned LZ formats (LZ4, LZSS, PackBits, most game
    LZs), bytes passed through a positional XOR or a per-byte table.
  - Generated: headers, match tokens, flag bytes.
- A layer's description plus its per-layer data therefore yields a program over its input: `COPY(range, transform)`
  and `ADD(bytes)`.
- Programs COMPOSE, exactly like `.vgdelta` segments flatten (chapter 04). An upper COPY of a range is replaced by the
  slice of the lower program under it; XOR keys and per-byte tables compose into one.
- All copy-class layers of a piece collapse into ONE program over the content. A read is a binary search plus memcpy,
  whatever the depth.
- **Adaptive coders are GENERATE steps:** LZMA's range coder, arithmetic coders, LZW, adaptive Huffman, chained
  ciphers. Their output bits depend on all earlier bits, so they must run, from the nearest derived checkpoint (22.7.2).
  - Everything below a GENERATE step (where its literals come from) and above it (who copies its bytes) still flattens.
  - **Read cost = the number of GENERATE steps on the path, not the number of layers.** In practice that is 0
    (LZ4/Unity, stored containers) or 1 (zip/deflate, MPQ/zlib, LZMA).
- **Static Huffman folds in (proven).** Within a deflate block the tables are fixed, so the output is a pure function of
  position. The program holds header bits (verbatim), literal runs (content bytes through the block's code table) and
  match / end-of-block codes (generated bits), each with its bit offset. A read starts at any entry: no checkpoints,
  no encoder state.
  - Measured on 22 KCD2 `.pak` members (339 KiB – 4.4 MiB), one i9-10900K core:
    - 44,000 random 4 KiB / 64 KiB reads and every full sequential read were bit-exact.
    - p50 0.11–0.17 ms, p99 ≤ 0.37 ms.
    - Sequential: 240–435 MB/s of shipped bytes, FASTER than zlib inflating the same streams (187–324 MB/s) and 10–20×
      a zlib re-encode (13–41 MB/s).
    - Compile: 1.4–16 ms per member.
  - Compiling a bit-level layer reads the content (a literal's code length depends on its byte); copy-class layers do
    not.
  - **No resident program (measured).** A materialized program of 24-byte entries is 1–3× the shipped size, and it is
    not needed. A read needs only:
    - the skeleton: per block, its bit offset, content offset, token-stream offset and verbatim header bits. That is
      0.30% of the shipped size, stored anyway.
    - the layer's neutral decisions as a token stream: (literal run, length, distance) per match, 32% of the shipped
      size raw and 18% zlib'd. They are stored data, read through the page cache, not owned RAM.
    - four decoded code tables (about 5 KB).

    The read jumps to its block, sums code lengths to its start, and emits. The 22 KCD2 members re-measured this way:
    every read bit-exact, sequential 215–367 MB/s, random p50 0.13–0.21 ms, so no slower than the resident program.
  - Where a piece keeps a recipe instead of stored decisions, a read re-encodes the block with the recipe (13–41 MB/s)
    and the RAM block cache holds the result. That trades disk (decisions) for CPU; the packer chooses per piece.
- **Compiled, never stored.** The program is derived from the per-layer data, never stored. For deflate it is not even
  kept: a read compiles just its own range (see above). Only LZMA-class checkpoints are kept in RAM (22.7.2).
  - Compiling touches no content bytes: it is arithmetic over the layers' tables. Composing 16 layers of a 128 KiB
    piece took 0.4 s in the Python prototype.
  - Storing it would be wrong: the program is large (300 KiB serialized there), and its version-2 delta (121 KiB) is
    larger than today's delta of the shipped files (70 KiB). The per-layer data behind it deltas to about 3 KiB.
