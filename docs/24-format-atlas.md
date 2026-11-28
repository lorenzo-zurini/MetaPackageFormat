# 24 · Format atlas

> **Status (2026-11-28).** Informative, kept current. Every compression codec, archive container, installer and disc
> filesystem met in the reference library and its original media, with how MPF reads it: as a **description** (data
> run on the format interpreter, chapter 22), as a **view** (a listing of byte ranges, no decoding), as a **leaf**
> (stored as it is), or not yet. A format is added to the atlas when it is met, and its row changes as it is cracked.
> The rule that drives it: every archive is decoded as far as a description can take it, and a missing piece is a new
> general primitive, never code shipped in data.

## 24.1 How to read the atlas

A **description** turns a stored stream into its **content** (what the stream holds, decoded) and a **record** (the
decisions its encoder made), and back, bit-exact (chapter 22). Content deduplicates across versions, mods and packages;
the record is usually tiny. A **family** is a reimplementation of a real encoder, run in the description, so the record
holds only where the stream departs from it (zlib level 6, MAKECAB 1.00, LZ4HC 32768): with a family the record
shrinks from megabytes to bytes. A **view** describes a container that compresses nothing: it lists its files as byte
ranges, and a `TREE` layer (chapter 04) places them. A **leaf** is a file decoding cannot help (lossy media, an
already-minimal stream).

| Mark | Meaning |
|------|---------|
| ✓ | described, bit-exact on real samples from the library; the ISA version it needs |
| ✓ view | a view, checked file-for-file against an independent reader and an established tool |
| ⧗ ISA 10 | cracked; its description waits on an instruction-set primitive (named) |
| ✎ | a draft description exists, checked by hand or on a pre-transformed sample |
| ✗ | met, not yet cracked |
| — | a leaf, or a wrapper that is never a base |

Counts ("packages") are from the 2026-11-28 census of the reference library and its original media.

## 24.2 Compression codecs

