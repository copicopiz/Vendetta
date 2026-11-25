# 13. Collision and surface categories

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

## 13.1 Bounds (GTA V `phBound` with small changes, confirmed)

About 87 K of ~98 K unnamed resources in `levels.rpf` are physics bounds
(`.ybn` equivalent).

| Off | Field |
|---|---|
| +0x10 | u8 type: **5 = geometry (BVH), 6 = composite** |
| +0x14 | f32 bounding-sphere radius |
| +0x20 / +0x30 | bbox max / min (world space); +0x2C f32 margin 0.005; +0x3C u32 ref count |
| +0x40 / +0x50 | bbox centre / centre of gravity |

**Composite (6):** +0x70 child pointer array, +0x78 and +0x80 child
transforms (0x40 bytes each), +0x88 child flags (16 B each),
+0x98 and +0x9A hold the same u16 child count. In 293 sampled composites,
counts span 1–13, both transform pointers alias, and all 509 child transforms
are identity. The u16 at +0x9C is 5 even when the count exceeds 5; do not use
it as a count or a limit.

**Geometry (5):** +0x88 polygons, +0x90 vec3 quantum, +0xA0 vec3 centre,
+0xB0 vertices (s16 × 3; **position = s16 × quantum + centre**), +0xD0 u32
vertex count, +0xD4 u32 polygon count, +0xF0 materials (8 B each), +0x118
per-polygon material index (u8), +0x120 u8 material count, +0x130 BVH
summary pointer. The target of +0x138 is still unidentified.

**Polygons** (16 B, type in byte 3):

| Type | Layout |
|---|---|
| 0 triangle | u16 v0, v1, v2 at +4 (top bit is a flag), i16 neighbours at +10 |
| 1 sphere | u16 vertex at +0, f32 radius at +4 |
| 2 capsule | u16 v0 at +0, f32 radius at +4, u16 v1 at +8 |
| 3 box | four u16 corners at +4, alternate corners; the other four mirror through the centre |
| 4 cylinder | as capsule |

**Material record** (8 B): **u16 surface id** at +0 (always < 0x8000), flag
bytes after it (unknown).

**Vertices outside the main stream** (2026-11-23, confirmed on the minigame
pack): either a small stream reached only by a pointer to its exact start,
or uncompressed bytes after the last zlib stream (16-byte aligned).

### CPU collision BVH

The geometry's `+0x130` points to a summary. Its `+0x50` points to
16-byte nodes and `+0x58` is the u32 node count. The following u32 can be
larger; do not walk it as a count. Each node holds `i16 min[3]`,
`i16 max[3]`, and two u16 words:

| `+0x0E` | `+0x0C` |
|---|---|
| Nonzero: polygon count | Index of the first polygon in this leaf |
| Zero: internal node | Subtree span in nodes, including this node |

The nodes are in preorder. At an internal node `i`, its left child is
`i + 1` and its right child is immediately after the left subtree (a leaf
spans one node). Skip the whole subtree with `i + span`. A one-node tree
is a leaf. Polygon ranges are in leaf order and partition the geometry's
polygon table; they are not restricted to triangles.

Decode node-box coordinates with `bound_min + ((q + 32768) / 65535) *
(bound_max - bound_min)` per axis of the owning geometry child. Vertices
use their own stored quantum and centre. The summary also stores copies
of the child's min/max at `+0x00`/`+0x10`, a centre at `+0x20`, and
reciprocal scale vectors at `+0x30`/`+0x40`.

