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

**The content is the lineage (USER 2026-11-25).** A piece's history is the history of its decoded content. Each
layer's data (recipe, knobs + corrections, or decisions, plus its skeleton) is per-version data stored beside it.
- Layer data deltas against the previous version's layer data only while the encoder stays the same.
- A recompression event (WC3 1.27a switched the zlib window to the sector size) changes only the layer data, and a
  recipe costs a few bytes.
- Measured on 1.27a: decoded 64 MiB, against 281 MiB in token form, where the parse IS the history and a new encoder
  breaks it. Tokens never carry the lineage; they are layer data.

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
- **Copy layers need no resident program either (measured).** Each layer keeps a sparse INDEX (its generator state
  every R output bytes) and reads its own stored decisions.
  - A read seeks the top layer's index and walks its decisions, generating framing bytes and asking the layer below
    for literal ranges.
  - Those requests arrive in increasing order, so each layer keeps a forward cursor and rarely seeks.
  - Measured: 8 MiB of WC3 content through the 16-layer stack (4.9 MiB shipped, 3.0 MiB of decisions), one core.
    Every sequential and random read at depth 1, 2, 4, 8 and 16 was bit-exact.

    | Depth | Sequential | Random 4K/64K p50 / p99 |
    |---|---|---|
    | 1–2 | 163–177 MB/s | 0.19–0.28 / 0.7–1.3 ms |
    | 4 | 121–135 MB/s | 0.22–0.30 / 1.3–1.6 ms |
    | 8 | 110–112 MB/s | 0.31–0.38 / 1.3–1.7 ms |
    | 16 | 52–53 MB/s | 0.98–1.02 / 2.1–2.4 ms |
  - RAM: the indexes only. With R = 64 KiB that is 1.0% of the shipped size for all 16 layers together; R = 4 KiB
    takes 14% and is barely faster.
  - **The trade-off.** Lazily, CPU grows with depth × piece granularity. LZSS literal runs of 1–8 bytes make this
    stack about the worst case: 2.2M pieces for 4.9 MB. A materialized flat program keeps depth free but costs RAM
    about 6× the shipped size at this granularity.
  - Real files are 1–3 copy layers deep, where lazy reads run at full speed, so lazy is the default. A per-region
    compiled program in the RAM LRU is the option for a deep stack read repeatedly.
- **Compiled, never stored.** The program is derived from the per-layer data, never stored. For deflate it is not even
  kept: a read compiles just its own range (see above). Only LZMA-class checkpoints are kept in RAM (22.7.2).
  - Compiling touches no content bytes: it is arithmetic over the layers' tables. Composing 16 layers of a 128 KiB
    piece took 0.4 s in the Python prototype.
  - Storing it would be wrong: the program is large (300 KiB serialized there), and its version-2 delta (121 KiB) is
    larger than today's delta of the shipped files (70 KiB). The per-layer data behind it deltas to about 3 KiB.

## 22.13 The algebra (USER 2026-11-25: "the cardinal concept")

The old deltas work because their one operation, COPY, is closed under composition: a chain of versions collapses
into one map, built once. Decoding deltas extend the operation set with encode/decode, and the set must stay CLOSED,
so that layers and versions both collapse and the reader applies "the sum of all mutations, once". The format is the
smart part (a compiled program); the reader is a dumb machine with a fixed instruction set.

### 22.13.1 Instructions (nodes)