| Codec | Status | Families (encoder reimplemented) | Found in | Notes |
|-------|--------|----------------------------------|----------|-------|
| Deflate (raw, zlib, gzip) | ✓ ISA 7+ | zlib level 6, zlib level 9 | ZIP, gzip, CAB MSZIP, BSA 103/104, RoN BIG, SWF, Inno, Fallout 2 DAT2, UnityFS | The level is per file: builders keep the smaller record. MindVision's (Installer VISE) is not zlib: no family, the record holds every decision (Oni: 117 MB). |
| LZX (MS CAB) | ✓ ISA 9 | MAKECAB 1.00 | MS CAB (49 packages in the library, 7 on media), DirectX redistributables | The encoder oracle reimplemented from MAKECAB's output; frequency tables carry across folders (NextFolderWindow). AoE2's cabinets: records 80 MB → 25 KB. |
| MSZIP (MS CAB) | ✓ ISA 7+ | zlib level 6, zlib level 9 | MS CAB, 17 media packages | A per-block deflate with a reset dictionary. |
| Quantum (MS CAB) | ✗ | — | Starsiege: Tribes | Needs a multi-symbol adaptive arithmetic decoder: a primitive asked for, lowest priority (one package). |
| LZ4 / LZ4HC | ✓ | LZ4HC 32768, 4096, 256 | UnityFS bundles, BSA 105 (Skyrim SE) | Unity's encoder matches LZ4HC only partly (RimWorld 81 MB → 2.08 MB of records). |
| LZO1X | ✓ | — | Unreal Engine 3 packages (Dishonored, BioShock Infinite) | Records hold every decision (Dishonored 21.5 MB → 7.5 MB); an LZO1X family would shrink them. |
| EA RefPack | ✓ | — | EA BIG/VIV (FIFA 2005, Need for Speed) | FIFA 2005: 16,982 files, 344 MB → content 1.45 GB, records 162 MB. EA's encoder is not yet a family. |
| Team17 LZ | ✓ | — | Worms Armageddon IMG | Reverse-engineered: the long form is next + 18 at distance d. 176/176 files. |
| LZW (MSB-first, 9-bit start) | ✎ ⧗ ISA 10 `swap16` | — | Resident Evil PC PAK (1,112 files) | Reset/widen/end codes 0x102/0x101/0x100. Bit-exact on a pair-swapped copy today. |
| PKWARE implode (DCL) | ✗ (cracked outside the ISA) | — | MPQ 0x08 sectors, Arx Fatalis PAK | A blast.c port exists in the decoding prototype; its ISA port comes with MPQ. |
| LZMA / LZMA2 | ✎ ⧗ ISA 10 range-coder restarts | the LZMA family (planned) | 7z, xz, Inno 5.2+ (LZMA1/LZMA2), NSIS, UnityWeb | A plain LZMA description exists (lc 3, lp 0, pb 2, 8 MiB, end marker); real installers need range-coder restarts, byte ops and slot resets. |
| bzip2 | ✗ ⧗ ISA 10 `swap16` | — | Strife VE sources, TimeSplitters FP | MSB-first bits: `swap16` + 16-bit MSB reads. |
| zstd | ✗ ⧗ ISA 10 reversed view | — | Zelda TotK romfs patches | Its entropy streams are read backwards. |
| x86 call filter (BCJ, Inno) | ⧗ ISA 10 view kinds | — | GOG Inno installers (.exe/.dll chunks) | A reversible transform before LZMA2: a view kind (`bcj-x86`, `inno-5200`, `inno-4108`). |
| E8 call translation (LZX) | ✓ ISA 8 | — | LZX streams with E8 preprocessing | The `E8 view` primitive (nearest) of ISA 8. |
| FLAC | ✎ described (decode: 260/260 files' PCM equal `flac -d`, every subframe type, all four stereo modes, RICE/RICE2, escapes) ⧗ a libFLAC family | libFLAC 1.4.3 / 1.5.0: `-5 -r 3,3 --blocksize=4096` (AoE2, 20/20 files), `-8 --blocksize=4096` (MW2 sets, 191/191) | AoE2 and MechWarrior 2 soundtracks | Every file's PCM is byte-exact to its disc's audio track: content = the track (a TREE reference through the WAV view), record = the encoder settings and the metadata blocks (STREAMINFO computed, VORBIS_COMMENT, PADDING). Needs the libFLAC encoder as a family: its rules are measured (Rice parameter 14,095/14,095 partitions, partition order 600/600, the stereo and precision rules); the LPC analysis (autocorrelation → Levinson → quantisation) must be a bit-exact port, so the ISA's floating point must be deterministic across machines. Without the family, the record holds 16.65 M decisions over the 260 files. New primitives: the BITS8M coder, `sar`. |

## 24.3 Archive containers

| Container | Status | Codec inside | Found in | Notes |
|-----------|--------|--------------|----------|-------|
| ZIP (pk3, pk4, jar) | ✓ | stored, deflate | 100+ packages, 1,577+ members | The most common container. |
| Microsoft CAB | ✓ | LZX, MSZIP, stored; Quantum ✗ | 49 library + 17 media packages | Reserve fields (DirectX cabinets) read; stored folders in both descriptions. |
| Bethesda BSA 103 / 104 | ✓ | zlib 6 | Oblivion, Fallout 3, New Vegas | FO3 Voices: 776 MB → an 8.6 KB record, every file in content. |
| Bethesda BSA 105 | ✓ (v1, v2) | LZ4HC 32768 | Skyrim SE | v2 accepts records that overstate an original size (2021-12-13 archives). |
| EA BIG / VIV | ✓ | RefPack, stored | FIFA 2005, Need for Speed | Whole-file RefPack is its own description. |
| Rise of Nations / Rise of Legends BIG | ✓ | zlib 6 | 73/73 archives | 609 MB → 4.36 MB of records. A packer bug (a stream cut at its uncompressed size) is stored as it is. |
| SWF (CWS) | ✓ | zlib 6, zlib 9 | Evoland | 64 MB → a 291 KB record. |
| UnityFS bundle | ✓ | LZ4 / LZ4HC | 9 packages, 205 members | UnityWeb (LZMA) is separate: ⧗. |
| Unreal Engine 3 package | ✓ | LZO1X chunks | Dishonored, BioShock Infinite | Every other Unreal game stores its packages plain: a view. |
| MPQ (Blizzard) | ⧗ ISA 10 side input + `sfilter mpq` | implode, zlib, bzip2 sectors; the Storm cipher | Warcraft III (the game, its maps), d2gl | Cracked first: the decoding prototype round-trips war3.mpq (436 MB) and a 1.16 GB WC3 zip byte-identical. What remains is the ISA port. |
| Fallout 2 DAT2 | ✎ ⧗ ISA 10 side input | zlib 9 | master.dat (333 MB), critter.dat (167 MB) | The directory is at the end; layout checked on CRITTER.DAT (4,635 entries, all zlib 9 exact). Highest value of the side-input formats. |
| Arx Fatalis PAK | ✗ ⧗ ISA 10 side input + `sxor` | PKWARE implode | Arx Fatalis | FAT at the end, XOR-scrambled. |
| CRI CPK | ✗ ⧗ ISA 10 reversed view | CRILAYLA (backwards) | Sonic Unleashed (154 members) | |
| Sonic CD ARCL, Rayman 2 LEVELS0.DAT | ✗ ⧗ ISA 10 `sxor` | scrambled | Sonic CD, Rayman 2 | Entropy 3.9 and 8.0: scrambled, then possibly compressed. |
| WAD (id), Build GRP, Quake PAK, Godot PCK, FMOD FSB, XACT XWB/XSB, Wwise AKPK/BNK, TimeSplitters P5CK, Unity .assets/.resS, Unreal packages (plain), GameCube RARC, Mohawk, Chromium .pak, Zeus / Caesar III / SimCity 3000 PAK | view (M2) | none | DOOM, Duke 3D, Quake, Interstate '76, BioShock, Burnout 3, Pikmin, Myst, … | Containers that compress nothing: a listing of byte ranges, then a TREE. |
| Not yet investigated | ✗ | ? | Cradle of Rome res.pak, Worms Armageddon resource.dat, Far Cry 2 .dat/.fat, Daggerfall PACKED.DAT, JJ2 Source.pak, Harry Potter QWC .dat, NFS MW .bin, Silent Hill 4 .bin, RE3 rofs | Unidentified by magic; investigated per package. |

## 24.4 Installers

An installer is never a game's files: it is how the files were shipped. Its data is a base (chapter 04); the install is
a delta or a TREE over its decoded contents.

| Installer | Status | Inside | Found in | Notes |
|-----------|--------|--------|----------|-------|
| Inno Setup (zlib chunks) | ✓ | zlib 6 | 15+ GOG installers | GOG installers mix zlib and LZMA2 chunks and filter .exe/.dll (TTD: 98 zlib / 106 LZMA2): the zlib chunks decode today (Jazz 2, Spear of Destiny, TTD bit-exact). |
| Inno Setup (LZMA1 / LZMA2) | ⧗ ISA 10 LZMA family + the Inno filter | LZMA, LZMA2, x86 filter | Total Annihilation: Kingdoms, Descent II, Planescape: Torment, Icewind Dale, … | |
| InstallShield 5 / 6+ cabinet | ✎ ⧗ ISA 10 side input | zlib 6, an obfuscation (`sfilter isobf`) | 17+ packages (Battlefield 1942/2/2142, GTA III/Vice City, NOLF 1/2, Black & White 1/2, Morrowind, RCT 1/2, …) | data1.hdr describes data1.cab: the side input. Header versions 5, 6, 7, 9 met. |
| NSIS | ⧗ ISA 10 range-coder restarts | non-solid LZMA per file | Air Guard, Elf Bowling 7 | |
| MindVision Installer VISE | ✎ ⧗ ISA 10 `swap16` | deflate in 16-bit big-endian words | Oni (ONISETUP.002, 348 MB: 95 streams) | Its TEST twin (pre-swapped) is bit-exact: content 641 MB, record 117 MB (MindVision's encoder is no family). |
| Wise | — | deflate | stubs only (Tribes 2, BF2, Alice) | The game data is loose on the disc. |
| MSI (OLE compound file) | view (M2), then CAB | CAB streams | 24 library + 2 media packages | A view over the compound file, then the CAB description over its streams. |
| 7z | ⧗ ISA 10 LZMA family | LZMA/LZMA2 | Thief, System Shock (NewDark docs) | |
| RAR, RAR SFX | — | | 37 packages | Third-party wrappers of the original installer: extracted once, never a base. |
| InstallShield setup stub / compiled script | — | | 26 / 21 packages | Not install data (decoded later, for granularity). |

## 24.5 Disc images and filesystems

A disc image is the base of what was installed from it (chapter 04, D20): the install is a TREE over its filesystem's
view. Sector decoding comes first, the filesystem view over it.

| Format | Status | Checked on | Notes |
|--------|--------|------------|-------|
| CD sectors: Mode 1, Mode 2 Form 1/2, raw 2352, 2448 with subchannel | ✓ ISA 7+ | 14 + 11 + 6 + 3 media packages | EDC/ECC and subchannel Q/P computed: AoK track 1 → 81 KB, Crash Team Racing 606 MB → 147 KB, Claw MDF 608 MB → 125 KB. |
| ISO9660 | ✓ view (ISA 10 draft) | 28 discs, 18,217 files exact (PS1 ×11, PS2 ×7, MW2, AoE2, AoM) | One view reads cooked 2048-byte and raw 2352-byte images (`estride`). Entries whose extent lies outside the image (PS1 CD-DA tracks) are names with no bytes. |
| Joliet | ✓ view | 4 discs, 5,434 files exact | The type-2 descriptor with the escape; `NAMES: utf-16be`. |
| Rock Ridge | detected | none in the library | |
| UDF 1.02 (PS2 DVDs) | ✓ view | 6 discs, 1,721 files exact | CS0 names (`utf-16be`); the PS2 quirk of one short_ad holding a ≥ 1 GiB file's whole 32-bit length; multi-extent files (`epart`/`efin`). 7z refuses these volumes. |
| XDVDFS (Xbox) | ✓ view | 9 discs, 18,173 files exact | Partition found by magic (offsets 0x18300000, 0xFD90000, 0x2080000, 0); binary-tree directories; `NAMES: latin-1`. The library's images are trimmed XISOs, not Redump dumps. |
| GameCube GCM / FST | ✓ view | 9 discs, 19,143 files byte-identical to dolphin-tool | sys/ as Dolphin extracts it; `NAMES: shift-jis` (Pikmin's developers' names). |
| Wii partition | ✗ | Super Mario Galaxy | AES-128-CBC over the partition: the common key cannot be package data; it must come from the user's machine (an open design question). |
| CD-DA audio tracks | ✓ view (ISA 10 draft) | AoK, TC, MW2 31st CC, GBL: 66/66 tracks byte-exact | A WAV view: a computed 44-byte RIFF header + one byte range of the track from INDEX 01 (pregaps of zeros skipped). AoE2's songs are cut from ONE track at sample-aligned offsets: the offsets are data of the view. Needs an op appending computed bytes to an entry (`ebyte`/`eu32`, proposed). |
| Cooked 2048-byte images | — | 15 packages | No sector layer: the filesystem view reads them directly. |

**Measured (2026-11-28), the 28 disc packages ready for these views:** 35.8 GB of their 55.5 GB of installed content
(65%; 70% without the ports' own binaries) is the disc's own files, byte for byte, and becomes TREE references into the
base images. Near-pure installs: Age of Mythology 99%, Crash Team Racing 99% (its loose image is the Redump image),
Far Cry Instincts 99%, Ninja Gaiden Black 98%, JSRF 96%, Burnout 3 93%. Partial: TimeSplitters FP 76%, AoE2 69%, Jak
63–69%, God of War 50%, Vagrant Story 40%, MechWarrior 2 19%. Near 0%: recompilations that read the image directly,
and ports that repack their data.

## 24.6 Leaves

Never decoded: lossy video and audio (Bink, Ogg, MP3), compressed textures (DDS/BCn), PNG and JPEG, already-minimal
streams. They are content as they are; deduplication still applies to them whole.

## 24.7 Primitives and what they unblock

| ISA | Primitive | Unblocks |
|-----|-----------|----------|
| 7 | `BITS16M`, `rawbytes`, div/mod, bounded `huffrun` | the deflate, MSZIP and LZX bit readers |
| 8 | the E8 view (nearest) | LZX with E8 preprocessing |
| 9 | the LZX encoder family | MS CAB records from megabytes to kilobytes |
| 10 (planned) | side inputs (`SIDES`, `sbyte`/`su16`/`su32`/`slen`) | Fallout 2 DAT2, MPQ, InstallShield, Arx Fatalis |
| 10 (planned) | stream filters: `sxor`, `sfilter` (named, versioned generators: `mpq`, `isobf`) | MPQ encrypted files, InstallShield, Arx FAT, Sonic CD, Rayman 2 |
| 10 (planned) | `swap16` (a final odd byte passes through) | Installer VISE, Resident Evil PAK, bzip2 |
| 10 (planned) | range-coder restarts (`rcstart`/`rcend`), byte ops, slot resets; the LZMA family | Inno LZMA/LZMA2, NSIS, 7z, xz, UnityWeb |
| 10 (planned) | the reversed view (`rview`/`rend`) | CRI CPK (CRILAYLA), zstd |
| 10 (planned) | view kinds (`lzx-e8`, `inno-5200`, `inno-4108`, `bcj-x86`) | GOG Inno executables, filtered LZX |
| 10 (planned) | the VIEW mode and `TREE` layer: `ename`, `efinish`, `edir`, `echar`, `etab`, `estride`, `epart`/`efin`, `NAMES` | ISO9660, Joliet, UDF, XDVDFS, GCM, and every container in §24.3 that compresses nothing |
| — (asked, lowest) | a multi-symbol adaptive arithmetic decoder | CAB Quantum |

## 24.8 Adding a format

1. **Find it.** The census reads every container in the library and its original media by magic; an unidentified
   file is investigated in its package's pass. Every format met is cracked: none is given up on.
2. **Describe it** in the existing primitives where it can be. A primitive that is missing is added to the next ISA
   version, general (named after what it does, never after a game), with a one-line specification: inputs, output,
   failure.
3. **Find its encoder.** Where the real encoder is known (zlib, MAKECAB, LZ4HC), reimplement it as a family in the
   description, so the record keeps only the departures.
4. **Prove it** on real samples: decode, re-encode, byte-for-byte equal; for a view, every file against an
   independent reader and an established tool. Record the sizes (stream, content, record).
5. **Add its row** here, with what it was checked on.

Next: back to the [README](../README.md).