Across 32 evenly spaced 8 MiB windows in `levels.rpf` (Xbox 1.0.2.1),
509 geometry bounds passed the subtree, AABB union and polygon-range
checks: 1,353,503 nodes and 1,998,333 polygons. The reconstructed shapes
(1,982,262 triangles, 8,039 capsules, 7,575 boxes and 457 cylinders)
all fit their owning leaf boxes within 2 cm. This is unrelated to
[GPU BVH4](17-other.md#gpu-bvh4).

**Raw vertex tails.** In nine sampled single-child resources, inflating
only the zlib stream leaves the vertex pointer at the end of the inflated
block. Read `6 * vertex_count` raw bytes starting at the next 16-byte
boundary at or after the compressed stream's end. For example, at
`levels.rpf@0x9b25a9010` the stream inflates to `0x20000` bytes and the
2,374 raw vertices begin at archive offset `0x9b25bb060`. All nine passed
the same leaf geometry checks after appending the raw bytes.

## 13.2 Surface ids

(Confirmed, 2026-11-22.) The surface id is

```
id = 18 + elf(name) % 32749
```

where `name` is the material name **as written** (upper case, underscores)
in the material XML of `common.rpf`, and `elf` is the classic Unix ELF /
PJW string hash:

```
h = 0
for each byte c:  h = (h << 4) + c;  g = h & 0xF0000000;
                  if g != 0: h ^= g >> 24;  h &= ~g
```

32749 is the largest prime below 2^15 (32749 + 18 = 32767), so ids lie in
18–32766 and bit 15 is never set; 0–17 look reserved, and only 0 occurs in
map collision. Example: `ROAD_TARMAC` → `elf` = `0x04D164F3` → 7199 → id
**`0x1C31`**.

| Evidence (389 distinct ids in map collision) | Count |
|---|---|
| readable `<name>` entries in the material XML | 413, all with distinct ids |
| of those, ids that occur in map collision | **212** (random ids: mean 5.1, max 12 over 200 runs) |
| `DEFORM_*` names from the deformable-terrain XML (`<PhysicsMaterials><Item>`) hitting otherwise unmatched ids | **9 of 11** (chance about 0.06) |
| total resolved | **221 of 389 ids, 64 % of collision triangles** |

The nine deformable materials are `DEFORM_DIRT_COMPACT`, `_DIRT_LOOSE`,
`_QUICKSAND`, `_SEAWEED`, `_PLASTIC_MULCH`, `_PERIPHYTON_SOFT`, `_CLAY`,
`_ALGAE` and `_RUBBISH`. Unmatched names are mostly ped and animal
body-part materials (`SPINE…`, `THIGH…`) that map collision does not use.
The remaining ids are most likely materials from the unreadable parts of
the XML: its first block (file offset `0x20f000`–`0x210a62`) is neither
zlib nor raw deflate at any start offset, and the block at `0x212190`
stops inflating after 672 input bytes (5,239 bytes out, which still hold
names).

**History.** Before ELF was tried, these were ruled out: plain index; 15/16-bit folds of joaat, FNV-1/1a
(32/64), CRC32/CRC32C, CRC-16 variants, Adler-32, Murmur2/3, xxHash32 over
many name spellings and prefixes; GTA V's packed {material index,
procedural id}. Part of the material list is also missing (holes in the
dump). No meta resource holds a material enum.

Shader and parameter names turned out to be plain joaat
([§12](12-materials.md)), but the surface ids are not a 15-bit
shift or fold of it (2026-11-21: 402 material names, as given, lower and
upper case, every shift 0–17, xor-fold, three moduli; also seeded joaat
with `0xC3CCF6AD` and five other seeds: best score 7 of 402, chance 3).
A grid of 19 hash functions × 20 spellings × 25 reductions to 15 bits also
stayed at chance; only adding an unknown constant to the reduction showed
the ELF `% 32749` + 18 outlier.

## 13.3 Inferred surface categories

Instead of names, each id gets an **inferred category** and confidence from
evidence over the whole map:

| Cue | What it measures |
|---|---|
| geometry | up-facing / vertical share, mean height, height vs water line |
| road overlap | share of up-facing area on navmesh road polygons |
| path-link density | road-graph links over the id, relative to its area share |
| plants on it | grass-batch instances standing on it, by plant family |
| props on it | reflectors, bollards, barriers, fences, rocks, trees on it |
| railway | surfaces under named railway entities (gravel / ballast ids, below) |

Categories: road surface, dirt track / unpaved road, hard surface around
roads, paved ground, grass / vegetated soil, forest floor, wetland, rocky
ground, walls, shore, seabed, gravel / ballast, and a generic default
surface (one id covers ~39 % of all area). **98.3 % of collision area is
categorised (22.7 % high, 68.1 % medium, 7.4 % low confidence), 243 ids**.
These are categories, not material names.