| Node | What it is | Composes |
|------|------------|----------|
| SRC | raw bytes (content, literal pools) | — |
| SEQ | COPY(node, range) / ADD(pool) segments, optionally through a MAP (XOR keystream, byte table) | SEQ over SEQ flattens at build time; keys combine; an ADD under a MAP is precomputed |
| FRAME | an LZ framing (LZ4, LZSS, game LZs, PackBits) generated from stored neutral decisions | its literals read the input node; stays a node |
| PACK | a static-table coder (deflate blocks) from stored decisions + per-block skeleton | MAPs fold into its table; a COPY above it is a bit range |
| GENB | block-bounded state (MPQ cipher per sector, CBC, BWT/MTF blocks) | the block is the unit |
| GENS | stream state (LZMA's range coder) from derived checkpoints | its input flattens; a COPY above it is a range |

### 22.13.2 Behaviour classes: every format fits one

C (copy-like) · T (table-coded) · B (block-bounded state) · S (stream state). (O, opaque, is retired by 22.15.5: a
format no class fits gets new primitives.) Rules that put a format into a class:
- decode to the last lossless form only;
- everything content-independent is precomputed at mint and stored (offsets, tables, salts, IVs, timestamps);
- linear checksums (CRC) combine;
- model and coder are separate;
- solid archives split into per-file content under one GENS;
- unknown encoders are fitted;
- shared dictionaries are COPY sources.

### 22.13.3 Proven (v0 prototype, `~/Storage/tmp/vacuum/algebra`, one core, every read verified)

| Class | Real data | Result |
|---|---|---|
| C | 16-layer copy stack, 8 MiB WC3 content | 16 layers built and composed in 41 ms; containers and XORs vanish (execution depth 16 → 8, top = 3 segments); every layer's node exact; RAM 1% of shipped. Speed (50 MB/s) is set by the 8 stacked LZ framings, which must each emit their tokens; real files have one |
| T | 5 KCD2 `.pak` deflate members | recipe exact. Version 2 stored as content delta + decisions delta is 47–190× smaller: m00 4,333 → 22.7 KiB, m14 930 → 7.2 KiB. Reads 220–383 MB/s with no search |
| B | 12 encrypted, sector-compressed WC3 files (War3xLocal) | full stack exact: SEQ(table) > GENB(cipher) > SEQ(zlib header, adler) > PACK > SEQ(slice) > content. 177 MB/s. Version 2: 115.5 → 26.3 KiB (sectors already keep today's deltas local) |
| S | 2 Unity LZMA blocks, 16 MiB each | reads exact, p50 ~3 ms; mount 108–149 ms per block (checkpoints derived). **Version 2 barely wins: 1,026 → 915 KiB.** The optimal parser's decisions drift long after an edit (prices depend on history), so the tokens delta is ~60% of the tokens. The content delta is 2.4 KiB; realizing it needs the recipe encoder once per changed block: 2.3 s per 16 MiB (7.3 MB/s) |

Open: the policy for class S lineages (store tokens, derive once with the recipe, or keep stored); compacting
per-unit node overhead (B: 19% derived RAM in the prototype); a fused executor; the instruction set's binary form.

## 22.14 LZMA cracked: stored encoder states (2026-11-25)

**The problem, measured.** Disco Elysium's `dialoguebundle` is ONE LZMA block (136.5 MiB → 43.5 MiB). Every update
re-ships all 43.5 MiB today, even when 2 KiB of dialogue changed (9 real updates: 391.5 MiB). The content deltas are
2 KiB – 1.8 MiB; the encoder's tokens delta is 31.6 MiB.

**Why.** Carrying version 1's decisions over to version 2 through the content delta (TRANSPLANT): 90.5% are
identical, but the 9.5% that differ are spread everywhere, every ~43 bytes. That holds even across the final 76.5 MiB,
where the content and the match candidates are unchanged. The cause is the encoder's sequential STATE, not the content:
- live probabilities, quantized to 128 price levels;
- length price tables rebuilt every 63 lengths per position class;
- distance price tables every 128 matches, align prices every 16.

An edit shifts the refresh phases forever, so near-tie decisions flip for the rest of the stream. (Not position
contexts: Unity data is 4-aligned, so every shift is ≡ 0 mod 4.)

**The crack.** The chaos lives ONLY in that state; the match finder's results and the parse are pure functions of
content + state. So the packer, which runs the real encoder at mint anyway, stores the encoder's true STATE in the
delta every few MiB (54 KB raw, 10.4 KB zstd-19 each; no match finder). The stream splits into independent chunks:
a reader rebuilds the match finder from the 512 KiB window, loads the state, and re-encodes the chunk.
- **Measured:** 17 of 17 chunks re-encoded bit-exact (8 MiB each), 1.9 s per chunk → ~4 s for the whole block on 8
  cores. Only the chunks a read touches are encoded.
- **Per update stored:** content delta + 17 × 10.4 KB ≈ 0.2–2 MiB, against 43.5 MiB today. No decisions stored, no local
  store needed.

| Option | Stored per update (Disco) | Reader CPU |
|---|---|---|
| today | 43.5 MiB | none |
| tokens delta | 31.6 MiB | replay only |
| transplanted decisions + corrections | 7.6 MiB | replay only |
| **stored states + content delta** | **~0.35 MiB** (161 KiB + 177 KB) | ~32 s of encoding, parallel and lazy (~4 s wall on 8 cores) |
| content delta + one full re-encode | 161 KiB | 30 s sequential + a local store |

**The CPU floor, measured.** A predictor ran the SDK's own optimizer (`GetOptimum`) with the exact state and cheap
candidates: bt4's exact hash2/hash3 heads, version 1's transplanted match, and the K closest earlier 4-byte
occurrences. It took the true decision wherever its own differed and logged a correction. Its stream was bit-exact
every time.

| K | corrections | stored | reader CPU (136 MiB block) |
|---|---|---|---|
| 0 | 3.05% | 3.9 MiB | 24 s |
| 4 | 1.63% | 2.2 MiB | 30 s |
| 16 | 1.06% | 1.4 MiB | 39 s |

Even with K = 0 the optimal parser alone costs ~15–20 s: it is the irreducible floor for rebuilding LZMA decisions
without storing them. The predictor is dominated by the skeleton of states (fewer bytes at similar CPU).

**Design: two blobs per changed LZMA block, both in the delta, content-addressed, fetched per device.**
- The SKELETON OF STATES (~10 KB per 8 MiB) is always fetched: exact chunked re-encode, parallel and lazy.
- The TRANSPLANT CORRECTIONS (7.6 MiB for Disco's update) are an optional accelerator: replay only, no search, for
  slow devices.

Alternatively a device can fetch the rebuilt block from another of the user's devices over the LAN.

Open: chunk size versus cold-read latency (8 MiB → 1.9 s); the binary layout of both blobs.

### 22.14.1 Everything in the delta: replay-only LZMA (USER 2026-11-25: "no selective stuff, all built into the delta")

The skeleton of states alone needs the full encoder at read time (~5 MB/s per core). For tens of GB of LZMA that is
minutes of CPU per launch, so it cannot be the read path. The default for every device is therefore replay-only.

The delta of a changed LZMA block holds:
1. the content delta;
2. **corrections** to version 1's decisions, carried across the content delta (where none starts at a place, the rest
   of version 1's covering match is the prediction);
3. **state checkpoints** every 512 KiB shipped: probabilities, reps, state, coder registers, decision-stream cursors.

A read jumps to its checkpoint, rebuilds the decisions (a correction, else the transplanted one) and replays them
through the range coder. No search, no encoder, no local store.

**Measured, Disco 11-07 → 11-11** (`~/Storage/tmp/vacuum/disco/dual.c`):

| | |
|---|---|
| corrections | 10.2% of 40.6M decisions, 7.6 MiB zstd-19 |
| checkpoints | 87, 0.8 MiB |
| delta for the block | 8.4 MiB, against 43.5 MiB today (5.2×) |
| reads | full block bit-exact in 2.6 s on one core (55 MB/s decoded; parallel across checkpoint ranges); 200–400 random 4K/64K reads all exact, p50 15 ms, p99 ~40 ms |

**Where the bytes are.** Coding corrections as a rank among cheap alternatives ordered by the encoder's own bit cost
saves only 4%. Of the corrections, 60% are ranks 0–2; 40% are other lengths or distances. The cost is WHERE flips
happen: ~10% of decisions, scattered, ≈0.47 bits per decision just to locate them.

Next lever: predict where flips happen. A flip needs a near-tie, so the price margin between version 1's choice and its
cheapest alternative, under version 2's exact prices (which the reader tracks), becomes the context for adaptively
coding the flip flags.

### 22.14.2 Margin-coded corrections (measured, `~/Storage/tmp/vacuum/disco/margin2.c`)

Same delta structure, with the corrections coded as a model the reader replays exactly:
- **Flip flags,** one per predicted decision, arithmetic-coded. The context is the price margin between the prediction
  and its cheapest alternative under the EXACT replayed probabilities (40 buckets), × prediction kind × source (a v1
  decision start, or the remainder of a covering v1 match).
  - Flip rate by margin: 91% when the prediction looks 4 bits/byte worse, 10% when it looks better.
- **On a flip:** the rank among cheap alternatives ordered by bit cost (57% rank 0).
- **Escapes, structured:** kind, then length, then index. Kinds: a rep slot; the predicted distance; a distance a
  nearby v1 decision used; or the RANK of the distance among the closest earlier places holding the same 3/4 bytes
  (52% rank 0 for lengths ≥ 4; the reader builds these chains from the content, 1.5 s per 136 MiB).
  - Of 1.67M escapes, only 120K remain raw.
- Coders and models restart at every checkpoint, so random access holds.

| Disco 11-07 → 11-11 | Delta | Reader |
|---|---|---|
| today | 43.5 MiB | memcpy |
| plain corrections (dual.c) | 8.4 MiB | 55 MB/s per core |
| margin flags | 6.3 MiB | 32 MB/s per core |
| **margin flags + structured escapes** | **3.9 MiB: flags/ranks/escapes 2.9 + raw 0.2 + checkpoints 0.8 (11.2× smaller)** | 28 MB/s decoded per core, parallel across checkpoints; random 4K/64K p50 29 ms; full block and all random reads BIT-EXACT; no search, no store |

Next: checkpoints coded as deltas of the previous one (≈ halves 0.8 MiB); richer flag contexts; a cheaper margin for
reader speed.

### 22.14.3 At scale: Subnautica's 552 MiB LZMA bundle (measured)

Disco's dialogue block was its largest LZMA file. All of Disco's LZMA bundles total 0.49 GiB; its 6.6 GiB of LZMA
history churn is that 43.5 MiB bundle re-shipped ~150 times. Big single streams exist elsewhere: Subnautica's
`AssetBundles/basegeneratorpieces` (Unity 5, 2018) is ONE LZMA stream, 552 MiB decoded, 174 MiB shipped.

The pipeline was made memory-safe: decisions streamed to disk, inputs memory-mapped, the same-bytes chains in a sliding
window rebuilt per checkpoint. It gives identical results on the Disco pair.

| Builds 60 → 61 | |
|---|---|
| content change | 23 KiB |
| today | 173.95 MiB |
| **margin-coded delta** | **14.0–14.6 MiB (12×)**: flags+ranks 9.5, escapes 1.7, checkpoints 2.7–3.3 |
| flips | 9.5% of 157.8M decisions |
| reads | full bundle BIT-EXACT in 26 s per core (22 MB/s decoded; ~3 s on 8 cores across 348 checkpoints); random p50 48 ms |

**Finding:** with corrections, the cost scales with the STREAM, not the change. A 23 KiB edit costs ~8% of the shipped
bundle (Disco: 8% too), because the encoder's price-state chaos spans the whole stream. Corrections cap every LZMA
update at ~8% of today's. Going further, toward cost proportional to the change, needs window-level prediction (the
near-tie lattice) or the state skeleton (which costs CPU at read time).

### 22.14.4 Reader speed (measured, `margin6.c`/`margin7.c`)

- **Where the time went:** the same-bytes chain index (16 MB hash heads → a cache miss per position), escape
  decoding, and neighbour lookups. With cache-resident heads (64K buckets) and a narrower neighbour search: 29 MB/s
  decoded per core (from 20), random p50 30–34 ms, delta unchanged (3.6 MiB).
- **Parallel:** checkpoints are independent entry points, so a bundle read splits over cores. Disco's 136 MiB block:
  | threads | wall | decoded | shipped |
  |---|---|---|---|
  | 2 | 2.53 s | 57 MB/s | |
  | 4 | 1.26 s | 113 MB/s | 36 MB/s |
  | 8 | 0.75 s | 191 MB/s | 61 MB/s |

  All bit-exact.
- **The fair comparison** is the game's own LZMA decompression of what we serve (stock liblzma, one thread): 94–98 MB/s
  decoded = ~30 MB/s shipped. On the PC we produce the shipped bytes ~2× faster than the game consumes them. On the
  Deck (4 cores, ~0.5×) roughly even. The RAM block cache makes repeat reads free.

### 22.14.5 Depth-free: predict from the anchor (measured)

Predicting version N's decisions from version N−1's would chain: N−1's decisions must be rebuilt first, then N−2's,
down to the anchor, so depth would cost. Instead every version predicts from the ANCHOR's decisions:
- they are free, by parsing its shipped stream;
- positions map through the composed content map, which flattens like any delta chain.

Measured, Disco version 7 predicted directly from the first build (7 updates apart; content delta 245 KiB):
**3,634 KiB**, against 3,621 KiB from version 6. Distance costs nothing, because the encoder's chaos already flips ~10%
of decisions for any edit.

So a version's LZMA layer data stands alone: corrections vs the anchor + checkpoints. Reading any version is one step
from the anchor, whatever its depth. The decision stream is parsed from shipped bytes and the range coder is the same
for every LZMA encoder, so this needs NO recipe: unknown LZMA encoders and SDK versions work as they are.

Status of LZMA: bit-exact, replay-only, no local store, depth-free, encoder-agnostic; ~12–13× smaller than today on
Disco and Subnautica; the reader serves shipped bytes ~2× faster than the game decompresses them (PC, 8 threads).
Not optimal: an update costs ~8% of the shipped stream however small the change. Window-level prediction (the
near-tie lattice) is the lead for going below that.

### 22.14.6 Remembered near-ties: measured, not adopted (2026-11-26)

A side study (Fable) proposed a memory lattice: per content place, every decision any earlier version shipped there,
re-decided at read time by a window-level shortest path that replays the 7-Zip SDK's price model. Facts it established,
which stand:
- the SDK's match finder is a pure function of the content (74.8M positions compared, 0 differing candidate lists);
  only the price state drifts;
- the parse is made of independent windows whose prices are frozen at the window start;
- the lineage saturates: decisions never seen before fall from 9.5% to 1.6% with seven remembered versions.

Built as one reader and measured over a real lineage (Disco v1..v9, the anchor as base of every delta, memory = the
hunks earlier deltas shipped, 22 checkpoints, every read bit-exact): 4,659 / 3,968 / 3,697 / 3,657 / 3,390 / 3,380 /
3,277 / 4,873 / 4,737 KiB. Margin coding (22.14.2) at the same checkpoint spacing is ~3,050 KiB for v7. With the
previous version as base it reached 1,909 KiB, but that base is not depth-free. Not adopted: it needs a reader that
carries one encoder's brain, and under depth-free it does not win.

Kept from the build: checkpoints only need to cover parallelism for streams games read whole (Unity LZMA bundles are
decompressed start to end), so one per 2 MiB shipped; an index over content needs only the format's window (22.15).

## 22.15 The coding algebra (USER 2026-11-26: "we just discovered new algebra that we didn't axiomize yet")

22.13's nodes treat each coder as a black box: PACK for deflate, GENS for LZMA, FRAME for each LZ framing. Under the
hood every one of them is the same few primitives wired differently, and so is the predictor that makes their deltas
small. The reader implements the PRIMITIVES, once. A format, and an encoder family, is a description that composes them,
shipped as data. Nothing in the reader knows a format or an encoder.

### 22.15.1 Sorts

| Sort | What it is |
|---|---|
| Content `X` | the decoded bytes; the lineage (22.11) |
| Token `t` | one decision in content coordinates: `Lit` at place `p`; `Copy(p, len, dist)`; plus any choice the format leaves the encoder, e.g. `asRep` (code a copy through the history even though a fresh distance exists) |
| Decisions `T` | the token sequence covering `X` |
| History `H` | a state machine stepped by tokens: rep slots (k distances, an update rule), a format state (LZMA's 12 states) |
| Model `M` | per token, a tree of binary or n-ary SYMBOLS, each coded under a SLOT chosen by a context expression over `H`, the place and the content (previous byte, match byte, position bits) |
| Slot | a probability with an update rule (adaptive: shift `s`, bit width `b`) or a static code (lengths from a header) |
| Coder `C` | turns (symbol, slot) into output bits: range coder, arithmetic coder, Huffman emitter, FSE/tANS, raw bits or bytes; bit order and carry rules are knobs |
| Framing `F` | blocks, headers, end markers, checksums: functions of `T`, `X` and the coder's output |

### 22.15.2 Axioms

1. **Replay, three ways.** Shipped bytes are a pure function `S = F(C(M(H(T), X)))`. ONE description runs in three
   modes: DECODE (`S` → `X`, `T`: the mint's parse), CODE (`X`, `T` → `S`: the read), PRICE (the cost of a candidate
   decision: the predictor). The reader never searches. Every primitive is lossless both ways.
2. **Neutrality.** `T` is stored in content coordinates. Whatever the format derives from it (symbol codes, rep
   indices, flag bytes, nibble packing, block headers that are functions of the tokens) is never stored. Only true
   choices are decisions: `asRep`, a rep slot when two slots hold the same distance, a block split, a header the
   encoder picked.
3. **Transport.** A content map (a SEQ, 22.13) carries tokens: `Copy(p, len, dist)` maps to the new version when its
   bytes and its source lie in copied segments; it lands at `(p', len, p' − q')`. Decisions of any version are
   therefore available in any other version's coordinates, through the flattened map, at no depth cost.
4. **Window.** Every format has a window `W` (deflate 32 KiB, LZMA the dictionary size, LZ4 64 KiB). No distance
   exceeds it. Any index over content (same-bytes chains, hashes) needs only the last `W` bytes, and a candidate
   beyond `W` does not exist. (Measured: 1.6 GiB of whole-file chains → 15 MiB per reader, same results.)
5. **Cost.** A model prices itself: `cost(t | state) = −log₂ P_M(t | state)`, read from the same slots the coder uses
   (adaptive: current probabilities; static: code lengths; raw: bit counts). There is no separate price description.
   An encoder's own price tables (refresh counters, snapshots) are NOT part of the format: if a family needs them,
   they are family knobs (22.15.4).
6. **Prediction.** A predicted token is a deterministic function of: the transported anchor token at this place (the
   default), the candidates (literal, every history slot, the same-bytes chain within `W` ranked closest first) and
   their costs. Prediction is the same for every format; only the model behind the costs differs.
7. **Correction.** `T = predict ⊕ corrections`: a flip flag per token in contexts of the cost margin, the token kind
   and the previous flip, a rank among the candidates, and escapes (history slot, transported distance, neighbour
   distance, chain rank, then raw). The correction coder is itself an instance of the algebra (adaptive binary slots +
   range coder).
8. **Checkpoint.** The state of `H`, every slot and the coder at a token boundary is a checkpoint, stored as data.
   Any read resumes from any checkpoint. Placement is a knob of the piece: one per unit of parallelism for streams
   read whole, denser for random access.
9. **Closure.** A coding node `CODE(description, X, T)` composes with SEQ above and below exactly as 22.12 states:
   what is copied flattens, what is generated runs. 22.13's PACK, GENS and FRAME are CODE nodes whose descriptions
   differ: PACK has static slots (flattens per block), GENS has adaptive slots (runs from checkpoints), FRAME has raw
   bytes (copies its literals).
10. **Demand** (2026-11-26, USER: "the fact that it doesn't flatten yet just means that our algebra isn't good enough
    yet"). A CODE node answers a RANGE of its output as a run list: GEN runs (bytes it generates, from its own pool) and
    COPY runs (a range of its content, passed through as it is). A COPY run is handed to the node below as a range and
    never materialised; `COPY ∘ COPY = COPY`, so a byte copied through any number of nodes is written once, by the node
    that holds it, straight into the reader's buffer. A node reads of its content only what its code reads: in growing
    windows while it reads contiguously, nothing behind its segment unless its format looks back (an LZMA literal's
    match byte; deflate and LZ4 never do), and nothing it jumps over. Corollaries:
    - **Read cost = the GEN work on the path**, whatever the depth. COPY-class nodes (stored, LZ4 literals, container
      members, framing) cost a run-list walk each; static coders cost their own block; adaptive coders run.
    - **The floor is real.** An adaptive coder's output bits depend on every earlier bit, so an adaptive coder over an
      adaptive coder runs both; no algebra removes that. What the algebra gives instead is PARALLELISM: segments form a
      dependency graph across nodes (a segment of node k needs only the segments of node k−1 under its content range),
      so decoding runs lower nodes first and every segment of every node is a job of its own.
    - **Checkpoints are cheap, so dense.** A snapshot costs about 5 KB for LZMA; placing them for parallelism and
      latency (axiom 8's knob) is the default, not the exception.

### 22.15.3 Formats as descriptions (on paper)

| Format | Tokens | History | Model | Coder | Framing |
|---|---|---|---|---|---|
| deflate | Lit, Copy(3..258, 1..32768) | none | per block: lit/len symbol (286, base + extra bits tables), dist symbol (30, base + extra), EOB; static slots from the block header | Huffman, LSB-first, codes bit-reversed | block type, final bit, stored blocks, dynamic header = decision |
| LZ4 | Lit runs, Copy(4.., 1..65535) | none | raw: token byte (lit-run nibble, len−4 nibble), 255-run extensions, offset u16 LE | bytes | frame header, block sizes, xxHash32 (derived) |
| LZMA | Lit, Copy, asRep (slot 0..3), ShortRep | 12-state table, 4 rep slots move-to-front | `isMatch[state][pos & pb]` → `isRep[state]` → `isRepG0..G2` / `isRep0Long`; length: choice + low/mid/high trees per pos state; distance: slot tree per length state, reverse trees, direct bits, align tree; literal: 0x300 slots × (lc, lp) context, matched mode keyed by the match byte; adaptive 11-bit slots, shift 5 | range coder (low, range, cache + carry) | props header, optional end marker |
| zstd | sequences (lit run, Copy) | 3 rep slots, repcode meaning shifts when lit run = 0 | literals: Huffman (1 or 4 streams); sequences: FSE tables per block (predefined / RLE / compressed / repeat = decisions) | backward bitstream, FSE | frame + block headers, xxHash64 (derived) |

Each row is a description, not code. A new format is a new row.

### 22.15.4 Families are descriptions too

An encoder family (22.11.3) is a description of how it CHOOSES tokens: candidate generator knobs (hash length,
chain depth, window), a strategy (greedy, lazy with its thresholds, optimal parse with its window length), and quirks
(zlib's TOO_FAR). With a family identified, predictions improve and corrections shrink; without one, axiom 6's
generic prediction still works. A family is never needed for correctness.

**Built (ISA 5, 2026-11-27): the PREDICT op and the first family.** A family is a strategy the READER implements once
(code, versioned with the instruction set), named and tuned by a description (data): `PREDICT` writes the family's token
anchored at a content place into three registers (literals, copy length, distance; copy length `kNoPrediction` where it
predicts none), from the family's parse of a block of the content the description bounds. A description that uses it
spends ONE flag decision per token: 0, the family's token; 1, the token follows as it is. Its record is PREDICTED
(`VGCQ`): a mint decodes twice (plainly, to have the whole content; then with the family, every token checked), and the
same description minted `--plain` gives the PLAIN record (`VGCR`: every flag 1), which is what a machine reads (chapter
04 §CODED: the plain twin, `coding expand`). Parses are cached per block (64 blocks or 64 MiB of tokens at most) and
shared by every segment decoding in parallel. A parse is budgeted by its WORK: chain steps since the run started or was
restored from a checkpoint, 128 a byte of that run's progress plus one largest block's. Real archives take about 4 a byte
(73 in Skyrim SE's worst block); content built to make the parse crawl takes hundreds, and is refused within seconds a
block. A predicted record checkpoints every 8 MiB shipped (a few family blocks a segment, so segments parse in
parallel), and carries the hash of every segment's shipped bytes: a reader whose family predicts otherwise (another
reader's, another knob, a hostile record) refuses the segment instead of serving other bytes. Its plain twin keeps the
dense checkpoints random reads need.

- **Family 1, LZ4 high compression** (`lz4family.h`): lz4 1.7's hash-chain parse (4-byte hashes, 2^15 heads, a 64 KiB
  chain, the three-match lookahead with its corrections), written from the algorithm; knob: attempts. **With 32,768
  attempts (lz4 1.7's level 16, its maximum) it is Bethesda's packer**: it reproduces Skyrim SE's archives block for
  block (Meshes1, 2016: 3,000 / 3,000 blocks identical; Textures1: 696 / 715, the rest corrections). Today's liblz4
  reproduces 226 of 300 at level 9 and none above (its levels 10–12 are another parser).
- **Family 2, zlib's deflate parse** (`zfamily.h`: deflate_slow — hash chains, lazy matching, TOO_FAR — levels 4–9;
  knob: level | memLevel << 8 | windowBits << 16; token for token zlib's, GOG's Inno installers included: level 6). It
  is INCREMENTAL (a deflate stream may be gigabytes): it parses only as far as asked and keeps a window of tokens behind
  it. Each reading machine keeps its own (a reader's segments decode in parallel; one shared incremental parse was dragged
  back and forth between them and refused), so a segment restored mid-stream re-parses its stream from the start once:
  the one catch-up its budget grants it. A mint simulates exactly that at every checkpoint. Records are the deflate block
  headers and Huffman trees (decisions: 0.3–0.7% of the compressed bytes) plus whatever zlib did otherwise.
- **BSA 105 with the family** (its description's `predict lz4hc 32768`; other attempts, or none: byte for byte the
  description before families): records drop from hundreds of MB to hundreds of KB — Meshes1 (373 MB shipped) 67 MB →
  0.9 MB; Textures1 (1.51 GB) 369 MB → 0.66 MB. Minting runs the family (Textures1: about 6 minutes); reading through the
  predicted record runs it too (Textures1: 36 MB/s front to back, but a random read decodes up to an 8 MiB segment), so a
  machine expands it once (Textures1: 95 s) and reads the plain twin (dense checkpoints) at plain speed. The content is kept at rest as a delta over nothing (chapter 04: zstd blocks), about the shipped size or less
  (zstd-9: Meshes0 2.10 GB content → 0.59 GB against 1.15 GB shipped; Textures1 2.10 GB → 1.40 GB against 1.51 GB).

### 22.15.5 Tiers, restated (USER 2026-11-26: "no opaque blocks, if there's no ops, invent them")

1. Declarative: a format row of 22.15.3, a family of 22.15.4.
2. Knobs: values for a row's parameters (lc/lp/pb, window, block size).
**Compiled code never ships (USER 2026-11-26: "never webassembly, we shall never ship compiled code").** No
package, delta or description carries executable code of any kind: no WebAssembly, no native modules, no bytecode for
a general-purpose VM. A description is data over the algebra's primitives; only the reader is code.

3. **A new primitive.** What no description can say yet is not shipped as code and not kept as opaque bytes: the
   algebra grows the op that says it (a cipher's round, a checksum, a transform). The instruction set is therefore
   VERSIONED: a description names the instruction-set version it needs, and a reader too old for it says so instead of
   guessing. There is no carried-code tier and no opaque class (22.13.2's O is retired).
4. Stored: corrections only, never whole blocks.

### 22.15.6 Primitives invented for decode mode (2026-11-26)

Deflate's block header is itself Huffman-coded, so describing deflate needs the description to READ as well as write:
- a mode mask on every op (an op may act in DECODE only, CODE/PRICE only, or all);
- decisions: a FREE flag makes a coding op's value a decision (CODE reads it from the decisions, DECODE records it); a
  DEC op moves a register to or from the decisions;
- content: OUTLIT and OUTCOPY (DECODE writes the content, CODE and PRICE only advance the place);
- arrays (description-owned, constant-initialised or written by STA) and loops (JZ/JMP over a counter);
- BUILD: a canonical Huffman code from a lengths array (encode table and decode table, over-subscription refused);
- the LSB-first bit coder: RAW (n bits) and HUFF (a symbol through a built code), and BITPOS (for alignment pads,
  which are decisions);
- CALL/RET, so a stream program calls a token program (PRICE runs the token program alone);
- ASSERT: DECODE refuses what the format forbids (a distance before the start, a length symbol the format leaves
  undefined).

### 22.15.7 Status (prototype `~/Storage/tmp/vacuum/cod/`)

Proven before the axioms existed, as hand-wired instances: deflate replay (PACK, 22.12, VidyaGodFS `deflatelayer`),
LZMA replay + prediction + correction (margin coding, 22.14.2–22.14.5), LZ4/LZSS framings (FRAME).

**Generic engine, measured 2026-11-26 (`cod.h`, `lzma_desc.h`, `margin_gen.c`).** The engine knows primitives only:
ops (BIT, TREE, RTREE, DIRECT, MLIT, JZ/JMP, SET, history MTF/PUSH), expressions over registers, content bytes and
tables, adaptive 11-bit slots, a carry range coder. LZMA is a description: 55 ops, 153 expressions, 7,990 slots, 4
state tables, knobs (lc/lp/pb, window, longest copy, the length-1 history slot, an end token).
- **Replay:** Disco v7's 40.6M decisions re-encoded by the engine from the description: all 45,609,513 shipped bytes
  BIT-EXACT. Expressions are specialised once (flattened to three-address micro-ops, constants folded): 30 MB/s
  decoded per core, against 67 MB/s for the SDK's own replay on the same tokens.
- **Margin coding on the engine:** costs come from the same description in price mode (axiom 5); transport,
  candidates, flags, ranks and escapes are generic; the one LZMA-only context of the hand-wired coder (state ≥ 7)
  became "kind of the previous token". Disco v6→v7: **3,593 KiB** (hand-wired: 3,621), every full and random read
  bit-exact. Reads 12 MB/s per core, 85 MB/s on 8 threads (hand-wired: 191); random p50 78 ms.
- **Open:** speed. About four prices per token go through the interpreter. Candidates: compile the description to
  native code at mount (a JIT over the micro-ops), or a per-op fused form for the affine slot expressions that make up
  most of a description.

**Engine v2: three modes, measured 2026-11-26 (`cod2.h`, `asm2.h`, `lzma2_desc.h`, `deflate_desc.h`, `test2.c`).** With
the primitives of 22.15.6 one description decodes, codes and prices:
- **LZMA** (119 ops, 217 expressions): DECODE of Disco v7's 45.6 MB block gives the 136 MB content exactly, and its
  40,639,329 decoded tokens equal the SDK's own token file one for one; CODE of content + decisions gives the shipped
  bytes BIT-EXACT.
- **deflate**, dynamic header included (118 ops, 163 expressions, 4 arrays, 3 codes): 22/22 KCD2 `.pak` members decode
  to exactly zlib's content and code back BIT-EXACT. Broken streams (cut short, trailing bytes, 298 bit flips) are
  judged exactly as zlib judges them: 300/300, nothing accepted that zlib refuses.
- Speed, unspecialised (recursive expression trees): LZMA 14 MB/s decode, 15 MB/s code; deflate 18 / 23 MB/s.
  Specialisation (v1 already measured 2.1–2.3× from it) and native compilation at mount are the next step.

- **margin coding on v2** (`margin_gen2.c`): costs from PRICE mode of the same token program that decodes the stream.
  Disco v6→v7: 3,594 KiB, the same 4,095,006 flips as v1, every full and random read bit-exact (unspecialised: 53 MB/s
  on 8 threads).
- **LZ4, from its description alone** (`lz4_desc.h`, 46 ops, 78 expressions, written from the block format; no LZ4
  code anywhere): liblz4's fast encoder and HC levels 3, 7, 11 over three kinds of content at seven sizes (0 B to
  4 MiB): **84/84 blocks decode to the exact content and code back bit-exact**.

Three formats, three coders (range coder, Huffman bit coder, raw bytes), no format code in the engine.

**In VidyaGodFS (2026-11-26 night, `src/coding.{h,cpp}`, `src/codingdescs.cpp`).** The engine is production C++:
- a binary description format (`VGCD`, instruction set version 2) with a validator: expressions refer only to earlier
  expressions; every op, register, table, array and code index is checked at parse; table and array totals are
  capped; a description whose compiled form would be too large is refused;
- the machine: expressions compiled once into micro-ops (constants folded, register and constant operands in place),
  a per-mode skip table, slots / tables / arrays bounds-checked at run, runs metered (fuel per byte produced);
- `Mint` (DECODE, then CODE back, refusing unless every byte matches; checkpoints at the description's MARK points) and
  a record (`VGCR`: index + one zstd payload per checkpoint: machine snapshot + that segment's decisions);
- `CodedByteSource`: random access from checkpoints; a sequential reader continues the paused run (RAM only);
  hostile records are refused (bounded carry counts, probabilities, lengths, payload sizes).
- Measured: Disco's LZMA block and 22 KCD2 deflate members mint and read back BIT-EXACT; reads 15 MB/s (LZMA) and
  25 MB/s (deflate) of content per core. The record still holds ALL decisions (42.8 MB for Disco): the predictor
  (margin coding against the anchor) is the next layer, and the engine's speed (about 4.5× behind native LZMA replay)
  the next optimisation.
- Found by the mint's code-back check: deflate's one non-canonical spelling (length 258 as code 284 + 31), now a
  decision (token kind 3).
- **Flattened and fast (2026-11-27, USER: "make all the layer types flatten... over 100MB/s for all, go crazy").**
  - *Copy runs pass through:* a run of content bytes in the output (LZ4 literals, stored blocks: ISA 3's BYTERUN) is one
    bulk read of the layer below, which composes through deltas; static-code literal runs (deflate: ISA 3's HUFFRUN)
    are one native loop.
  - *Static coders checkpoint per block:* a description without adaptive slots checkpoints every ~64 KiB shipped
    (`SuggestedSpacing`); adaptive ones (LZMA) every 2 MiB. Every checkpoint segment decodes on its own, so a reader
    going front to back is served by a shared pool decoding the next segments ahead (RAM ring, a few per stream).
  - *The machine compiles each mode once:* only that mode's ops, targets remapped and threaded through JMPs, operands
    resolved in place (constant, register, compare-with-constant, table[register], AFFINE forms with up to four terms
    of register / content byte / table lookup, else micro-ops), common patterns fused (compute-bit + code-bit +
    branch, code-bit + branch, decision runs), direct-threaded dispatch; checkpoints keep original op numbers.
  - *Descriptions do less:* an LZMA literal carries one decision, a copy four (record 42.8 → 38.0 MB).
  - Measured (i9-10900K, default pool of 8; all bit-exact on real data):

    | Format | Sequential | Random 4K/64K p50 | One core | Four workers |
    |---|---|---|---|---|
    | LZMA (Disco block) | **184 MB/s** | 133 ms (2 MiB spacing) | 27 MB/s | 95 MB/s |
    | deflate (22 KCD2 members) | **475 MB/s** | 0.38 ms | ~105 MB/s | 334 MB/s |
    | LZ4 (Disco content, 4 MiB blocks) | **856 MB/s** | 0.70 ms | 173 MB/s | 512 MB/s |

    The hand-written deflate layer read 241 MB/s. Open: LZMA per core (the range coder's serial chain plus interpreter
    state in memory; native replay is 67 MB/s) — the next step would be compiling a description to machine code at
    mount (the reader's own, never shipped); LZMA random reads (closer checkpoints trade record size for latency).
- **Mounted.** A layer of type `coded` in the mount spec: `source` = the record, `description` = the description file,
  `bases` = ONE CID, the content (any layer carrying bytes: a bytes-only file, a delta's rebuild, another coded stream),
  `name` = the file it appears as in its `target` folder. Planned like a delta (its content resolved by CID; a content no
  layer carries plans it broken and nothing is served) but it hides nothing; served as a one-entry archive, so open,
  read, stat and copy-up are the zip path's. The record carries the shipped size.
- **Demand: copy-class layers are depth-free (2026-11-26, axiom 10).** A decoded segment is its generated bytes plus
  its copy runs (byte runs of 256 B and more, kept as content ranges); a read hands the copy runs down to the layer
  below outside every lock, so copied bytes are written once, by the layer that holds them. The content window reads
  only what the code reads (16 KiB fresh, doubling while reads are contiguous, small again after a jump, nothing behind
  a segment unless the format looks back), the decoding pool runs lower layers' segments first, and adaptive coders
  checkpoint every 64–512 KiB shipped (about 32 segments; an LZMA snapshot is about 5 KB). Found on the way: the mint
  aimed its k-th checkpoint at `k × spacing`, so a long literal run left a burst of tiny segments behind it; checkpoints
  are now spaced from the last one.
  - Measured (i9-10900K, pool of 8, `~/Storage/tmp/vacuum/stack/stackbench`, Disco content, all bit-exact):

    | Stack (bottom first) | Before | Now (MB/s of content) |
    |---|---|---|
    | LZ4 ×1 / ×2 / ×4 / ×8 | 2431 / – / 562 / 362 | 2355 / 2382 / 1644 / 1992 |
    | deflate stored ×1 / ×8 | 6328 / – | 5815 / 2943 |
    | deflate ×1, LZ4 over deflate, deflate over LZ4 | 1078 / 906 / 561 | 1286 / 1114 / 1841 |
    | LZMA ×1 (8 MiB / 128 MiB content) | 289 / – | 527 / 210–226 |
    | LZMA over LZMA; LZ4·deflate·LZMA ×1 / ×2 / ×7 (21 layers) | 170; 297 / 117 / 19 | 238; 484 / 269 / 41–53 |

    LZMA random 4K/64K p50 on 128 MiB: 48 ms (was 133 ms at 2 MiB spacing); deflate 2.4 ms, LZ4 1.0 ms. What remains is
    the adaptive floor (axiom 10): the 21-layer stack holds seven LZMA layers over high-entropy data, each running its
    coder over every byte. The engine runs LZMA about 3.6× slower per core than native liblzma; compiling a description
    to machine code at mount (the reader's own code, never shipped) is the lever left there.
  - Tests with teeth (mutation batches 4–6: every mutant that changes behaviour caught; two equivalent): copied bytes
    are not generated again, the content is read about once in few reads, a refused read below is refused above,
    record lengths that wrap are refused, a refusal pending at RET refuses, and the compiler's folds (affine mask,
    table bound, fusion with a jump landing inside its pattern) mean what they fold. Fusion no longer avoids landing
    points: it rewrites only a pattern's first instruction, so a jump into the pattern runs the originals.
- **ISA 4 and the first archive (2026-11-27).** `EX_STREAMPOS` (the shipped stream's byte position: read so far in
  DECODE, written so far in CODE, from the stream's start) lets a description check what an archive's index says
  against where its members really are. With it, **Bethesda's BSA v105 (Skyrim SE) is a description** (120 ops, 769
  expressions): its content is the header region (header, folder records, folder names and file records, file names) as
  it is, then every member's bytes in data order; members lie back to back in any order: the next member is the record
  whose offset is where the stream stands, tried at the record after the last one and otherwise found by a scan of the
  records (real archives store thousands out of record order: `Skyrim - Misc.bsa`, 21 474 members, 5 582 breaks, mints in
  10 s), and the count of members coded must be the archive's file count; bit 30 of a record's
  size flips the archive's compression for that member; embedded names stay with their member; every LZ4 frame option
  (content size, dictionary id, block and content checksums, stored blocks) is read and coded back. A description using
  `EX_STREAMPOS` is refused by an ISA-3 reader. Real archives mint and verify bit-exact (`Skyrim - Interface.bsa`,
  65.7 MB: 0.5 s to mint, 4.5 GB/s to verify). `MintInto` decodes into a caller's buffer (a tool reserves a lazy 4 GiB
  map). The algebra is on the command line twice: VidyaGodFS's `vgcode`, and VidyaGod's `coding check | info | mint
  | verify | cat | deflate-split | deflate-verify`.

## 22.16 Descriptions are node text (USER 2026-11-27: "they should be data"; "the descriptions should be in the node")

A description is a field of a node: `DESCRIPTION`, readable JSON. A `CODED` layer names the node that holds it, by CID
(chapter 04), and the launch must contain that node (a `NODE` layer). There is no binary description anywhere: the
engine reads the text, validates it (descriptions come from peers: untrusted), and runs it. The engine carries no
format; every format a package decodes is described by the package's own data. Writing descriptions is packaging work;
the engine owns the instruction set and the encoder families (code, versioned with it, named and tuned by
descriptions).

```jsonc
{ "LABEL": "Inno Setup v1",
  "DESCRIPTION": {
    "NAME": "Inno Setup, zlib level 6", "ISA": 6, "CODER": "BITS", "WINDOW": 32768, "MAXLEN": 258,
    "REGISTERS": { "place": 0, "left": 52, "doff": 53 },
    "TABLES": { "lbase": [3, 4, 5, …] },
    "ARRAYS": { "lens": { "SIZE": 320 }, "fixl": { "SIZE": 288, "INIT": [8, 8, …] } },
    "HUFFS": ["litlen", "dist", "codelen"],
    "ENTRY": "@stream",
    "CODE": [
      "# the loader stub as it is, up to the loader table's magic",
      "@stream",
      "byterun 12",
      "@scan",
      "jz (and (eq (u32 (sub place 12)) 0x506C4472) (eq (u32 (sub place 8)) 0xE6CD5374)) @next",
      "decode: set left (shr (inleft) 3)",
      "code: set left (xleft)",
      "…" ] },
  "LAYERS": [] }
```

### 22.16.1 Fields

| Field | |
|---|---|
| `NAME` | what it describes (shown in tools and logs) |
| `ISA` | the instruction set it needs (1–9); a reader refuses a newer one. 3: `huffrun` `byterun`; 4: `(streampos)`; 5: `predict`; 6: `(u16)` `(u32)` `(u16be)` `(u32be)`; 7: the `BITS16M` coder, `rawbytes`, `(div)` `(mod)`, a bounded `huffrun`, codes up to 1024 symbols and 16-bit lengths; 8: `e8frame` `e8end`, `nearest`; 9: the LZX family (`lzxfolder` `lzxend` `lzxsize` `lzxemit`) |
| `CODER` | `BITS` (an LSB-first bit coder, deflate's: `raw`, `huff`, `huffrun`, `byterun`, `rawbytes`), `BITS16M` (ISA 7: bits MSB-first from 16-bit little-endian words, LZX's, the words counted from where the bits begin: after bytes the next bits start a new run of words where they stand, as a container's bitstream begins wherever its block does; the same instructions; `byterun`/`rawbytes` at a word boundary of the bits before them) or `RANGE` (an LZMA-style range coder through adaptive slots: `bit`, `tree`, `rtree`, `direct`, `mlit`) |
| `SHIFT` | RANGE: how far adaptive slots move (1/2^SHIFT; default 5) |
| `SLOTS` | RANGE: how many adaptive slots |
| `WINDOW`, `MAXLEN` | an LZ format's window and longest copy (the generic predictor's bounds) |
| `LEN1SLOT`, `HISTORY`, `TOKEN` | the generic predictor's view of an LZ format's tokens (`HISTORY: {FROM, COUNT}` repeat-distance registers; `TOKEN: {KIND, LENGTH, SLOT, DISTANCE}` registers) |
| `REGISTERS` | names for registers 0–63 (`{ "left": 52 }`); register 0 is the place in the content |
| `TABLES` | named constant tables (lists of numbers) |
| `ARRAYS` | named arrays (`SIZE`, `INIT`: the first values, the rest 0), written by `sta`, read by `(arr …)` |
| `HUFFS` | names of the Huffman codes `build` makes |
| `ENTRY` | the label the stream starts at; `TOKEN_ENTRY`: the token program a PRICE run calls |
| `CODE` | the instructions, one per string; `@name` alone is a label; `# …` is a comment |

Tables, arrays and codes are named, and named only: JSON objects have no key order a reader must keep. Tables and
arrays are numbered in sorted-name order (records store arrays by number), so a description reads alike whatever order
its text lists them in; `HUFFS` is a list (its order is its numbering).

### 22.16.2 Modes

One description runs three ways: DECODE (shipped bytes → content + decisions), CODE (content + decisions → shipped
bytes, bit-exact) and PRICE (the cost of a candidate decision, for prediction). An instruction runs in every mode unless
prefixed: `decode:`, `code:`, `price:`, joined by `+` (`code+price: set e (sub l (table lbase c))`). A coding instruction
READS its value in DECODE and WRITES it in CODE; `free` makes the value a decision (CODE takes it from the decisions,
DECODE records it).

### 22.16.3 Values

Numbers (`12`, `0x1A626C7A`), registers (`r5` or a name), and `(op value …)`, nested at most 200 deep. All arithmetic
is 32-bit unsigned, wrapping; comparisons give 0 or 1; shifts by 32 or more give 0.

| Value | |
|---|---|
| `(add a b)` `(sub a b)` `(mul a b)` `(shl a b)` `(shr a b)` `(and a b)` `(or a b)` `(xor a b)` | arithmetic |
| `(lt a b)` `(eq a b)` `(ne a b)` `(ge a b)` | comparisons |
| `(div a b)` `(mod a b)` | ISA 7: quotient, remainder (b 0: the run is refused) |
| `(byte a)` | the content byte at place a (0 outside what exists; in DECODE, only bytes before the place exist) |
| `(u16 a)` `(u32 a)` `(u16be a)` `(u32be a)` | ISA 6: an integer at place a, little- or big-endian (each byte as `(byte)` reads it) |
| `(table NAME a)` `(arr NAME a)` | a table's or an array's entry (out of range: refused) |
| `(log2slot a)` | 2·⌊log2 a⌋ + the next bit (a below 4: a): LZMA/deflate distance slots |
| `(bitpos)` | the bit coder's position within its byte (`BITS16M`: within its 16-bit word, 0 right after bytes; word-align = `raw free (and (sub 16 (bitpos)) 15) r`) |
| `(dleft)` `(xleft)` `(inleft)` | decisions left (CODE), content bytes after the place, shipped bits left (DECODE) |
| `(streampos)` | ISA 4: the shipped stream's byte position |

### 22.16.4 Instructions

| Instruction | |
|---|---|
| `set r v` | register r = v |
| `dec r` | register r ↔ the decisions (CODE reads one, DECODE records it) |
| `jz v @l` · `jmp @l` · `call @l` · `ret` · `end` | control (calls nest at most 64 deep) |
| `assert v` | DECODE refuses a stream breaking a rule of its format (and every mode stops on a false one) |
| `mark` | a place a checkpoint may be taken (between whole decisions): a record's segments start at marks |
| `flush` | the coder writes out what it holds (CODE) |
| `outlit v` · `outcopy len dist` | content: a byte, a copy from `dist` back (DECODE writes, every mode advances the place) |
| `byterun n` | BITS: n content bytes through the stream as they are (a stored region; copied whole when byte-aligned) |
| `raw [free] n v r` | BITS: n raw bits (≤ 32; `BITS16M`: the first read is the value's highest bit) |
| `rawbytes [free] n v r` | ISA 7: n (1–4) stream bytes as a little-endian integer, at a byte boundary (`BITS16M`: a word boundary of the bits before it, or after bytes) |
| `huff [free] H v r` | BITS: a symbol through code H |
| `huffrun H n r term lim [max]` | BITS: a run of literals through code H; the symbol ending it (≥ lim) goes to register `term`, the run's length to r. ISA 7: `max` bounds the run (formats with no end symbol: LZX): DECODE also stops after `max` literals, `term` then 0xFFFFFFFF, and CODE then writes no ending symbol |
| `build H A x n` | code H from array A [x, x+n) of code lengths (canonical; over-subscription refused) |
| `sta A i v` | array A [i] = v |
| `bit slot v r` · `tree nb slot v r` · `rtree nb slot v m r` · `direct n v r` · `mlit slot v match r` | RANGE: a bit through an adaptive slot; an nb-bit tree (MSB first / LSB first, `m` the bits when nb is 0); n direct bits; an LZMA matched literal |
| `hmtf r n v` · `hpush r n v` | a history of n registers from r: move entry v to the front; push v |
| `e8frame at n cur rFS` · `e8end t` | ISA 8, a TRANSLATED VIEW (LZX's E8): frame [at, at+n) of the content, its first byte at stream position `cur`, the filesize in register rFS; `e8end` says whether it was translated (DECODE decides there, CODE learns it at `e8frame`: one decision). The format's code works on what the encoder saw (copies, `(byte)`, literals read the translated bytes); the content is the real bytes (the E8 inverse, frame by frame: last 10 bytes untouched, the 4 after an E8 skipped), so it dedups with the installed files. Limits: a frame is 1 byte–1 MiB, in order, never overlapping, `at+n` and `cur+n` within 2^32, the filesize below 2^31 (else refused). The frames the code may still read are those ending after `place − WINDOW` (older ones are dropped when a frame registers, and checkpoints carry the rest as runs); more than 4096 of them is refused. A family (`predict`) cannot yet read through a view: a description using both is refused |
| `nearest r len lo maxd` | ISA 8: register r = the NEAREST distance (1..maxd, never before content place lo) at which the content repeats the `len` bytes at the place (`len` ≥ 2; newest candidates first, at most 128 looked at), or 0xFFFFFFFF; `maxd` may not exceed the description's `WINDOW` (refused). A predictor like `predict` (known when coding; when decoding, only in a mint's second pass): an LZ format's description stores a flag where it matches the stream's distance and the distance where not — match finders hand out nearest distances (LZX: 99.999%) |
| `lzxfolder at n e8 knob r` · `lzxend t` | ISA 9, the LZX ENCODER FAMILY (Microsoft Cabinet Maker 1.00's compressor, reimplemented from the binary; it reproduces MAKECAB's cabinets block for block, windows 15–21, its window slides and its change of window between folders): folder [at, at+n) of the content (its real bytes, E8 translation done by the compressor with file size `e8`; `knob` = window bits 15–21 \| second partition in 32 KiB frames (1–32) << 8, MAKECAB's: `21 \| 2 << 8`). ONE decision says the folder is the family's: DECODE reserves it at `lzxfolder` and fills it at `lzxend t`; it is the family's when t ≠ 0 AND the family made every frame it was asked for, and then every decision since is dropped. CODE reads it; r = 1 when the family is at hand (CODE: the decision says so; DECODE: a mint's second pass only), else 0xFFFFFFFF (then the description codes the folder as it decodes it). A plain record never takes the family's frames (refused). Folders run in order, never overlap, at most 4096; the family keeps state from folder to folder (its frequency tables even across a change of window, as MAKECAB does), so it makes them in order, fed only as far as the frames asked for; it keeps 64 MiB of frames a coding run has written, and up to 512 MiB of frames none has yet (older ones are made again from the stream's start when asked for: paid once, and such rework is bounded at 8 x the stream's account). Its work is the stream's account and at most 64 steps a folder byte (real data: 7–35): past either, refused. `lzxend` with an E8 frame open is refused; it closes the folder |
| `lzxsize k r` · `lzxemit k r` | ISA 9: frame k of the open folder: its payload size (or 0xFFFFFFFF: no such frame, or the family not at hand), and its payload: CODE writes it to the stream (at a byte boundary; a folder that is not the family's, or no frame k, is refused), DECODE compares it with the stream at its position (r = 1 when they are equal). A description checks every frame (`lzxemit`, the sizes) and that there is no frame past the folder's last (`lzxsize` of its count is 0xFFFFFFFF) before it calls the folder the family's, and records again after `lzxend` whatever the family does not make (CAB: each block's checksum) |
| `predict F knob r at lo hi` | ISA 5: registers r, r+1, r+2 = encoder family F's token at content place `at`, from its parse of [lo, hi): F is `lz4hc` (knob: attempts; 32768 = Bethesda's packer) or `zlib` (knob: level \| memLevel << 8 \| windowBits << 16) |

### 22.16.5 Canonical form

A description reads into one canonical form: each value laid out left to right, children first, each distinct value
once. `coding check` prints it back with the author's names; reading what it prints and printing again gives the same
text. Two texts that read to the same canonical form run alike, byte for byte (the same records).

### 22.16.6 Writing one (packaging work)

1. Learn the format from its spec and samples; walk them with a script until every byte is accounted for (regions,
   streams, their lengths, gaps).
2. Identify every compressed stream's encoder exactly (re-encode the decoded data until it matches): a match is an
   encoder FAMILY (records shrink 100–10,000×); none is fine (every decision stored: exact, larger).
3. Write the node; `VidyaGod coding check <node>` until it reads; `coding mint <node> <shipped> <content> <record>`
   (bit-exact or refused); `coding verify`; `coding expand` for a predicted record's plain twin.
4. Rules learned: read only bytes before the place (DECODE has nothing after it); keep DECODE and CODE symmetric
   (`decode:`/`code:` pairs compute the same thing from what each mode has); stored regions are `byterun`; anything a
   walk does not understand ends in "the rest as it is", never a refusal of the file; `mark` at every unit boundary a
   reader may want to start from.

Built-in examples, ported from the engine's former builders and verified record for record: deflate, deflate with the
zlib family, Inno Setup (zlib chunks predicted, LZMA2 chunks walked), LZ4 block, BSA 105 with and without the LZ4HC
family, LZMA.

A family that makes a WHOLE unit (ISA 9, LZX: a CAB folder) is used like this (the engine's test copy:
`VidyaGodFS/test/data/cab_lzx_family.json`, made from the Packagizer's CAB + LZX description):
- at the unit's start, its size from the format's own tables (CAB: the furthest file end in CFFILE), then `lzxfolder`;
- DECODE decodes the unit as it always does and, at each frame, checks it against the family's (`lzxemit`, `lzxsize`, the
  frame's uncompressed size), keeping what the family does not make (the checksum) in an array; at the unit's end,
  `lzxend` with the verdict, and when it is the family's, `dec` of each kept value;
- CODE, when the decision says the family's, writes each of the folder's frames (the CFFOLDER count): its header
  (`rawbytes free 4` the checksum, `lzxsize`, the uncompressed size), `lzxemit`, the place moved on by the frame's bytes,
  and a `mark` per frame.
The four AoE2 cabinets (90–95 MB each): records of 79–82 MB (every decision) become 25–26 KB, bit-exact, every installed
file identical (2026-11-27).
