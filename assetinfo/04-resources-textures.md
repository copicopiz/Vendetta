# 2. Resources and textures

## Resource page layout

RAGE 9 resource pages begin at a **4 KiB boundary plus `0x10`**. The 16 bytes at the page start
are zeroed in stored form — the resource body follows.

Pointer values inside a resource use paged addressing: `0x5xxxxxxx` and `0x6xxxxxxx` ranges appear
as relocated addresses, and a section's own pointer-sized fields resolve within the resource body.

## Texture descriptor (`.rsc` bodies)

A texture-carrying resource body has a descriptor table. The descriptor for a record sits **28
bytes before** the record's first byte (`rec − 0x28`):

| Field | Meaning | Confidence |
|---|---|---|
| `desc + 0x00` | `A` — allocation bytes divided by `B` | OBSERVED |
| `desc + 0x04` | `B` — bytes per block (BCn) or per pixel (uncompressed) | OBSERVED |
| `desc + 0x10` | GPU virtual address of the data | OBSERVED |
| `rec + 0x00` | u32 pointer, `0x50xxxxxx`; the NUL-terminated name is at `(ptr & 0xFFFFFF) + 8` | OBSERVED |
| `rec + 0x08` | u16 width | OBSERVED |
| `rec + 0x0a` | u16 height | OBSERVED |
| `rec + 0x0c` | u16 depth (1 for 2D) | OBSERVED |
| `rec + 0x0e` | u16, always 1 | OBSERVED |
| `rec + 0x0f` | u8 DXGI format | OBSERVED |
| `rec + 0x12` | u16 mip count with flag `0x8000` (`0x8000 \| n`) | OBSERVED |

### Size law

For a resource body: `file_size = H + Σ(A × B)`, where `H` is the per-file system prefix
(observed from 240 to 192,240 bytes) and the sum runs over all descriptors. Residual is zero on
all 43 multi-record files tested. Two files carry one 64 KiB alignment gap each, both verified
all-zero.

Data offset for a record: `H + (GPU − minGPU)`, masked to 28 bits (GPU addresses reach
`0x62603000`). Mip 0 is stored tight at true dimensions.

**Naming rule:** the `rec + 0x00` pointer resolves a name for every record in the tested set
(3,243 of 3,243), so a name table is derivable without any external index. A raw printable-run
scan over the header instead picks up digest fragments and is not a substitute.

## Texture store descriptors (`.ttpf`)

See `05-ttpf-stores.md`. The descriptor's second u32 carries the format in bits 3..9:
`((u1 >> 3) & 0x7F)` is the standard DXGI format enum. This was verified across 357,115
descriptors with no unknown values, covering BC1, BC2, BC3, BC4, BC5, BC6H, BC7, RGBA8, BGRA8,
RGBA16F/UNORM, R16F, R8 and A8.

## Tile mode

Descriptor bits 24..27 select the tile family. Across 1,722 descriptors in one store the values
divide as mode 0 ×199, mode 1 ×176, mode 2 ×1,147, mode 3 ×200 — four modes: **mode 0 = `SW_64KB_D`, modes 1–3 = `SW_4KB_S`**. This resolves which family to
apply per surface; the exact bit permutation within each family still has to be validated against
a known image.

## Deswizzle

Xbox Series stores tiled surfaces. The family is selected by descriptor bits 24..27: mode 0 is
`SW_64KB_D`; modes 1–3 are `SW_4KB_S`.
Deswizzling requires the tile family, the element size and the surface dimensions; the index is a
Morton-style bit permutation over (x, y) whose exact terms differ per element size. Treating a
swizzled surface as linear produces plausible-looking noise, which is why a correct tile
transform must be validated against a known image rather than eyeballed.

## Open questions

- Which tile family applies to which descriptor (not yet correlated with a header bit).
- Whether 4- and 2-byte formats use a different tiling.
