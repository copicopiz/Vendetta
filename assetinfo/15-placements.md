# 9. Placements, LOD links, archetypes and batches

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

## 9.1 Map file and entity

**Map file** root struct `d3593fa6` (0x460 B): name hash +0x08, parent
hash +0x68, flags +0x80, four vec3 extents +0x90…+0xC0, `entities` array
+0xD0 (GTA V `CMapData`).

**Entity** struct `ce501483` (0x90 B, GTA V `CEntityDef`):

| Off | Field | Confidence |
|-----|-------|-----------|
| 0x08 | u64 per-archetype hash (function unknown) | confirmed field, unknown hash |
| 0x10 | rotation quaternion x, y, z, w: **the inverse** of the entity rotation | confirmed |
| 0x20 | u64 GUID | inferred |
| 0x28 | `extensions` array (lights, particles …) | confirmed |
| 0x38 | position vec3, world, Z-up, metres | confirmed |
| 0x44 | u32 **archetype = joaat(model name)** | confirmed |
| 0x48 | u32 flags | confirmed field |
| 0x54 / 0x58 | f32 scale XY / scale Z | confirmed |
| 0x4C, 0x5C–0x82 | other fields, kept by hash | unknown |

## 9.2 LOD hierarchy (confirmed)

| Field hash | Meaning | Values |
|---|---|---|
| `6c8f1715` | lodLevel | 0 HD, 1 LOD, 2/3/4 SLOD1–3, 5 ORPHAN_HD, 6 SLOD4 |
| `d89231bd` | parentIndex | LOD slot in the parent map file ([parents](#resolving-lod-parents)) |
| `e74c3c23` | parentInstance | −1, or an instance of a prop batch |
| `a687ac89` | numChildren | 0 on HD/orphans; 1–16 on LODs |
| `bf74dfa7` | lodDist | −1 = use the archetype's `lodDist` |
| `ca974bcd` | childLodDist | 0 on HD |
| `2691f019` | priorityLevel (joaat checked) | 0 on every entity |
| `84799847` | unknown; **not** priorityLevel as first labelled | −1 to 10; −1 on orphans and SLOD4 (below) |
| `0dc5f270`, `540ca436` | AO multipliers | usually 255 (inferred) |

*Correction (2026-11-22; measured, full census of 30,789 map files and
2,250,722 entities):* `priorityLevel` is `2691f019`, not `84799847`, and it
is 0 on every entity at every LOD level. Neither field is 0–3 as first
given. `84799847` runs from −1 to 10 (−1: 982,890; 0: 611,731; 1: 343,406;
2: 177,316; 3: 84,839; 4: 33,923; 5: 12,547; 6: 3,237; 7: 666; 8: 123;
9: 41; 10: 3). It is −1 on every ORPHAN_HD (970,787) and SLOD4 (352)
entity; HD uses 0–10, mostly 0–3 (6,371 at −1); LOD 0–3, SLOD1 and SLOD2
0–1, SLOD3 only 0, each with a few −1. So it narrows with LOD level and is
−1 where an entity has no LOD parent; its meaning is unknown.

The parent index points into the entities of the map file whose name hash
equals this file's parent hash. Checked on 400 child files: every link goes
exactly one level up (HD → LOD, LOD → SLOD1), none is out of range, the
median child–parent distance is 0.47 m, 80 % of named pairs share a name
prefix (`x` → `x_lod` → `x_slod`). Of 2,089 files with a parent hash, 1,304
resolve.

### Coverage gaps near the camera (2026-11-23)

Found while streaming the map in a 3D viewer:

| Finding | Confidence |
|---|---|
| **Unlinked SLODs cover their area alone.** Many SLOD placements (SLOD4 ground tiles especially) have no LOD parent or child link. With plain distance bands they vanish inside their own band and nothing finer replaces them, leaving holes next to the camera (downtown: about 33 such placements near the camera at low detail, 71 at high) | high |
| **Placements with empty archetypes.** Some placements resolve to no model name, with render flags `0x41` (one at lod 0 with `lodDist` 171, one at lod 1 with `lodDist` 450, both with a 1 m placeholder radius); they look like the real ground pieces for those spots, so only coarse SLOD ground covers them | medium (a few spots, not counted map-wide) |
| **Large buildings without a main HD placement.** Some downtown towers have only detail, canopy or roof-detail parts in the placements, so only the SLOD box shows close up | medium (one tower checked in depth) |
| **SLOD textures are whole-area atlases.** A SLOD base texture (256–512 px) is a mosaic of tiny facade and ground patches for a whole area; at 64 px it averages to near-white. Its texel density is far below HD, so a streamer should size it by distance to the placement's edge, not its centre | high |
| **SLOD materials with no texture.** Several SLOD4 materials have no base texture and base-colour alpha 0 (placeholders) | high |
| **Building glass carries only a dirt layer.** Glazing materials (glass / window class) hold only a light grime texture; reflections and interiors are added at runtime, so drawn as a see-through texture they read as grey or missing buildings | high |

Open: where the missing ground and building archetypes live (another map
file, an interior or streaming set), and how the game fills these areas
near the player.

### Archetype LOD distances (confirmed)

The archetype definition (`82d6fc83`, [§9.4](#94-archetype-definitions-and-unnamed-drawables))
carries the same names: `lodDist` +0x38, `bsCentre` +0x58, `bsRadius`
+0x64, `hdTextureDist` +0x70. 1,060,711 of the 1,158,518 HD entities have
`lodDist` −1. Where an entity sets it, it equals the archetype value for
52 % of HD and almost all LOD/SLOD entities, and overrides it for the rest.
1,492 of 278,174 archetype hashes have definitions with different `lodDist`
values.

### Resolving LOD parents

A map file's **LOD slots** are its entities followed by its prop batches
(`ba049770`). There are two parent forms.

**parentInstance = −1**: parentIndex is an entity of the parent map file.
Only 2,089 files name that file with a parent hash (the parent's name hash;
the 64-bit `e5cdb664` equals the parent's `8bfb6095`). For the others, the
parent file can be chosen per child file, since all children of a file
share it (inferred rule):

1. Candidates are files in the same region (map-data field `6c0f9c8c`,
   shared by a child file and its LOD files) whose entity at each child's
   index is one level up and has children. SLOD1 → SLOD2 links leave the
   region, so there any file near the children is a candidate.
2. Rank by the number of children inside the parent's bounding sphere,
   then by children per parent not above `numChildren`, then the nearest
   following file in the archive.

On the files that do have a parent hash, this picks the same file in 95 %
of cases; the misses are state variants of one LOD at the same place
(identical candidates).

**parentInstance ≥ 0** (about 1.05 M HD entities): the parent is instance
`parentInstance` of prop batch `parentIndex − <entity count>` in another
map file, a `<name>_batchlod` model (confirmed). No decoded field names
that file, but the instance sits exactly on the entity (median 0.0 m), so it
is found by position. About 3 % of prop batches do not decode (garbage
instance positions); for those, match by slot and by the name `<entity
name>_batchlod`, preferring the same region. Where both methods work they
pick the same batch in 99.9 % (one same-region candidate) and 99.0 %
(several) of cases.

Results on `levels.rpf`: batch parents 1,042,067 of 1,052,215 (99.0 %);
entity parents 214,517 of 214,705 (177,809 high, 30,444 medium and 6,264
low confidence).

## 9.3 Batches (instanced grass and props, confirmed)

Subclasses of the entity struct, listed in the map data's `f14a15be`
(`rage__fwInstancedMapData`; +0x140: arrays at +0x08 / +0x18). The entity prefix gives the archetype
and a batch centre; instance arrays sit in the graphics page.

| Struct | GTA V equivalent | Fields |
|---|---|---|
| `ba049770` prop batch (368 B) | `PropInstanceList` (hash = `rage__fwPropInstanceListDef`; instances `rage__fwPropInstanceListDef__InstanceData`) | +0x60 LOD distance, +0x80 u16 array, **+0x90 instances `ea6b3a96` (64 B)**: 3 × vec4 rows of a 3×4 matrix (rotation × scale in xyz, world translation in w), +0x30 f32, **+0x34 tint RGBA**, +0x38 f32, +0x3C u16, +0x3E u8 × 2; +0xA0 bbox |
| `7c475b5d` grass batch (416 B) | `GrassInstanceList` (hash = `rage__fwGrassInstanceListDef`; instances `rage__fwGrassInstanceListDef__InstanceData`) | +0x80 bbox, +0xA0 vec3 scale range, +0xB8 LOD distance, **+0x178 instances `ed86f522` (16 B)**: u16 × 3 position quantised in the bbox, u8 normal x / y, u8 × 3 colour, u8 scale, u8 AO, u8, u16 |

906,922 batches: 24,880,278 prop and 219,484,128 grass instances; 99.76 %
of batches have a model name.

**Minigame pack** (`shared_minigame000`, 2026-11-23; counts high): 3,909
batches, 3,317 prop and 592 grass, with 437,990 grass instances, nearly all
in the hunting game's valley (largest kinds `veg_grass_short_cut_ps_01`
117,161, `veg_grass_sml_01` 21,405, `veg_grass_lrg_01` 19,818) and on the
puzzle game's island; 2,298 leaf and litter instances in the beat-'em-up
area. Batch scale ranges are e.g. 0.75–1.25. No per-instance rotation is
stored. The three grass colour bytes are **not a plain tint** (low): used
as a multiply they turn everything dark olive (sample 83, 78, 57).

## 9.4 Archetype definitions and unnamed drawables

Archetype struct `82d6fc83` (184 B, GTA V `CBaseArchetypeDef`):

| Off | Field | Confidence |
|-----|-------|-----------|
| +0x40 / +0x4C | bbox min / max | confirmed |
| +0x74 | name (joaat) | confirmed |
| +0x78 | texture dictionary | confirmed |
| +0x7C | drawable dictionary (0 where checked) | inferred |
| +0x80 | physics dictionary | confirmed |
| +0x88 | asset name | confirmed |
| +0x90 | extension pointer array | confirmed |

92,925 archetypes have no named drawable because the name string was not
recovered. The drawable's bounds equal the archetype bbox, so they can be
linked by bbox (within 0.03 m). **The archive is stored in name order**,
which breaks ties (e.g. between 64 m ground tiles that share a bbox).

Two more rules (2026-11-20):

- **Embedded drawable name** (tried first, confirmed): about 8.8 K drawables
  that the archive scan left unnamed still carry their own name in the
  resource's name block. When joaat(name) is an unresolved archetype and
  only one drawable carries it, that is the link. These 8,616 links agree
  with all 14,777 earlier links they overlap. Their bounds often differ
  from the archetype bbox, which is why the bbox rule missed them.
- **Name-order gap** (fallback, inferred): an archetype whose name is
  guessed (joaat of sibling-name stems plus common suffixes) has its
  drawable between the named entries just before and after that name.
  With exactly one candidate in the gap it is linked; with 2–6, the one
  whose bounds clearly fit best within 0.5 m (3,645 links).

Result: **15,412** of 92,925 linked (8,616 by embedded name, 3,645 by name
order, 2,565 by unique bbox, 586 by bbox plus locality); 25,277 bbox
matches remain ambiguous.

`35abc248` (32 B, one per file, 6,304 files) is a named entity group
(prefab) in group space; no map data places these groups.

## 9.5 Render hints and indoor dressing

**Proxies and placeholders** (naming rules, checked by exporting examples;
medium): `vfx_proxy` = `*vfx*proxy*`, `vfx_dummy*`; `light_proxy` =
`*_lit_proxy`, zero-size geometry; `interior_proxy` = `*_pxy`,
`*shell_proxy`, `int_*_proxy` (stand-ins for interior shells);
`shadow_proxy` = `*_shadow`; placeholder = no model resolved. Shop items
`gmt_*_proxy_*` and cables `*_occl_*` are real geometry. Models with bounds
under 1 cm in two axes are degenerate. No flag bit has been looked for.

**Indoor-only entities: no marker found** (measured on a 1/6 sample;
negative). Furnished rooms (ceiling and wall lights, chairs, tables,
`int_*`) are ordinary world entities mixed with outdoor props; only 3.9 %
lie inside an interior instance. Of 45 K indoor-named and 68 K
outdoor-named HD entities: flag bits 3 and 18 are indoor-only but rare
(1–4 %); ORPHAN_HD is 48 % against 9 %; map-file fields do not differ and
files are not grouped by indoor share; "covered" (collision more than 3 m
above) is 43 % against 20 %. A converter has to use a name heuristic,
optionally with the covered test.

## Open questions

- The name and meaning of `84799847`.
- Whether any field marks indoor dressing or proxies.
- The 64-bit per-archetype hash at entity +0x08.
- Where prefab groups (`35abc248`) are placed.
- 785 map files whose parent hash does not resolve.


---

← [8. Meta resources (type 0x7a)](08-meta.md) · [Index](README.md) · [10. Interiors (MLO)](10-interiors.md) →
