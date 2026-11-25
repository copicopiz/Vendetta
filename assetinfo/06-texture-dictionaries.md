# 3. Texture dictionaries

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

Layout matches GTA V's `pgDictionary<grcTexture>` (confirmed).

| Off | Meaning | Confidence |
|-----|---------|-----------|
| 0x20 | ptr → u32 name hashes | confirmed |
| 0x28 | u16 count, u16 capacity | confirmed |
| 0x30 | ptr → texture pointer array | confirmed |
| 0x38 | u16 count, u16 capacity (equal to 0x28) | confirmed |

**Texture struct** (0x70 bytes; also used for textures embedded in models):

| Off | Meaning | Confidence |
|-----|---------|-----------|
| 0x08 | u32 size field (usually bytes / 16; inconsistent, not used) | inferred |
| 0x0C | u16 bytes per element (8 = BC1/BC4, 16 = other BC) | confirmed |
| 0x10 | flags (0x980 typical) | unknown |
| 0x18 | data pointer (`0x6…`) | confirmed |
| 0x30 | ptr → name block (C string at +8) | confirmed |
| 0x38 / 0x3A | u16 width / height | confirmed |
| 0x3C | depth / array size | inferred |
| 0x3F | u8 **DXGI format** (71 BC1, 74 BC2, 77 BC3, 80 BC4, 83 BC5, 95 BC6H, 98/99 BC7, 28 RGBA8, 87 BGRA8 …) | confirmed |
| 0x42 | mip count | confirmed |

## Mip storage versus runtime upload (2026-11-24)

The mip count describes subresources; it does not prescribe a streaming
priority. The tiling and mip-tail rules for dictionary/model textures are
in [§4](04-xbox-tiling.md); bulk `ttpf` stores use the separate GFX10 layout
in [§5](05-ttpf.md).

**Victory implementation cross-check:** the title writes tiled texture
memory. On first use in a host list, `victory-gpu` records a copy for every
texture subresource footprint; when the title-memory copy changes, its
detiler visits every mip and array slice
([texture upload](../../crates/victory-gpu/src/texture.rs)). The bindless
path also checks heap-indexed texture resources and uploads changed contents
before the submission ([bindless uploads](../../crates/victory-gpu/src/bindless.rs)).
The sampler adapter forwards the title's MinLOD/MaxLOD/bias values
([sampler](../../crates/victory-gpu/src/sampler.rs)); that affects sampling,
not the file reads, detiling, or upload budget. This code has no
coarse-mip-first residency policy, so “prioritize mipmaps” needs a measured
streaming/upload change rather than just a sampler LOD override.

## Open questions

- Flags at 0x10; array and cube handling (only slice 0 is exported).


---

← [2. Resources and pages](02-resources.md) · [Index](README.md) · [4. Xbox Series texture tiling (dictionaries)](04-xbox-tiling.md) →
