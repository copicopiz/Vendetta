# 14. Navigation meshes

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

`navmeshes.rpf`: one **128 × 128 m tile** per resource (GTA V `CNavMesh`,
`.ynv`).

| Off | Field | Confidence |
|---|---|---|
| +0x14 | u32 version 0x00010015 (used to detect tiles) | confirmed |
| +0x20 | 4 × 4 transform (identity) | confirmed |
| +0x60 | vec3 AABB size (128, 128, tile height) | confirmed |
| +0x70 / +0x80 / +0x88 / +0x98 | lists: vertices (u16 × 3) / indices (u16) / edges (12 B) / polygons (44 B) | confirmed |
| +0xA0 | sector tree: vec3 min at +0, max at +0x10 = tile bbox | confirmed |
| +0xC0 | u32 vertex count, polygon count, area id, total bytes | confirmed |

A list is a header (+0x08 u32 item count, +0x10 ptr parts, +0x18 ptr part
start indices, +0x20 u32 part count) over 16 KiB pages; each part is {ptr,
u32 count, u32 pad}. **Vertex = tile min + u16 / 65535 × AABB size.**
Polygon: +0x00 u32 area id, **+0x04 u32 = first index << 4 | index count**
(convex fan), +0x0C..+0x18 cell AABB as u16 pairs, **+0x18 u32 flags**.

| Flag bit | Meaning | Confidence |
|---|---|---|
| **0x2000** | road surface: continuous carriageway bands | confirmed |
| 0x100 (with 0x2000 or 0x10000) | squares ~10 m apart along roads and track | unknown |
| 0x10000 | narrow continuous band (rail track or path?) | inferred |
| 0x1000 | most open ground | inferred |
| 0x0200_0000 | blob-shaped areas (vegetation / cover?) | inferred |
| 0x0800_0000 | strips beside roads (shoulder?) | inferred |

Also in `navmeshes.rpf`: 9,603 meta resources with struct `0bcb00e4` (160
B, a triangle-adjacency point graph in a 16 × 16 bucket grid; vertex-key
packing unsolved), and resources starting `CHVT` at +0x60 (not decoded).

## Open questions

- Remaining polygon flag bits; edge records; the `0bcb00e4` graph; `CHVT`.


---

← [13. Collision and surface categories](13-collision.md) · [Index](README.md) · [15. Lights](15-lights.md) →
