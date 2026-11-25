# 2. Resources and pages

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

## 2.1 Pages (confirmed)

- Pointers are 64-bit. `0x5xxxxxxx` points into the **system page** (CPU
  structures), `0x6xxxxxxx` into the **graphics page** (GPU data). The low
  28 bits are a linear offset into the concatenated page bytes.
- The u64 at +0x08 is the page-map pointer.
- An entry's zlib streams are the system-page parts followed by the
  graphics-page parts. The system page ends at the first stream that takes
  the total past the page-map offset.
- **Extra system streams.** Some resources have one more small system
  stream that only other system pointers reach. Keep appending the next
  stream to the system page while at least `max(2, size / 64 KiB)`
  **4-aligned** `0x5…` values in the system bytes point into it
  (confirmed, 2026-11-20). Values that are not 4-aligned must not count:
  hash-like values would pull a graphics stream in. Requiring 8-alignment
  is too strict, because some meta resources reach their continuation
  only through 4-aligned pointers. Without this rule the stream lands in the graphics page, every
  graphics offset shifts and the model has no geometry.
- `PRD0` (on disk `0DRP`) at +0x10, followed by a type byte at +0x14 (e.g.
  `0x7a`, [§8](08-meta.md)), marks self-describing resources.
- Small graphics pages of meta resources are stored **uncompressed right
  after the zlib stream** (next 16-byte boundary), invisible to a zlib
  carver ([§8](08-meta.md)).
- **Graphics page as its own stream** (2026-11-23; layout high, trigger
  medium): when a map resource's last zlib stream inflates to a whole
  multiple of 64 KiB (64 K, 128 K and 448 K seen; 105 files in the levels
  archive, 2 in the minigame pack), the graphics page is not stored
  uncompressed after it. It is a separate zlib stream a few KiB to about 20
  KiB further on, 16-byte aligned, inflating to exactly the graphics block
  extent or that rounded up to 64 KiB (e.g. 309,056 bytes needed, 327,680
  stored). Files whose last chunk is partial keep the uncompressed page.
  What fills the gap before the stream is unknown. Reading this layout
  brought prop-batch instances with garbage transforms from 130,066 to 0
  of 24.9 M in the levels archive and from 5,728 to 0 of 57,545 in the
  minigame pack (together with the pool-reference rule in
  [§8](08-meta.md)).

## 2.2 Resource types

| Suffix in name string | Type | GTA V equivalent |
|---|---|---|
| `#dr` | drawable | `.ydr` |
| `#ft` | fragment (vehicles, breakables) | `.yft` |
| `#dd` | drawable dictionary | `.ydd` |
| `#td` | texture dictionary | `.ytd` |
| none, `PRD0` type 0x7a | meta resource (map data, archetypes, lights, paths …) | `.ymap`, `.ytyp`, `.ynd`, … |
| none (unnamed `rageobj`) | physics bounds, navmesh tiles | `.ybn`, `.ynv` |

## 2.3 Names (confirmed)

The original asset name survives as a lowercase string `name.#xx` inside
the system page. Bytes in front of it can be printable, so scan backwards
only over `[a-z0-9_-]`. Texture dictionaries carry no `.#td` string; SixNG
names them after their first texture.

## Open questions

- About 17 K drawables and most physics bounds have no name string. A part
  of them are linked to their archetype by bbox ([§9.4](09-placements.md)).


---

← [1. RPF9 archives](01-rpf9.md) · [Index](README.md) · [3. Texture dictionaries](03-texture-dictionaries.md) →
