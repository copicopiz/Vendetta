# 5. `ttpf` texture stores

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

`data/ttdb/**/*.ttpf`: 57 stores (maps, terrain, props, peds, vehicles, UI,
weapons…), magic `RDS5`. Each store is a virtual byte space (can exceed
4 GiB) split into independently compressed chunks, plus an index keyed by
the **64-bit texture hash that drawable shaders reference** ([§7.6](07-models.md)). Most
map and vehicle textures live here, not in texture dictionaries.

## Container and texture dispatch

Treat the outer container and its payload as separate decisions. An RPF9
is a general archive whose entries can be different resource and plain-file
types; its encrypted TOC is not a texture index. A TTPF in the observed
layout is a texture store, but it can mix texture formats, codecs, mip
chains, tile modes, and 2D, array, cube, or 3D surfaces. Do not select one
decoder or layout solely from the filename. For RPF entries, classify the
decompressed payload by its signature/resource header; for TTPF, dispatch
from each ETPF descriptor and chunk-table codec, then apply the matching
surface layout.

**Content-pack stores** (confirmed): `pack/<name>/data/ttdb/**/*.ttpf` holds
11 more stores (4,654 textures, about 1.2 GB) in `core_restofworld`,
`core_restofworld001`, `shared_minigame000` and `story000`, with the same
format. They hold the textures of the models in that pack's own archive
(`story000.rpf`: 4,156 models, `core_restofworld`: 814,
`shared_minigame000`: 668) and of 24 ped models in `packs.rpf`; no
`levels.rpf` model references them. Only 4 hashes are in both a pack store
and `data/ttdb`, with identical textures. On 40 sample models per pack,
using them cuts the missing textures from 944 to 135 (`story000`), 288 to
96 (`shared_minigame000`) and 344 to 234 (`core_restofworld`, where most of
the rest is one placeholder hash, `5d7a3a87cc19967d`, used by gore and body
shaders and found nowhere in the install).

## Header

| Off | Meaning | Confidence |
|-----|---------|-----------|
| 0x00 | `RDS5` | confirmed |
| 0x04 | u32 n = chunk-table entries (last is a zero terminator) | confirmed |
| 0x08 | u32 virtual size (low 32 bits only) | confirmed |
| 0x0C | u32 0x16000000, or 0x16000001 on 18 stores (mostly the largest); meaning unknown | confirmed values |
| 0x10 | u32 ?, u32 0 | unknown |
| 0x18 | chunk table, 8 bytes per entry | confirmed |
| `align_up(0x10 + 8n, 0x100)` | `ETPF` texture index | confirmed |

## Chunk entry (u32 a, u32 b)

| Field | Formula | Confidence |
|-------|---------|-----------|
| file offset | `(a & 0xFFFFFF) << 8` | confirmed |
| virtual offset | `0x1000 + (b & 0xFFFFF) × 0x4000 + (a >> 30) × 0x1000` | confirmed (2,810 chunks contiguous) |
| codec id | bits 24–29 of `a` (table below) | confirmed |
| `b >> 20` | trailing zero padding of the chunk, in bytes: stored length = next file offset − this file offset − (`b >> 20`) | confirmed (541/541 sampled zlib chunks end exactly at the RFC 1950 end-of-stream; 535/535 sampled BCPack chunks) |

Use the indexed length, not a length found by stripping trailing zeros: some
zlib streams end in a real `0x00` byte (2 of 541 sampled), and stripping cuts
them short.

A chunk runs to the next entry in both spaces. Missing the 4 KiB
sub-offset in `a >> 30` puts a quarter of chunks 4–12 KiB early, which
looks like a scrambled swizzle.

