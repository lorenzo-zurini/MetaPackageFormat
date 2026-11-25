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

A procedural oddity (name-keyed encryption, a seeded obfuscation) becomes a new named registry function, never script.

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
- **LZMA single streams:** checkpoints are not yet proven; their state is the range coder, the probability model and
  the bt4 match finder. Until proven, an LZMA piece is decoded only if its dedup win pays for a whole-unit re-encode.
  Otherwise it stays stored.

### 22.7.3 Reading (USER, 2026-11-25: "no cache"; "the skeleton map is the cache")

A read of bytes X..Y of a shipped archive:

1. Look up, in the unit map, the units covering X..Y.
2. Re-encode exactly those units from the decoded content with their recipes; skeleton bytes are copies.
3. Return the bytes. No unit larger than the mint-time bound (e.g. 256 KiB of decoded input) is ever re-encoded per read.

The RAM block cache holds recently produced units. There is no disk cache and no first-run reconstruction.

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
- LZMA checkpoints (22.7.2).
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