**Railway areas** (inferred): sampled under 3,224 named railway entities
(sleepers, signal boxes, rail wheels and stacks, crossing signs,
`*_rail_track*` pieces), two ids are strongly over-represented. `2287` is
19–29 % of the samples against 0.4 % of collision area, but it is also in
12,158 bounds files all over the map, so it is gravel / ballast in general
(dirt tracks, verges, railway beds), not railway-only. `4180` is 9–21 %
against 0.13 % of area, in 5,535 clustered bounds files, probably a gravel
variant (verges, yards). The rest falls on the generic default id `317b`
and grass ids; the track geometry itself mostly has no up-facing collision
under the sample points. Neither id resolves with the formula of
[§13.2](#132-surface-ids) from the readable material XML, so their names are
in the part that is not read, and the labels stay inferred.

## 13.4 Fragment physics

In sampled `.#ft` fragments, the resource root points to two arrays
separate from the [drawable and skeleton](07-models.md#75-skeleton-fragments-confirmed).
All offsets below are relative to the named record; integers are little-endian.

| Root offset | Field |
|---|---|
| `+0x20` / `+0x34` | Physics-group pointer / u16 group count (stride `0x70`) |
| `+0x28` / `+0x36` | Collider pointer / u16 collider count (stride `0x50`) |

A group stores u16 first-child, parent and direct-child count at `+0x14`,
`+0x16` and `+0x18`; `0xFFFF` denotes a missing index. The u16 values at
`+0x1A` and `+0x1C` are the start and exclusive end of its collider range.
The end is repeated at `+0x1E`.

| Collider offset | Field |
|---|---|
| `+0x00` | Unit quaternion (four f32s) |
| `+0x10` | Bone-local collider translation (three f32s, on the tested rigs) |
| `+0x20` | Three f32 inertia components, then f32 mass at `+0x2C` |
| `+0x30` | Pointer to a bounds record (type at `+0x10`, max/min at `+0x20`/`+0x30`) |
| `+0x48` / `+0x4A` | u16 owning group index / u16 collider index |

Each inertia component equals mass times the corresponding unit-mass
component stored at bound `+0x60`. On all sampled type-3 bounds, these
three stored components and the f32 at bound `+0x6C` match the analytic
inertia per unit mass and volume of a box with the stored dimensions.
Group links, collider ranges and the mass/inertia identity held for
934 groups and 2,329 colliders in 228 sampled fragments. Another 25
candidates did not match this count layout and were excluded.

**Skeleton link (two ambient human rigs).** Each group stores two u32 bone
hashes at `+0x08`/`+0x0C` and matching u16 skeleton indices at `+0x10`/`+0x12`.
The two slots reference `SKEL_*` and `BM_*` bones, respectively; the second
can be `(0, 0xFFFF)`. In `packs.rpf@0x454db010` (`ambient_malebody`) and
`packs.rpf@0x45404010` (`ambient_femalebody`), all 80 nonempty links across
42 groups match the u32 hashes in the indexed skeleton rows. Each rig has
28 colliders. In the male rig, group 0 at body `0x24008` links `SKEL_Pelvis`
(index 1) and `BM_pelvis` (index 77) to collider 0 at `0x24950`, whose bound
is at `0x26c50`.

### Collider bind transforms

On the sampled rigs, follow the fragment's root `+0x10` pointer and then
`+0x20` of that record to a type-6 composite. Its child pointers match the
bounds pointers in the collider table in the same order. Its transform
arrays at `+0x78` and `+0x80` agree in their XYZ lanes.

Use the **first** bone index of the owning group. Chain the bone's local
transforms through the skeleton to get its model-space rotation `R_b` and
position `t_b`. With collider rotation `R_c` and translation `t_c`, the
model-space placement is `R_b * (R_c * p + t_c) + t_b` for a point `p` in
the collider. In the stored 0x40-byte matrix, the XYZ lanes of the first
three rows are the transformed basis vectors and the fourth row is the
translation. Do not use the fourth lane of each row for this comparison.

This reproduces 472 of 472 child transforms in the 24 fragments with
`limits` blocks: translation error is below 0.003 mm and every rotation
entry differs by less than `3e-6`. Using the second bone when present,
with the same `4e-6` tolerance, matches only 359; reversing the composition
matches none. In `packs.rpf@0x454db010`, the composite is at `0x262b0` and
the first matrix is at `0x25210`. This validates bind-pose placement, not
how the runtime updates the body.

### Vehicle fragment bounds and wheel discs

(2026-11-23.) A vehicle fragment's physics is an **embedded type-6
composite** inside the fragment's system page (not at its root), with the
same header as a standalone composite: `+0x70` child pointers, `+0x78`
child transforms (64 bytes each), `+0x98` u16 count twice (high: the
children and transforms resolve). Its children use more bound types than
level collision: 0, 1, 2, 3, 4, 8, 9, 10, 13 and 14 were seen. Type 13 has a
symmetric bbox and reads as a box, type 14 is cylinder-like and type 4 looks
like convex geometry (all inferred); only type 8 is decoded.

**Type 8: disc** on the local X axis (high): `+0x14` f32 radius, `+0x2C` f32
margin = half the width, bbox max at `+0x20` = (margin, radius, radius). The
translation of the child's transform (`+48` of its 4 × 4) is the centre.
Each wheel has one disc at its wheel bone's pivot in Y and Z (up to a few cm
off in X) with the **tyre's outer radius and half width**; a second, thinner
disc per wheel is the brake disc (medium-high: a brake disc model scaled to
it sits behind the rim spokes on two cars): radius 0.142 front / 0.151 rear
on a supercar, 0.146 on a small hatchback, 0.157 on a sports car; half width
about a tenth of the radius; centre 1–2 cm inboard of the wheel disc. Over 243 wheeled `_hi` vehicles, 228 had a disc on every wheel
(matched on the pivot); helicopters have none. Examples (radius / half
width): small hatchback 0.325 / 0.0975; sports car 0.35 / 0.1325 front,
0.1625 rear; golf cart 0.24 / 0.1075; BMX 0.268 / 0.02; dirt bike 0.324 /
0.045 front, 0.318 / 0.055 rear; armoured truck 0.61 / 0.174. This is the
stored wheel size ([§11](11-vehicle-paint.md#vehicle-tables)).

### Physics-group pairs

A non-null fragment root `+0x58` can point to two u32 count words (equal
in all sampled tables) followed by an array pointer at `+0x08`. Each array
entry is a pair of u16 physics-group indices, not bone or collider indices.
Across 19 nonempty tables (1,535 pairs), no pair repeats, joins a group to
itself, or joins a direct parent to its child. Resolve indices to each
group's first bone hash: both full ambient human rigs have the same 127
pairs, and every sampled nonempty variant table matches that list filtered
to its remaining groups. For the male rig the header is at `0x28400` and
the array at `0x25ca0`. The data does not establish how the runtime uses
these pairs.

### Joint limits

The fragment's `+0x18` points to its main drawable. At drawable `+0x90`,
a pointer can lead to a block with the inline string `limits` at `+0x58`.
Check the tag rather than assuming every non-null pointer is a limits
block. The block stores two u32 count words at `+0x18`/`+0x1C` (equal in
all samples) and an array pointer at `+0x20`.

| Limit row offset (stride `0x90`) | Stored value |
|---|---|
| `+0x00` | Unit quaternion (`f32[4]`); exact reference frame unknown |
| `+0x10` / `+0x14` | Ordered f32 angular endpoints, in radians |
| `+0x20`, `+0x2C`, `+0x38`, `+0x44` | Four optional f32 angular values, consistent with swing limits; the axis and evaluation rule are not verified |
| `+0x78` / `+0x7C` | Two u32 mode words; do not assume a single encoding |
| `+0x80` | u32 bone hash; join to the group's first hash at `+0x08` |

Across 24 sampled fragments with the named block (330 rows, including
human rigs and two fish), each non-root physics group has one row and
no root group has one. All quaternions are unit length within `1e-5`, and
all ordered endpoints are in `[-pi, pi]` within float precision. The
56 rows with mode words `(0, 0x200)` have no additional angles; the 262
rows with `(4, 4)` all have four nonzero values. Other observed pairs include
`(4, 2)` and `(4, 0x104)`. The latter has unequal first and third angles
in the alien rig's right-hand row, so do not assume mirrored values.

For example, `ambient_malebody` has its limits block at body `0x270d0`
and its `SKEL_L_Calf` row at `0x1eea0`: the endpoints are about
`-140°` and `+10°`. The corresponding `ambient_femalebody` row
(body `0x1e5e0`) has `-140°` and `+6°`. The knee and forearm rows in both
rigs have mode words `(0, 0x200)`, identity quaternions and no additional
angles. This supports a hinge-like form but does not identify the hinge
axis or the runtime limit solver.

## Open questions

- The rest of the material list (the XML blocks that do not inflate).
- Material record flag bytes and the geometry `+0x138` target.
- Joint-limit reference frames, swing evaluation and runtime constraint composition.
- The runtime use of the physics-group pairs.
- A railway-only surface (none found).
- The fields of vehicle composite child types 0–4, 9, 10, 13 and 14.


---

← [12. Material classes and texture slots](12-materials.md) · [Index](README.md) · [14. Navigation meshes](14-navmesh.md) →