| Codec id | Chunks | Content | Codec |
|---|---|---|---|
| 0x20 | 908,643 | any | zlib |
| 0x04 | 100,026 | BC1 | BCPack (decodes, [§6](06-bcpack.md#60-status)) |
| 0x0C | 35 | BC3 | BCPack (decodes, fitted tables, [§6](06-bcpack.md#62-bc3-blocks-codec-0x0c)) |
| 0x10 | 77 | BC4 | BCPack (decodes, fitted tables, [§6](06-bcpack.md#64-bc4-and-bc5-blocks-codecs-0x10-0x14)) |
| 0x14 | 325 | BC5 | BCPack (decodes, fitted tables, [§6](06-bcpack.md#64-bc4-and-bc5-blocks-codecs-0x10-0x14)) |
| 0x18 | 268 | BC6H | BCPack (decodes, fitted tables, [§6](06-bcpack.md#65-bc6h-blocks-codec-0x18)) |
| 0x1C | 5,120 | BC7 | BCPack (decodes, fitted tables, [§6](06-bcpack.md#63-bc7-blocks-codec-0x1c)) |
| 0x00 | 5 | any | stored (read as is) |

BCPack ids are the BC number × 4 (`0x04` BC1 … `0x1C` BC7); zlib is `0x20`.
Confirmed by joining every chunk to the texture whose data it holds and
reading that texture's descriptor format: `0x0C` chunks belong only to DXGI
77/78 (BC3) textures (35/35), `0x10` only to DXGI 80 (BC4) (77/77), `0x04` to
BC1, `0x14` to BC5, `0x18` to BC6H, `0x1C` to BC7.

The non-zlib ids are compressed, not raw pages: their payloads measure
about 7.95 bits/byte and do not match raw BCn block sizes (confirmed).

## `ETPF` index

+4 u32 version (0x27A), +8 u32 texture count t, +0xC u32 table length, then
t × 16-byte descriptors, then t × u64 texture hashes **sorted** (descriptor
i belongs to hash i), then more tables (a u32 list, t × 32-byte digests:
unknown).

| Descriptor field | Meaning | Confidence |
|---|---|---|
| u32 d0 bits 0–13 / 14–27 / 28–31 | width − 1 / height − 1 / mip count − 1 (on cube maps, tile modes 9–12, the field is the mip count itself: cubes marked "2" store one mip per face; *refined 2026-11-23: cube mip count = field + copies − 1, see [Arrays and cube maps](#arrays-and-cube-maps)*). The mip field **undercounts on about 44.9 K textures** ([below](#real-mip-count)) | confirmed (width, height); mip field unreliable |
| u32 d1 bits 0–2 | flags; on every cube = copies − 1 | unknown (cubes confirmed) |
| u32 d1 bits 3–9 | DXGI format | confirmed |
| u32 d1 bits 10–19 | array size − 1, or **depth − 1 on 3D textures** (0 on 356,418 of 357,115 textures). "Bit 10" alone is the low bit of this field: it is set on every array with an even size (598 textures). Sizes: 6 = cube maps (567); 2 (19), 3 (99), 8 (3), 16 (1), 32 (4), 48 (2), 64 (2) = texture arrays. Layouts in [Arrays and cube maps](#arrays-and-cube-maps) | confirmed (the sizes match the store spacing; every slice decodes to a separate, coherent image) |
| u32 d1 bits 20–23 | **resource kind**: 2 = 2D, 4 = cube, 6 = 3D, 10 = 2D array. The four values cover every texture | confirmed (2026-11-23) |
| u32 d1 bits 24–27 | tile mode | confirmed |
| u64 loc bits 0–20 | first 4 KiB page, counted after the 0x1000 header page (virtual offset = page × 0x1000 + 0x1000) | confirmed |
| u64 loc higher bits | ? | unknown |

Textures are packed tightly and small ones can sit inside a larger one's
padding, so **the size comes from the mip chain**, not the distance to the
next texture.

### Real mip count

The mip field (d0 bits 28–31) is wrong on about 44.9 K of the 352 K store
textures (confirmed):

| Case | Textures | Field says | Stored |
|---|---|---|---|
| tile mode 1 or 2, field 0 (mostly 1024 × 1024 terrain normal maps) | 41.5 K | 1 mip | 3 mips |
| tile mode 1 UI sprites, field 1 | about 2.7 K | 2 mips | 1 mip (in tile mode 1, field 1 always means 1 mip: store spacing says so on all 4,071 such textures where it can tell) |
| uncompressed formats, tile modes 1–3, field 1 | | 2 mips | mip 0 only |
| others | a few hundred | | off by 1–2 |

Mips are stored smallest first with mip 0 last, so a wrong count puts mip 0
at the wrong offset: a scrambled band of the smaller mips across the top of
1024 × 1024 terrain textures, or noise on small UI sprites.

**A single-mip surface has no mip tail** (2026-11-23, high): its mip starts
at offset 0 even when it would fit in a tail, since Gfx10 builds a tail
only for more than one mip. E.g. a 16 × 8 RGBA8 texture fills only the
256-byte blocks 0 and 2 of its 4 KiB block; the 2,249 tail-sized tile-mode-1
textures hold their data at offset 0. Why tile modes 2 and 3 use other
rules is open (mode 3: spacing gives field + 3).

Rule (2D textures and arrays): each texture's mip chain runs right up to
the next texture's virtual offset. Where the size given by the field does
not reach it and exactly one mip count does, use that count. Checked as
mip 1 = mip 0 downsampled: median 28–46 dB with the corrected count against
7–15 dB with the field's, and by eye on terrain and UI textures. Where the
true count is stored is unknown; it does not map to a single bit field.

## Layout: AMD Gfx10 (RDNA2) swizzle modes (confirmed)

| Tile mode | Gfx10 mode | Block | Element bit order in a block (low → high) |
|---|---|---|---|
| 0 | `SW_64KB_D` | 64 KiB | 16-byte: `x0 y0 x1 y1 y2 x2 y3 x3 y4 x4 y5 x5` · 8-byte: `x0 y0 x1 x2 y1 y2 x3 y3 x4 y4 x5 y5 x6` · 4-byte: `x0 x1 y0 y1 x2 y2 y3 x3 y4 x4 y5 x5 y6 x6` · 2-byte: `x0 x1 x2 y0 y1 y2 x3 y3 x4 y4 x5 y5 x6 y6 x7` · 1-byte: `x0 x1 x2 y1 y0 y2 x3 y3 y4 x4 y5 x5 y6 x6 y7 x7` (y1 before y0) |
| 1, 2, 3 | `SW_4KB_S` | 4 KiB | 16-byte: `y0 y1 x0 x1 y2 x2 y3 x3` · 8-byte: `x0 y0 y1 x1 x2 y2 x3 y3 x4` · 4-byte: `x0 x1 y0 y1 y2 x2 y3 x3 y4 x4` · 2-byte: `x0 x1 x2 y0 y1 y2 x3 y3 x4 y4 x5` · 1-byte: `x0 x1 x2 x3 y0 y1 y2 y3 y4 x4 y5 x5` |
| 9, 10, 11, 12 | `SW_4KB_S`, cube maps only: tile mode − 8 = number of stored copies (below) | 4 KiB | as modes 1–3 |

- Blocks are row-major; no pipe/bank XOR.
- Independent check of tile mode 0: decoded with these `SW_64KB_D` orders,
  textures are smoother than with the `SW_64KB_S` equations on 161/200 BC1/BC4
  and 191/199 BC3/BC5/BC7 textures (median block-edge error 5.2 vs 11.4 and
  2.9 vs 7.9).
- **Mips go smallest first; mip 0 is last.**
- **Mip tail:** a mip whose element size is ≤ (block width / 2, block
  height) and that is among the last `max_mips_in_tail` mips (8 for 4 KiB
  blocks, 12 for 64 KiB) is in the tail. All tail mips share one block at
  offset 0; the other mips follow, each padded to whole blocks.
- Tail mip k (0 = largest in the tail): `m = max_mips_in_tail − 1 − k`,
  `off = m > 6 ? 16 << m : m << 8`. Origin x = bits 9, 11, 13… of `off`
  × 8 (8-byte elements), × 4 (16-byte) or × 16 (1-byte); y = bits 8, 10,
  12… × 4, or × 16 for 1-byte elements. The unit is one 256-byte block:
  8 × 4 (8-byte), 4 × 4 (16-byte), 8 × 8 (4-byte), 16 × 8 (2-byte) or
  16 × 16 (1-byte) elements. Apply the block equation to (origin + x, y).
- **Uncompressed formats** (4-byte RGBA8 / BGRA8, 2-byte R16_FLOAT / RG8,
  8-byte R16G16B16A16 UNORM / FLOAT) decode with these orders in every tile
  mode (confirmed, 2026-11-21; 2,730 textures newly decodable). No
  (format, tile mode) pair in the stores is left undecodable. The 2-byte
  `SW_64KB_D` order is x-first by the block shape; square samples cannot
  tell it from its transpose (inferred).
- **R16_UNORM** (DXGI 56; also 53 and 57; 2026-11-23, high): 2 bytes per
  texel, laid out like the other 2-byte formats. The minigame pack's 12
  water height maps use it (the images are coherent height fields).
  Whether any texture uses the signed form for real data is open.
- **R16_FLOAT array at twice its size** (one texture, `fd954c597b21d41c` in
  `ttdb_graphics`: 256 × 32, 8 slices, tile mode 1, one mip; confirmed,
  2026-11-21; *later shown to be a 3D volume, see [3D textures](#3d-textures)*). Each slice takes 32 KiB instead of 16 KiB: every other KiB is
  zero. Each 4 KiB block holds 32 × 32 elements, with address bits 1–9 =
  `x0 y0 y1 x1 y2 x2 x3 y3 x4` and bit 11 = `y4` (bit 10 is the unused
  KiB). The content is smooth gradients, probably a lookup table. With it,
  every texture in the stores decodes apart from the BCPack codecs of
  [§6](06-bcpack.md).
- **1-byte formats** (A8 = DXGI 65, R8) decode with these orders in every
  tile mode (confirmed, 2026-11-21): 126 textures (119 A8 decal and mask
  textures, 7 R8); mip 1 matches mip 0 downsampled at a median of 63 dB.

Checked per store and format: mip n + 1 equals mip n downsampled (mean
error 0–4 levels) on BC1, BC3, BC4, BC5 and BC7 in every mode.

## Arrays and cube maps

The array size is d1 bits 10–19 + 1. Arrays and cube maps are stored in two
different ways (both confirmed).

**Texture arrays (tile modes 0 and 1; 2–64 slices; 130 textures)** are stored
level by level, smallest first, with each level's slices back to back:
the mip-tail block of every slice, then the smallest non-tail mip of every
slice, and so on, ending with mip 0 of every slice. With the single-slice
layout above:

```
address(mip m, slice s) = offset(m) × slices + s × size(m)
```

where `offset(m)` and `size(m)` are the single-slice offset and padded size
(the whole tail block for tail mips). The total is slices × the single-slice
size, which matches the store spacing exactly (e.g. 256×256 BC5, 9 mips,
32 slices = 4 MiB). Decoding slice 0 with the single-slice layout reads
other slices' tail blocks where mip 0 should be, which is the mosaic. Seen:
cloth and detail-map sets (48 slices), pattern sets, and 2–3-slice ped face
wrinkle normal maps.

**Cube maps (tile modes 9–12; 6 faces; 567 textures)** use `SW_4KB_S` and
store **tile mode − 8 complete copies of the cube**, smallest first. Split
the mip chain into *units*, smallest first: the mip-tail block (if any),
then each non-tail mip. With u units and c = tile mode − 8 copies, copy k
(0 … c − 1) holds the smallest u − c + 1 + k units. Inside a copy the faces
follow each other, each face with its own chain (tail block, then the
non-tail mips up to that copy's top mip, as in the single-slice layout). The
last copy is the full-resolution cube.

- Each copy is one chunk of the store. A copy larger than 128 KiB is padded
  to a multiple of 64 KiB; smaller copies are not padded. So the full copy
  starts at the sum of the padded sizes of the copies before it, which is
  also the virtual offset of the chunk that starts c − 1 chunks after the
  texture. The two agree on all 533 cubes with a supported format.
- Example, 512×512 BC1, 7 mips (128×128 elements; tail + 32×32 + 64×64 +
  128×128 elements = 1 + 2 + 8 + 32 pages per face), tile mode 12: copies
  of 6, 18, 66 (padded to 80) and 258 pages; the full copy starts at page
  104.
- Example, 128×128 BC7, tile mode 11: copies of 6, 12 and 36 pages (padded
  to 48); the full copy starts at page 18.
- Tile mode 9 (one copy) is a plain face-by-face cube: e.g. 16×16 and 32×32
  cubes, whose whole chain is one tail block per face.

**Cube mip count** (2026-11-23, high): mip count = d0 mip field + copies −
1, with copies = tile mode − 8 (d1 bits 0–2 = copies − 1 on every cube). On
every cube the data runs exactly to that mip and the next tail slot is
empty (64 × 64 BC7 field 4: 42 of 42 faces; 128 field 4: 18 / 18; 256
field 2: 5 / 6). On 43 64 × 64 BC7 field-6 cubes the restored 1 × 1 mip
matches the box-filtered 2 × 2 mip within 2.20 levels on average. Open:
the u64 loc bits 21 and up on cubes (they look like packed per-copy
values).

Seen: light cookies (`lighting.ttpf`), sky/star cubes and interior room
cubes (shop walls, ceiling and floor).

## 3D textures

(2026-11-23.) Resource kind 6 (d1 bits 20–23); the array-size field gives
the depth. They use the **3D `SW_4KB_S` pattern** (non-XOR); element
address bits, low first:

| Element size | Bits | Brick |
|---|---|---|
| 8 bytes | `x0 z0 y0 x1 z1 y1 x2 z2 y2` | 8 × 8 × 8 |
| 2 bytes | `x0 z0 y0 z1 y1 x1 z2 y2 x2 z3 y3` | 8 × 16 × 16 |

Bricks are 4 KiB, ordered x, then y, then z. High for 2- and 8-byte
elements; other sizes are not measured. The stores hold three volumes:
R16G16B16A16_FLOAT 32 × 64 × 32 and 32 × 32 × 32, and R16_FLOAT 256 × 32 ×
8. The R16_FLOAT one is the "array at twice its size" above: depth 8 leaves
z3 = 0, which is the "every other KiB unused" pattern. Slice-to-slice
differences drop from 37.5 to 5.7, 85.6 to 5.2 and 42.4 to 0.36, with no
seams every 8 rows.

## Recovering individual plain mips

A texture with mixed codecs can have readable individual levels.
Compute the tiled surface layout from its format, dimensions, tile
mode, mip count and slice count. Map its chunks to decoded byte ranges
using the logical chunk index. Check that the sum of the declared
decoded lengths equals the expected surface size.

For a chosen mip and slice, use only the chunks covering its tiled
byte range, including the tail block where applicable. The coverage
must be contiguous and wholly within plain zlib chunks. Inflate every
covered chunk and require its decoded length to match the logical
index before gathering and detiling the mip. A codec tag alone is
insufficient to validate a chunk. Report levels independently if any
chunk cannot be decoded.

## BC6H textures and decoder pitfalls

(2026-11-23.) BC6H textures in the stores are **standard BC6H**: UF16
(format ids 94 / 95) and SF16 (96). A decoder written from the BC6H
specification gives clean images (high confidence). Three mistakes are
easy to make, and at least one widely used decoder makes all three; each
gives wrong HDR lighting textures:

| Pitfall | Correct | Wrong result |
|---|---|---|
| unsigned unquantize | only the top endpoint value `(1 << bits) − 1` maps to `0xFFFF` | mapping `1 << (bits − 1)` there makes half-range endpoints full-bright, so dark-to-mid gradients turn bright |
| 16-bit one-region mode (mode code 15: 16-bit base, 4-bit deltas) | mask `(1 << 16) − 1` computed in at least 32 bits | computed in 16 bits the mask is 0, so the second endpoint decodes as 0 |
| signed format (SF16) | two's-complement sign extension of the endpoints | taking `value & 0x7FFF` as a magnitude after extension makes negative endpoints wrong |

A fourth, smaller one: in the signed format a negative value that
truncates to zero must decode as +0, per the specification's unquantize
rule (high).

The first two are high confidence; the signed case is high from the
specification but only 25 signed textures were available to look at.

**Effect on the lighting store** (`data/ttdb/lighting/lighting.ttpf`,
mip 0, high confidence, measured on the whole store): 1,159 textures, 1,156
decodable (3 have mip 0 only in BCPack chunks): BC7_UNORM 714, BC6H_UF16
401, BC6H_SF16 25, BC1_UNORM 10, BC7_UNORM_SRGB 6. Fixing the three
pitfalls changes 234 of 401 UF16 and 17 of 25 SF16 textures (BC7 and BC1
unchanged). The median changed texture differs in 0.06 % of its pixels,
but 243 of the 251 have a pixel off by 64 or more levels (8-bit, clamped to
0..1), and two (a UF16 1024² and an SF16 512²) were garbled over 94–98 % of
their pixels.

Open: BC6H textures in the other stores (effect sprites, colour ramps) were
not compared; BCPack BC6H output goes through the same block decoder, so it
changes too (not checked separately); one SF16 2048 × 1024 texture looks
like noisy line art with either decoder (maybe a data texture); what each
lighting texture is for is not established.

## Coverage (all 57 stores, 352,461 textures)

**Since BC1 BCPack decodes** (2026-11-21; chunk-table check over the 46
stores in the main install): mip 0 is readable for **98.4 %** of textures
(86.8 % before). Maps 99.2 %, props 99.5 %, peds 99.2 %, interiors 99.7 %,
vehicles 98.9 %, terrain 97.4 % (67.6 % before), UI 91.1 %. Flat fallbacks
fall from 29,310 to 1,309 textures and low-mip exports from 14,443 to
1,410. The table below is from before BC1 decoded:

| | Textures |
|---|---|
| whole mip chain in zlib chunks | 298,332 (84.6 %): vehicles 96.7 %, maps 91.8 %, props/peds/interiors ~90 %, terrain 66 %, UI 57 % |
| chain touches BCPack, mip 0 still in zlib | 6,966 (2.0 %) |
| mip 0 in BCPack, a lower mip in zlib (exported at that mip) | 14,441 (4.1 %) |
| every mip in BCPack | 29,310 (8.3 %) |
| unsupported 4-/2-/1-byte formats or special tile modes 9–12 (all decode since) | 3,412 (1.0 %) |

The coverage table predates the array and cube-map layouts. With them, 563 of
the 567 cube maps and 118 of the 130 arrays decode (the rest have 4-/2-byte
formats, except three cubes with more copies than their mip chain has units).

**Lower-mip recovery:** chunks are 4 KiB–4 MiB ranges, not whole textures,
and mips go smallest first, so the small mips of a BCPack texture often sit
in zlib chunks. SixNG exports the largest mip that lies entirely outside
BCPack chunks (tail mips count as the whole tail block).

**Who is hit** (before BC1 decoded; the BC1 cases below now decode):
BCPack hits linear `BC1_UNORM` almost only (normal, packed and mask data),
not colour. Over all materials in `levels.rpf`, 37.9 % have
a flat slot and 22.6 % a low-mip slot, but weighted by placements that is
5.0 % and 3.5 %; only 7,088 materials have a flat colour slot. Terrain is
worst: its layer normals and its RGB layer-blend mask are often flat.

## Open questions

- Where the descriptor stores the real mip count ([Real mip count](#real-mip-count)).
- Format 157 (one vehicle texture slot) is unknown.
- Three cube maps declare more copies (tile mode − 8) than their mip chain
  has units (256×256 with "3" or "4" mips, 128×128 with "3"); how they are
  stored is unknown.
- The other descriptor flag bits, the digest tables.
- The 64-bit texture hash function. [structure/05](../structure/05-ttpf-stores.md)
  reports one plain joaat case (`hauler` → `0x5A82F9AE`). A check of every
  index hash in 46 stores found no hash whose low or high 32 bits equal that
  value, so this is unconfirmed.


---

← [4. Xbox Series texture tiling (dictionaries)](04-xbox-tiling.md) · [Index](README.md) · [6. BCPack](06-bcpack.md) →
