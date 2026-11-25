# 1. RPF9 container

## Header

The fields below occupy the first `0x40` bytes.

| Off | Size | Meaning | Confidence |
|---|---|---|---|
| 0x00 | 4 | magic `39 46 50 52` (`"9FPR"` on disk; `RPF9` as a little-endian u32) | OBSERVED |
| 0x04 | 4 | `toc_end` — absolute end of the table of contents; payload starts after it | OBSERVED |
| 0x08 | 4 | varies by archive; not `toc_end`, not a count | UNKNOWN |
| 0x0C | 4 | length field; see [Trailer](#trailer) for the observed correspondence | OBSERVED |
| 0x10 | 8 | 64-bit per-archive value; unique across archives | OBSERVED |
| 0x18 | 4 | `0x47A00` or `0x57A00` | UNKNOWN |
| 0x1C | 4 | `0x150` or `0x48D0` | UNKNOWN |
| 0x20 | 2 | observed values 3 through 8 across the two samples | UNKNOWN |
| 0x22 | 2 | `1` or `2` | UNKNOWN |
| 0x24 | 4 | per-archive file count (2,078 for the largest archive); **not** the carved stream count | OBSERVED |
| 0x28 | 4 | four values in the original 22-archive sample: `0xC0A92754`, `0xD5C19F6B`, `0xD98004D5`, `0xDA72573E`; two additional values below | OBSERVED |
| 0x2C | 4 | index variant — exactly **2 distinct values** across 22 archives: `0x00090F01`, `0x00090F02` | OBSERVED |
| 0x30 | 4 | `0x10` or `0x4790` | UNKNOWN |

### `data.rpf` header observations

In one Xbox `data.rpf` sample, reading little-endian fields at all 42
4 KiB-aligned `9FPR` headers (one outer header and 41 nested headers) gives:

- `0x20` takes values `4`, `5`, `6`, `7`, and `8`; `0x22` is `2` in 16
  headers and `1` in 26.
- The nested header at absolute offset `0x1228E000` has `0x20 = 8`,
  `0x18 = 0x57A00`, `0x1C = 0x48D0`, and `0x30 = 0x4790`. The other 41
  headers have `0x47A00`, `0x150`, and `0x10` at the latter three offsets.
- `0x28` contains all four values from the original 22-archive sample plus
  `0x5A5B4C6F` (13 headers) and `0xABE9B070` (one header). These are field
  values, not confirmed key identifiers.
- The bytes in `[0x40, 0x50)` are identical across all 42 headers. Extending
  the comparison to `[0x40, 0x60)` gives 41 matches, with `0x1228E000` as
  the exception. This does not establish the start of the encrypted entries.

### Header variants

Across 22 archives the `0x2C` field splits them into two groups:

- `0x90F01`: `audio`, `navmeshes`, `packs`, `textures`, the top-level `script` archive, and
  all small `*_restofworld` / `*_bvh4` archives.
- `0x90F02`: `levels`, `data`, `anim`, `story000`, `movies`, `common`, `core_bvh4`, and the
  nested PSO archive inside `script`.

The second group shows `round16(10 × file_count)` for the region between `index_end` and
`block_end` exactly (e.g. 2,078 × 10 + 4 = 20,784). The first group does not follow that law.

## Table of contents

Encrypted. Entropy is at the uniform-random level in every archive, and no repeated 16-byte blocks
occur. No file names or directory tree are recoverable from the TOC.

## Payload

- Sampled payload bodies are not encrypted. Most RPF9 bodies are plain zlib
  streams, each starting on a 16-byte boundary. `movies.rpf` is an observed
  exception: its Bink 2 files are stored uncompressed
  ([format notes](../formats/17-other.md#video)).
- Large resources are stored as **chains of 64 KiB blocks**: a block continues into the next
  stream **iff** it decoded to exactly 65,536 bytes and the next stream begins at the 16-aligned
  slot after the previous compressed stream. There is no flag; the full-block size is the only
  marker.
- RAGE resource pages begin at a 4 KiB boundary plus `0x10`.
- Unused tails are zero-filled; empty regions are absent rather than padded.

## Nested archives

RPF9 archives may contain RPF9 archives. One archive of 4.2 MiB has three headers: offset `0x0`
(2 entries), `0x1000` (2 entries) and `0xC4000` (1,012 entries, `0x90F02` variant, with
`index_end < block_end` — the reverse of the usual ordering). Enumeration must therefore scan for
the magic at 4 KiB boundaries rather than assume one header per file.

## Trailer

Do not require the trailer length divided by 16 to be at least the header's
file count. Four nested headers in the same `data.rpf` sample have fewer
16-byte units in the length field (`0x0C`) than entries in the count field
(`0x24`). Offsets are absolute within `data.rpf`:

| Header offset | `0x0C` (bytes) | `0x0C / 16` | `0x24` (entries) |
|---|---|---|---|
| `0x9CA000` | 160 | 10 | 11 |
| `0xA8A000` | 160 | 10 | 12 |
| `0xB07000` | 1,872 | 117 | 147 |
| `0x9E13000` | 336 | 21 | 27 |

The 13 nested headers with `0x28 = 0x5A5B4C6F` each have `0x0C = 0x74A0`.
In each case a region of exactly `0x74A0` bytes starts `0x8000` bytes before
the next aligned `9FPR` header. It begins and ends with a nonzero byte,
has zero padding on both sides, and is followed by `0xB60` zero bytes up
to the next header. These 13 trailer regions are byte-identical, despite
distinct `0x10` values and distinct header-to-trailer-end lengths. For
example, zlib streams at absolute offsets `0x11F4010` and `0x1C44010` in
the first two members decompress to different bytes.

Matching trailers do not imply identical payloads or establish a shared
file list. Trailer record layout and semantics remain unknown.

## Open questions

- Meaning of header fields `0x08`, `0x18`–`0x22`, `0x2C`, `0x30`.
- `0x24` versus carved counts: 2,078 by header read for the largest archive, against 523,552
  carved streams for the same archive. These are different quantities — file count against
  stream count — and do not conflict once named. The implied streams-per-file ratio is untested;
  a full carve of that archive is required before it can be stated.
