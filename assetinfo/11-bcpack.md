# 6. BCPack

*Part of the [SixNG format specification](README.md). Confidence: confirmed / inferred / unknown.*

BCPack is the Xbox Series hardware texture codec. It compresses existing
BC blocks, not pixels. Separate three steps when testing a decoder:
BCPack bytes to lane symbols, lane symbols to ordinary BC blocks in
stored order, and detiling those blocks for display. Passing one step
does not verify the next.

## 6.0 Status

The counts and tests below cover one Xbox Series build, not the whole
format across all titles.

| Codec | Entropy decoding | BC block reconstruction |
|---|---|---|
| `0x04` BC1 | Confirmed on sampled chunks with two implementations. | **Decodes** ([BC1 blocks](#bc1-blocks-codec-0x04)): 254 of 256 preset slots fitted (slots 0 and 4 are never referenced, so no block depends on them); 8 selector codebook entries (4 mirror pairs) are uncertain. Mip 0 against mip 1: median 33.5 dB over 35,563 textures. Fitted (inferred); not byte-exact. |
| `0x0C` BC3 | All 35 BC3 chunks decode with the expected symbol counts, the 65,536-block ones as one stream (confirmed). Two implementations also agree on every symbol on 5 samples. | **Decodes** ([§6.2](#62-bc3-blocks-codec-0x0c)): 27–47 dB against neighbouring mips, mostly 31–39. The preset table and selector codebook are fitted (inferred); not byte-exact. |
| `0x10` BC4, `0x14` BC5 | Two implementations agree on every symbol and consume the full input on 2 BC4 and 5 BC5 samples. | **Decodes** ([§6.4](#64-bc4-and-bc5-blocks-codecs-0x10-0x14)): BC4 35–50 dB, BC5 median 43 dB against neighbouring mips. Index-pair tables and presets are fitted (inferred); not byte-exact. |
| `0x18` BC6H | All 268 BC6H chunks in the local scan consumed their exact input and emitted the expected symbol counts per lane. All symbols also matched a second implementation on 5 spot checks. | **Decodes** ([§6.5](#65-bc6h-blocks-codec-0x18)): 87.7 % of blocks within 0.15 stops of plain neighbouring mips, 28.1 dB tonemapped; on textures stored only as BCPack, 71.4 % of blocks match the decoded neighbouring mips after the refits in [§6.7](#67-open-unknowns) (62 % before). History tables are fitted (inferred); not byte-exact. |
| `0x1C` BC7 | All 5,167 BC7 chunks of the data and content-pack stores decode (5,120 in the data stores). All symbols also matched a second implementation on 24 spot checks. | **Decodes** ([§6.3](#63-bc7-blocks-codec-0x1c)): on 1,029 chunks checked against the next mip, 65.5 % of blocks within 3 levels, median 35.4 dB, no invalid blocks. The 256-slot history table is fitted (inferred); not byte-exact. |

For BC1, the block count is the chunk's virtual size divided by 8.
Strip the trailing pad from the stored bytes (`b >> 20` in the chunk
entry, [section 5](05-ttpf.md#chunk-entry-u32-a-u32-b)). The output is in
the texture's stored tiled order; detile it exactly as for a raw or zlib
chunk of the same surface.

The earlier SixNG measurement remains a coverage figure, not a
byte-exact benchmark: all 2,776 BCPack BC1 textures in one terrain
store decoded without errors. With BC1 decoding, mip 0 was readable
for 98.4% of store textures
([coverage](05-ttpf.md#coverage-all-57-stores-352461-textures)). With BC3
and BC7 as well (2026-11-22), mip 0 is outside undecodable BCPack chunks for
99.94 % of the 357,112 textures of the data and content-pack stores (99.41 %
before BC7). 1,872 textures gain a full-size mip 0 from BC7, 17 from BC3.
With BC4, BC5 and BC6H too, **every texture of the 357 K has a decodable
mip 0** by the chunk-table check (99.97 % before BC6H); BC4 adds 40
textures, BC5 91 and BC6H the last 98.

**Every BCPack chunk decodes** (2026-11-23, counted, high): 108,605 chunks
in the 57 stores, 0 failures.

| Codec | All 57 stores | 46 main stores | Unfitted blocks |
|---|---|---|---|
| BC1 | 102,725 | 100,020 | 0 |
| BC3 | 35 | 35 | 0 |
| BC4 | 77 | 77 | 0 |
| BC5 | 333 | 325 | 0 |
| BC6H | 268 | 268 | 0 |
| BC7 | 5,167 | 5,120 | 0 (the last 34 endpoint preset slots, 839,119 blocks in 251 chunks, fitted 2026-11-23) |

The main-store counts match the codec table of [§5](05-ttpf.md); the
content-pack stores add BC1 2,705, BC5 8 and BC7 47 chunks.

**Whole-install accuracy** (2026-11-23; numbers high, reading inferred):
every texture whose mip 0 lies in BCPack chunks (37,226 textures, slice 0,
at least 2 mips), mip 0 box-downsampled against mip 1, RGB PSNR on the
8-bit decode, with every 40th plain texture as a baseline.

| Codec | Plain median | BCPack median | BCPack textures | BCPack under 20 dB |
|---|---|---|---|---|
| BC1 | 37.1 dB | 33.5 dB | 35,563 | 0.8 % |
| BC3 | 50.3 dB | 34.3 dB | 16 | 0 % |
| BC4 | 54.8 dB | 47.0 dB | 40 | 0 % |
| BC5 | 44.4 dB | 44.2 dB | 85 | 0 % |
| BC6H | 29.1 dB | 18.0 dB | 89 | 58 % |
| BC7 | 40.9 dB | 32.8 dB | 1,433 | 6.7 % |

*Later figures (2026-11-23):* BC7 median 37.9 dB with 1.7 % under 20 dB
after the [secondary index reset](#63-bc7-blocks-codec-0x1c); BC6H 22.1 dB
with 36 % under 20 dB after the refits in [§6.7](#67-open-unknowns).
Later still: BC7 colour median 38.1 dB (mean 39.1, 5.0 % under 30 dB, 1.5 %
under 20 dB) with alpha in the preset fit, alpha mean 39.8 → 48.0 dB
([§6.3](#63-bc7-blocks-codec-0x1c)); BC6H median 24.0 dB (mean 24.7, 23.6 %
under 20 dB) after the no-wrap refit.

BC1 has no fitted tables, so its 3.6 dB gap is a population effect (BCPack
mip 0s are larger and more detailed), not decode error. Allowing for that,
BC5 is at plain quality and BC7 about 4–5 dB short. BC6H is weakest, partly
because the measure clips HDR to 8 bits (plain BC6H reaches only 29 dB)
and partly real; how much is each is open. The plain baselines contain
real outliers (9.3 % of plain BC7 under 20 dB, where the mips differ).

**Limits of the mip check and the fair gap** (2026-11-23, high for the
counts):

- The lighting store's mips are **prefiltered, not downsampled**, so mip 0
  against mip 1 is not a valid check there: plain BC6H in that store scores
  14.9 dB, and its BCPack blocks with a plain reference match only 12 %.
  Leaving that store out, the BC6H plain-reference check reads 90.3 % of
  blocks within tolerance and 32.2 dB. A refit that leaves it out of the
  targets is described under the BC6H no-wrap rule in
  [§6.7](#67-open-unknowns).
- Comparing BCPack against plain textures **of the same store and size**
  gives the fair gap: BC1 −2.2 dB (32,459 textures; 1024² −1.0, 512² −3.1,
  128² −4.3), BC7 −2.2 dB (1,161), BC6H −3.9 dB (40; *corrected below*),
  BC5 −0.6 dB (9). BC3
  and BC4 have no matched plain textures. *This revises the reading above
  that BC1's gap is only a population effect:* BC1 with the
  [§6A.2](06a-bcpack-tables.md#6a2-bc1-preset-slots) presets is not exact
  either, and a refit of its preset table is running.
- In the imposter store plain BC7 scores 22.0 dB and BCPack 22.9 dB, so most
  BC7 textures "under 20 dB" reflect the check, not the decode.
No exact preset values are published anywhere found.

## 6.1 What is solved

**Lanes (confirmed for the tested chunks).** A lane is an independent
rANS state machine. The lanes share one interleaved source stream:
12 lanes for BC1/3/4/5 and 16 for BC6H/7. Each lane emits one symbol
per 8 bytes of unpacked data, so a 16-byte BC block uses two symbols
per lane. The BC1 lanes are:

| Lane | Role |
|---|---|
| 0 | Endpoint-history back-reference code. |
| 1 | Constant zero in the sampled chunks; not the three-colour flag. |
| 2-7 | Residuals for the six RGB565 endpoint components: R0 G0 B0 R1 G1 B1. |
| 8-11 | Selector codebook symbols for the top-left, top-right, bottom-left and bottom-right 2x2 quadrants, not four rows. |

**Fragment framing and byte order (confirmed for the tested streams).**
Each lane is a byte-wise rANS state machine.
The first header byte sets the alphabet narrowing, length form, flush flag
and static model number. The short form has `lo + 1` payload bytes; the
long form has `(lo + (hi << 8)) + 257`. This includes state bytes read
by the lane. The feeder services pending requests in lane order, at most
eight bytes per clock. It does not allocate an independent byte range to
each lane or give every requesting lane a byte every clock.

**End checks (confirmed).** Require the expected symbols in every lane,
every non-padding input byte accounted for, and a legal terminal state.
With `L = 2^24`, the allowed final states are `L`, `L >> 8` and `L >> 16`;
requiring `L` alone rejects valid streams.

### Entropy decoder in full

(2026-11-23; confirmed: every lane of all 108,605 chunks ends in a legal
state with these rules, and changing any one rule breaks convergence.)

**Chunk level.** A BCPack chunk has **no stream header**: its stored bytes
(the chunk length minus the pad, [§5](05-ttpf.md#chunk-entry-u32-a-u32-b))
are one byte stream shared by all lanes, and the first byte is already
lane 0's first fragment header. The output size is the chunk's virtual
length; blocks = virtual length / 16 (/ 8 for BC1 and BC4).

| Codec | Lanes | Symbol width per lane, bits | Symbols per lane |
|---|---|---|---|
| BC1 | 12 | 8, 8, 5, 6, 5, 5, 6, 5, 8, 8, 8, 8 | blocks |
| BC3 | 12 | 4, 4, 8, 6, 6, 6, 6, 5, 6, 5, 8, 8 | 2 × blocks |
| BC4 | 12 | 8, 8, 8, 8, then 8 × 6 | blocks (8-byte blocks: block 2j = E, 2j + 1 = O) |
| BC5 | 12 | 4, 4, 8, 8, then 8 × 6 | 2 × blocks |
| BC6H | 16 | 4, 4, 8, 8, 8, 5, 5, 5, 6, 6, 6, 8, 8, 8, 8, 5 | 2 × blocks |
| BC7 | 16 | 4, 4, 8, 8, 8, 8, 8, 7, 7, 5, 6, 8, 8, 8, 8, 5 | 2 × blocks |

A lane's alphabet index is `alphabet_bits = width − 1`.

**Fragment header.** Each lane reads its own fragments from the bytes the
feeder hands it:

```text
byte 0: bits 7..6  z      alphabet narrowing: eab = alphabet_bits - z
                          (clamped at 0; eab > 8 is an error)
        bit  5     long   1 = 16-bit length, 0 = 8-bit length
        bit  4     flush  reset the state to 0 when this fragment ends
        bits 3..0  q      static model number 0..15
byte 1: short form: payload length = byte1 + 1
        long form:  payload length = (byte2 << 8 | byte1) + 257
```

The payload length counts every byte the lane reads in the fragment,
including those that form the initial state.

**State and decode step.** 32-bit state x, L = 2^24, 16-bit probabilities.

```text
at the start and after a flush: x = 0
while x < L and payload remains:  x = (x << 8 | next_byte) mod 2^32
    # so the first four payload bytes are the big-endian initial state
when x > L and the lane still owes symbols:
    cf = x & 0xFFFF
    s  = the symbol whose cumulative range contains cf   # model (eab, q)
    x  = (freq(s) * (x >> 16) + cf - cum(s)) mod 2^32
when the payload is used up:
    if x == L: the next fragment header follows
    else: keep emitting without reading until x <= L
```

After its last symbol a lane's state must be L, L >> 8 or L >> 16 (the
encoder may drop up to two trailing zero bytes); anything else means a
wrong lane width, lane count or symbol count. This is the all-or-nothing
check.

**Static models.** 9 alphabet sizes (eab 0–8, alphabets of eab + 1 bits)
× 16 models. Symbols are grouped in segments: segment 0 is symbol 0,
segment k ≥ 1 is symbols [2^(k−1), 2^k). Each segment has a cumulative base
and one frequency shared by its symbols, so a model is at most 10 (base,
frequency) pairs. Cumulative value 0 is reserved: a symbol whose
cumulative frequency would be 0 gets (frequency − 1, cumulative 1). Exact
division inside a segment finds the same symbol as the hardware's
comparator chain and reciprocal multiply for every cf of every model. The
(base, frequency) values for all 144 models are in
[§6A.1](06a-bcpack-tables.md#6a1-static-rans-models) (copied from the
vendor's published hardware description; no formula that reproduces them
is known).

**Lane feeder.** Lanes run on a common clock with two phases:

1. Phase 0: a lane holding a decoded symbol hands it to its output slot
   (only if the slot is empty) and applies the state update. Lanes that
   want a byte request one.
2. Requesting lanes are served in lane order 0, 1, 2 … from the single
   stream, **at most 8 bytes per clock**. A lane that asks and is not
   served stalls for that clock.
3. Phase 1: header parsing, renormalisation and symbol decode.
4. Lockstep output: each lane has one output slot, and all slots are
   released together once every lane that still owes symbols holds one,
   so symbols leave one vector at a time.
5. A flush is handled per lane as above; the feeder does not treat it
   specially.

On 120 sampled BC1 chunks, no chunk converges when any one rule changes (a
window of 7, 9 or 16 bytes, reverse lane order, free-running lanes, or a
symbol count off by one).

**Output order** (all codecs; high): stream order is the texture's stored
(tiled) order. The decoded blocks replace the chunk's virtual bytes
directly; detile as for a zlib chunk.

### BC1 blocks (codec `0x04`)

(Fitted constants; structure high.)

- Lane 0: distance d = unzigzag(v) mod 256 (v even: v >> 1; v odd:
  255 − (v >> 1)). d = 0 predicts zero for all six fields; d ≤ i predicts
  from block i − d; d > i reads preset slot (i − d) mod 256. Lane 1 is
  always 0.
- Lanes 2–7: zigzag residuals (0, −1, +1, −2 …) of R0 G0 B0 R1 G1 B1,
  added modulo the field width (5, 6, 5, 5, 6, 5). Fields never mix
  between the two endpoints.
- Three-colour blocks: there is no swap flag and no special handling;
  c0 ≤ c1 occurs and is written as decoded. Such blocks fit the next mip
  slightly worse, and no separate table was found (open).
- Lanes 8–11: selector quadrants top-left, top-right, bottom-left,
  bottom-right. Each symbol indexes a fixed 256-entry codebook of 2 × 2
  patterns of 2-bit codes (a bijection onto all 256 patterns, roughly
  frequency-ordered). Symbols 0–3 are the uniform quadrants with code
  `sym XOR 2`. Selectors are not predicted.
- Presets: 254 of 256 slots fitted; slots 0 and 4 are never referenced.
  Codebook: two fits on disjoint halves of the data agree on 248 of 256
  entries; the other 8 are 4 rare mirror pairs (229/235, 230/231,
  247/248, 254/255). The preset slots and codebook are in
  [§6A.2](06a-bcpack-tables.md#6a2-bc1-preset-slots) and
  [§6A.3](06a-bcpack-tables.md#6a3-bc1-selector-quadrant-codebook).
- **Three-colour index check** (2026-11-24; a near-exact rule, not exact,
  since real cut-out textures break it): a block in three-colour mode
  (c0 ≤ c1 as 16-bit values) does not use index 3. On a 5 % sample of the
  102,725 BC1 chunks, chains that start at the default prediction break it
  in 1,705 of 9,402,103 blocks (0.018 %), and c0 = c1 blocks never use
  index 3 (0 of 168,197). Chains that start at a preset slot break it in
  121,918 of 85,387,017 (0.14 %), concentrated in a few slots (192: 55 %,
  127: 37 %, 152: 31 %, 33: 24 %), so some
  [§6A.2](06a-bcpack-tables.md#6a2-bc1-preset-slots) presets are wrong.
  Refits of slots 33, 48, 73, 120, 159, 166, 194, 208 and 222 pass both
  tests (fewer violations on held-out chunks, 28,387 → 907 in the sample;
  error against plain child mips 11.6 % lower, at least 30 reference
  blocks each; medium-high). *Superseded by the recount below.* 31 more slots moved in the fit but failed a test (no plain
  reference, or higher reference error; probably real cut-out textures),
  among them 192, 127 and 152; their true values are unknown.
  *Revised (2026-11-24; medium, all fitted):*
  a slot's plain references come from few textures (often under 50), so
  counting blocks overstated the evidence above. Counting each texture once
  and validating on held-out textures, 6 of the 9 refits go back to the
  third-party values (48, 73, 120, 159, 194, 208), 3 stay (33, 166, 222)
  and 17 more slots are refitted: 20 replaced in all. Two independent
  checks agree. Against the third-party window, the mip check of 35,533
  BCPack BC1 textures has 2,003 better and 207 worse (837 more than 2 dB
  better, 1 worse); under 20 dB 0.77 → 0.39 %; mean 33.58 → 33.79 dB.
  Three-colour blocks using index 3 on preset-rooted chains (5 % sample):
  121,918 → 50,334. Open: slot 152 (31 % of its blocks break the rule;
  references inconclusive), slot 153 (the reference and mip checks
  disagree), and slots 0 and 4 (no plain reference). The 20 slots (3, 27,
  28, 33, 69, 80, 81, 102, 115, 117, 125, 127, 140, 142, 160, 165, 166,
  192, 197 and 222) replace the third-party values in
  [§6A.2](06a-bcpack-tables.md#6a2-bc1-preset-slots), marked `*`.

**Prediction (BC1, BC3, BC7).** Every block is coded as residuals against a
reference: the zero state (distance 0), block `i − d` of the same chunk in
stream order (`d ≤ i`), or, for `d > i`, slot `(i − d) mod 256` of a fixed
256-slot **history table**, separate per codec. Values add as
`reference + unzigzag(residual)` modulo the field width. "E" and "O" below
are the two symbols a lane emits per block (E first, O second).

## 6.2 BC3 blocks (codec `0x0C`)

Decodes on all 35 chunks (2026-11-22). Symbol widths in bits per lane:
`4, 4, 8, 6, 6, 6, 6, 5, 6, 5, 8, 8` (an earlier note gave each one less,
the parameter form; the bit widths match the fields: 6-bit alpha pair
symbols, R5 / G6 / B5, an 8-bit codebook index).

| Lane | Role | Confidence |
|---|---|---|
| 0 | back-reference distance: x takes O's bits at positions 0, 2, 4, 6 and `E >> 1`'s at 1, 3, 5; `d = E odd ? 255 − x : x` | confirmed |
| 1 | always 0 | confirmed |
| 2 | alpha endpoint residuals, a0 (E) and a1 (O) | confirmed |
| 3–6 | alpha index pairs: symbols of a fixed 64-entry permutation of 3-bit index pairs | inferred (symbol 54 placed by elimination) |
| 7–9 | R / G / B endpoint residuals (5 / 6 / 5 bits), c0 (E) then c1 (O) | confirmed |
| 10–11 | colour selectors: 2 × 2 quadrants TL, TR (lane 10) and BL, BR (lane 11) from a fixed 256-entry codebook, not BC1's | inferred (codebook fitted) |

Distance 0 predicts colour 0 and alpha 128. The colour block always uses
the 4-colour palette; alpha is standard BC3. The data uses 35 of the 256
history slots; they and the selector codebook were fitted against
neighbouring mips (no plain copy of any mip 0 exists).

**Colour endpoint order** (2026-11-24, high): colour endpoint 0 is never
below endpoint 1 as a 16-bit 565 value (0 of 61,383 blocks on chains
without presets). With it as a constraint, preset-rooted violations fall
from 727 of 534,073 to 13, and the two slots that held 713 of them go from
about 36 dB to 41.4 and 38.9 dB against plain child mips. The same gap
rule as BC4 (below) holds for BC3 alpha without presets (4 of 61,383
exceptions), but enforcing it on the alpha presets raised the error, so
they are unchanged (open).

## 6.3 BC7 blocks (codec `0x1C`)

Decodes on all 5,167 chunks (2026-11-22). Lanes 0 and 1 are not BC3's
distance lane. `il(E, O)` interleaves two symbols bit by bit, E on the even
bit positions and O on the odd ones.

- **Distances:** index distance `di = il(E1, O1)`; endpoint distance
  `de = (di + unzigzag(il(E0, O0))) & 255` (confirmed).
- **Mode, partition, rotation, index selection** come from the lane-15
  symbol pair. Sort the 285 observed pairs (a, b) by ((a + 1)(b + 1), a):
  ranks 0–63 = mode 7 partition 63 − r; 64 = mode 6; 65–68 = mode 5,
  rotation 3 − (r − 65); 69–76 = mode 4 (k = r − 69, rotation 3 − k / 2,
  index selection 1 − k mod 2); 77–140 = mode 3 partition 63 − (r − 77);
  163–226 = mode 1 partition 63 − (r − 163). Ranks 141–162 and 227–284
  are modes 2 and 0 in an order fitted from the data (inferred).
- **Endpoints:** the state is 3 subsets × 2 endpoints × RGBA, 8-bit
  MSB-aligned. A field of width w reads `state >> (8 − w)`, adds the
  unzigzagged residual mod 2^w and is stored back `<< (8 − w)` (confirmed).
  Subset 0 uses lanes 2, 3, 4 (, 5 for alpha), subset 1 lanes 6, 7, 8 (, 9),
  subset 2 lanes 10, 5, 9 (R, G, B; 98.3 % fit). RGB-only modes 0–3 set
  alpha to 0; subsets a mode does not use copy subset 0.
- **Alpha presets** (2026-11-23; tables medium): earlier BC7 preset fits
  scored only RGB, so the alpha preset values had never been fitted. With
  alpha in the fit, on 1,433 BC7 textures (mip 0 against mip 1): alpha mean
  39.8 → 48.0 dB, textures with alpha under 30 dB 40.8 → 18.4 %; colour mean
  39.1 dB, median 38.1 dB, 5.0 % under 30 dB, 1.5 % under 20 dB. A second
  alpha refit is running.
- **Rules tested and rejected** (2026-11-23): each scored worse than the
  rules above on table-independent blocks against plain references, so the
  text above stands: unused subsets keeping their previous state, unused
  subsets set to zero, unused subsets copying only colour, RGB-only modes
  keeping alpha, and endpoint state held in block-field order for rotated
  mode 4/5 blocks.
- **Field widths** (from lane maxima): mode 0 R/G/B 5/4/4 (R includes the
  p-bit), 1 7/6/6, 2 5/5/5, 3 8/7/7, 6 8/7/7/7, 7 6/5/5/5, mode 4 5/6 and
  mode 5 7/8 (colour/alpha). Mode 6 uses R8 = 2·R7 + p.
- **Indices:** lanes 11–14 hold rows 0–3, E columns 0–1 and O columns 2–3;
  within a symbol the even bits are one pixel and the odd bits the other
  (rows 0–2 left = odd, right = even; row 3 swapped). Index = reference
  index (block `i − di`) + unzigzag(residual) mod 16, 4-bit MSB-aligned;
  `di = 0` gives zeros. Modes 4/5 put the second (2-bit) index set in lanes
  6, 7, 8, 10 with the same layout.
- **Secondary index state resets** (2026-11-23, high; corrects the earlier
  decode): a block with no second index set (every mode except 4 and 5)
  **resets the second index state to zero**; it does not carry the previous
  block's second set forward. Evidence is exact, not fitted: an anchor index
  never has its top bit set, and on chains that do not touch the preset
  tables the old rule broke that on 29 % of chained mode 4/5 blocks, the new
  rule on 0 of 2,249,610 (first 300 BC7 chunks). Mip 0 against mip 1 over
  1,433 BC7 textures: mean 35.1 → 38.7 dB, median 33.5 → 37.9 dB, under 30
  dB 27.1 → 6.8 %, under 20 dB 2.0 → 1.7 %; 917 textures gain more than
  1 dB, none lose more than 1 dB.
- **Anchors:** blocks whose anchor index has the top bit set (5.4 %, all
  modes 4/5) are made legal by swapping the endpoints and inverting the
  indices, which leaves the pixels unchanged (confirmed). *Probably mostly an
  artefact of the old secondary-state rule above, since with the reset no
  table-independent block has such an anchor (inferred; not re-counted).*
- **Anchor rule as a check** (2026-11-23, high; BC6H and BC7): in every
  block whose index chain stays inside the chunk, each anchor index decodes
  with its top bit clear: BC6H 0 violations in 2,433,955 anchor reads (all
  268 chunks), BC7 0 for the primary set in every mode. This gives exact
  constraints on index preset values at anchor pixels. Over the full BC7
  corpus with the secondary reset: 0 violations in 24,316,547 anchor reads
  across all 5,167 chunks, both index sets, every mode. The fitted BC7 index
  presets needed only 7 value changes to satisfy it (374 violations in
  131,497,437 table-dependent reads before, 0 after); 1,166 of the 2,942
  preset entries read as anchors are pinned to one exact value.
- **BC6H no-wrap rule** (2026-11-23, high): in transformed modes an
  endpoint is w plus a signed delta, and it never leaves [0, 2^bits): 0
  wraps in 1,606,398 table-independent endpoints (all 268 chunks). The
  fitted endpoint presets break it in 23,162 of 30,227,202 table-dependent
  endpoints (0.077 %, concentrated in a few slots), so those values are
  wrong. *Refit (tables medium-low):* with wraps penalised in the fit, 599
  remain. BCPack-only mip check: mean 18.7 → 24.7 dB, median 19.6 →
  24.0 dB, under 20 dB 51.7 → 23.6 %. Plain-reference check: parent mip
  unchanged (87.7 % of blocks within tolerance, 28.6 dB), child mip 86.1 →
  88.4 % and 18.5 → 23.6 dB; plain BC6H scores 29.1 dB. The fit has
  converged (a third round moved the BCPack-only score by 0.1 point), so
  further gains need new evidence rather than more rounds.
  *Refit without the lighting store (2026-11-24; medium):* since the lighting store's mips are prefiltered and do not
  predict each other, its chunks only feed the no-wrap rule, not the fit
  targets. Blocks breaking the rule fall from 599 to 221. Plain references:
  down-mip 24.6 → 25.1 dB, up-mip unchanged at 32.2 dB. Mip check of the
  89 BCPack-only BC6H textures: median 24.0 → 24.9 dB (29 better, 4 worse),
  under 20 dB 23.6 → 22.5 %. The share of blocks within tolerance did not
  move. The 221 remaining violations show some preset values are still
  wrong. *Corrected BC6H gap (2026-11-24, medium, small sample): the
  earlier "−3.9 dB" and "about 4 dB below plain" came from a check run
  with an older BC6H pixel decode and an incomplete plain baseline.
  Re-measured against every plain texture of the same store and size (82
  matched textures, lighting store left out), BCPack BC6H is about 1.1 dB
  below plain; 16.9 % of BCPack BC6H textures are under 20 dB against
  2.1 % of plain ones.* The BC6H tables in
  [§6A.4](06a-bcpack-tables.md#6a4-fitted-bc3bc7-tables) are this refit
  (39 endpoint and 7 index slots changed).
- **Mode 1 p-bits** (2026-11-23, finding only): mode 1 has one p-bit per
  subset, but the stream carries channel 0 of both endpoints with a p-bit
  position; on table-independent chains the two always agree (0 of 247,606
  differ). The fitted endpoint presets break this in 17,451 of 447,824
  table-dependent cases, so some preset values are provably wrong. *Later
  count over all BC7 mode 1 data:* 38,866 of 8,853,332 still mismatch.
  *Exact rule (2026-11-24, high):* both endpoints of a mode 1 subset carry
  the same p-bit position (bit 1 of the 8-bit MSB-aligned channel-0
  state): 0 of 3,359,376 subsets differ on chains without presets. Used as
  a constraint, it fixes the preset-rooted cases from 31,054 of 8,853,332
  to 0 (103 of 478 slot / subset entries change, in the low two bits of
  channel 0 only).
- **History:** endpoints and indices both have a fixed 256-slot table. Each
  chunk's root blocks only fix the top w bits their mode reads, so the
  table was fitted by voting per value at each root's width (inferred).
  Every slot that any chunk uses is now fitted (0 blocks on an unfitted
  slot; an earlier count was 137).

Result on 1,029 chunks against the next mip up, downsampled: 65.5 % of
blocks within 3 levels, median 35.4 dB, no invalid BC7 blocks. Mis-fitted
history slots show as speckles near chunk starts; 12 % of the checked
chunks are under 20 dB, partly from poor reference mips.

## 6.4 BC4 and BC5 blocks (codecs `0x10`, `0x14`)

Decode on all chunks (2026-11-22). Distance 0 predicts 128; endpoints are
modulo 256. Each format has its own fixed 64-entry table of 3-bit index
pairs (fitted, inferred).

| | BC5 | BC4 |
|---|---|---|
| lanes, symbol widths | 12: `4, 4, 8, 8`, then 8 × 6 | 12, the same but lanes 0–3 are 8-bit |
| symbols per block | two (E, O) per 16-byte block | **one 8-byte block per symbol**: block 2j = E, 2j + 1 = O |
| lane 0 | distance, as BC3 | distance = unzigzag(v) mod 256 |
| lane 1 | unused | |
| lanes 2 / 3 | channel 0 / channel 1 endpoint residuals (E = e0, O = e1) | e0 / e1 residuals |
| index lanes | 4–7 channel 0, 8–11 channel 1: lane 4 + k has E = pixels 4k, 4k + 1 and O = 4k + 2, 4k + 3 | lane 4 + k = pixels 2k, 2k + 1 |

BC4 and BC5 differ only in lanes 0–3 (8 bits each in BC4; 4, 4, 8, 8 in
BC5). BC4's distance is unzigzag(v) mod 256 like BC1's; BC5's is BC3's
bit-interleaved form (widths confirmed, meaning high).

**Endpoint gap** (2026-11-24): e0 − e1 is never in −4 … 0, so 8-value
blocks have e0 > e1 and 6-value blocks have e1 at least 5 above e0.

- BC4 (high): 0 of 52,049 blocks on chains without presets. As a
  constraint it cuts preset-rooted violations from 407 of 149,679 to 31;
  against plain child mips 38.12 → 38.31 dB.
- BC5 (medium-high, near-exact): 286 of 1,256,644 endpoint pairs break it,
  leaving out the flat 128 / 128 default blocks (23,296 per channel).
  Applied only where the child-mip error allows: preset-rooted violations
  380,478 of 8,524,092 → 1,052; 38.98 → 39.60 dB. Whether the remaining
  exceptions are real or mark further preset errors is unknown.

## 6.5 BC6H blocks (codec `0x18`)

Decodes on all 268 chunks (2026-11-22). 16 lanes, symbol widths
`4, 4, 8, 8, 8, 5, 5, 5, 6, 6, 6, 8, 8, 8, 8, 5`, two symbols per block.
Distances as BC7 ([§6.3](#63-bc7-blocks-codec-0x1c)).

- **Mode:** lane 15 E = mode 0–13 (BC6H modes 1–14 in the standard order),
  O = partition.
- **Endpoint state:** every stored field (w, x, y, z of each channel, as
  the block stores them; x, y, z are deltas in transformed modes) has a
  16-bit MSB-aligned state. A field of b bits = `(state >> (16 − b)) +
  unzigzag(residual)` mod 2^b. Residuals: w = `il(E, O)` of lanes 2–4, x =
  `il(E, O)` of lanes 5–7, y = E and z = O of lanes 8–10. Distance 0
  predicts `0x8000` for every field.
- **12- and 16-bit w:** these modes hold w's bits from bit 10 up reversed
  relative to the block field, as the BC6H block layout stores them.
- **One-region hand-over (confirmed by the fix it gave):** a one-region
  block passes its state on as z = x; y = w for the untransformed 10-bit
  mode; for transformed one-region modes y = `(w << (16 − delta_bits)) XOR
  0x8000`. A later two-region block therefore predicts y from the low bits
  of w. This removed the speckles after mode changes (blocks within 0.15
  stops: 81 % → 86 %; 23 → 27 dB). Refitting the preset endpoint slots
  (below) then gave 88 % and 28 dB (median per texture 29 → 35 dB;
  BCPack-only textures 58 % → 62 %). The last 17 endpoint slots (81,093
  blocks, 1.56 %) were then fitted, 15 against the next smaller mip and 2 by
  edge continuity (below): edge cost 1.81 M → 36.5 K, reference scores
  unchanged (medium-high for the 15; **medium-low for the 2 edge-fitted
  slots**: their relative structure is right, but their absolute level is
  not verified, see the limit of edge continuity below).
  With perfect presets the reference score would top out at 94.3 % of
  blocks within 0.15 stops.
- **Open:** the x prediction after a change from mode 12 to mode 11 is
  sometimes off by +25 to +30 (44 blocks seen).
- **Signedness:** every BC6H texture in the stores is the unsigned format,
  so only unsigned was tested. The BCPack layer codes the stored fields
  and does not depend on signedness; a signed texture would differ only in
  the final block-to-pixel step (not tested).
- **Indices:** the BC7 scheme with a 4-bit MSB-aligned state, 3-bit reads in
  two-region modes; distance 0 predicts 8. Anchor top bits are dropped
  (truncating fits better than keeping them). *Superseded (2026-11-23,
  high):* an anchor index never has its top bit set on chains that stay
  inside the chunk (the [anchor rule](#63-bc7-blocks-codec-0x1c)), and the
  published index presets satisfy it, so there is nothing to drop; the
  earlier truncation was compensating for wrong preset values (inferred).

Chunks of the lighting stores have inconsistent neighbouring mips (a flat
mip 3 under a mip 1 with a logo), so they cannot be scored.

## 6.6 Ruled out

- The BC1 index lanes do not need an adaptive model or a separate
  extension-lane parser. The fragment header selects a static rANS model.
- A separate sub-stream is not needed solely because a BC1 chunk has
  1,536 or 3,072 blocks. Five chunks of each size decoded as one stream
  in both implementations, with matching symbols in every lane and
  exact input consumption.
- Matching symbol streams does not settle reconstruction. One tested
  BC1 terrain chunk differs at six output byte positions even though
  every lane matches. All six differences trace to one endpoint preset
  and two selector symbols.

## 6.7 Open unknowns

- **Fitted tables (BC3, BC4, BC5, BC6H, BC7):** the history slots, the BC3
  selector codebook and the index-pair tables are fitted against
  neighbouring mips (per-slot coordinate descent over the blocks each slot
  roots). BC6H preset endpoint slots are scored by the full pixel error with
  the decoded indices; an index-free best-palette score was too lenient on
  the second region's first endpoint. No plain copy of any affected
  mip 0 exists in the install, so nothing confirms them byte for byte.
- **Edge continuity** (2026-11-23): preset slots reached only by textures
  with no neighbouring mip (texture arrays, lighting textures stored as
  mip 0 only) are fitted by minimising the pixel difference across block
  edges between the blocks rooted at the slot and their neighbours. For
  BC5 this is exact arithmetic: endpoints are prediction + residual mod
  256, so a slot is a constant offset on every block rooted at it. The last
  13 BC5 slots (1,811 blocks in 18 chunks, 15 of them in texture arrays)
  were fitted this way, about 6 / 255 per edge pixel on the large slots
  (medium: no reference mip; slots with 1–2 blocks are weakly determined).
- **Limit of edge continuity** (2026-11-23, measured): it cannot fix a
  constant per-channel offset when (nearly) every block of a texture is
  rooted at the slot being fitted, because a uniform shift leaves all
  edges continuous. A first edge-only BC7 fit of the 34 slots gave mip 0 vs
  mip 1 agreement of median 27 dB over 71 textures, 21 under 20 dB, with
  blue-channel biases of +30 to +105 and −129 on the bad ones (red and
  green fine). The BC5 slots are anchored (their blocks are a minority
  inside fitted neighbours, e.g. 440 of 4,096 per chunk), so they stay
  medium. The 2 edge-fitted BC6H slots are mostly unanchored (64,431 of
  65,536 and 16,384 of 16,384 blocks rooted at the slot, in lighting
  textures with mip 0 only); the render has a black background and smooth
  glow, but their absolute level is unknown. BC7 was refitted against the
  decoded next smaller mip instead, which pins the level (next item).
- **BC7 last slots** (2026-11-23): the last 34 endpoint preset slots
  (839,119 blocks in 251 chunks, 1.05 % of BC7 blocks, 17.7 % in the
  content-pack stores) are fitted against the decode of the next smaller
  mip, in two passes (first only child mips that decode from fitted slots,
  59 % of the blocks, then all, 85 %), plus a small edge-continuity term.
  **Every codec now has 0 unfitted blocks** over all 108,605 chunks. Check,
  mip 0 against mip 1 on the 71 affected textures: median 33.4 dB, 53 at
  30 dB or more, 4 under 20 dB (edge-only fit: 27.2 dB, 27, 21); the score
  on the 1,029 chunks with plain references is unchanged. Medium-high for
  the 30 slots with many child-mip targets, medium for small slots (4 have
  under 100 targets). Open: the 4 textures under 20 dB did not move between
  passes, so their error has another cause; the preset values remain
  fitted, not known. The public patent material on this codec covers only
  the entropy layer, not the block history or presets.
- **BC7 refinement** (2026-11-23): every BC7 preset slot, endpoint and
  index, was refined field by field, scored on the blocks each slot roots
  in the 1,029 chunks that have a plain neighbour mip. 185 endpoint and 54
  index slots changed; good blocks on that reference rose from 1,163,847
  to 1,264,377. Mip 0 against mip 1 over 1,431 BC7 textures in the whole
  install: textures under 20 dB fell from 6.7 % to 2.5 %; 283 gained more
  than 1 dB and 64 lost more than 1 dB; the median rose from 39.5 to 42.5
  dB for textures whose neighbour mip is uncompressed and from 32.0 to 32.4
  dB for the rest. High that this is a net improvement (it also holds on
  textures outside the fit set); medium per slot. Six textures with no
  plain reference drop below 20 dB, so the slots they use are still not
  the true values; the exact presets remain unknown.
- **BC6H refit of unreferenced slots** (2026-11-23): the 111 endpoint and
  29 index slots that have no plain reference were refitted index-aware
  (full pixel error with the decoded indices) against the decoded
  neighbour mips in both directions; slots with a plain reference stayed
  frozen. Repeated for five decode-and-fit rounds, it converged: rounds 4
  and 5 changed nothing on the independent check, so the method is
  exhausted. The reference set is unchanged (87.7 % of blocks within 0.15
  stops, 28.1 dB). On textures stored only as BCPack, good blocks against
  the larger neighbour rose from 61.1 to 71.4 %; mip 0 against mip 1 on
  the 89 textures in the quality table above: median 18.0 to 19.6 dB,
  under 20 dB 58.4 to 51.7 %. (After one round the figures were 65.0 % and
  18.6 dB.) Medium: the targets are themselves BCPack
  decodes, so this shows self-consistency, not ground truth. BC6H remains
  the weakest codec: plain BC6H scores 29.1 dB on the same measure, so
  about 10 dB is still preset error or an unknown rule.
- **Joint BC6H refit** (2026-11-23, medium-high): a slot fitted only from
  blocks with a plain reference can be badly wrong in fields those blocks
  never read. One texture's second mip decoded as noise (2 dB) while its
  neighbours were fine, with 90 % of its blocks rooted at such a slot.
  Refitting all slots jointly, with the plain references weighted higher,
  raised the BCPack-only match from 71.6 to 75.8 % and the mip 0 against
  mip 1 median from 19.6 to 22.0 dB (36 % under 20 dB), with the reference
  set at 87.7 % / 28.5 dB. These are first-round figures; more rounds are
  running.
- **How to reproduce a fit** (most reliable first):
  1. Slots with a plain reference: per-field coordinate descent over the
     blocks each slot roots, scored against the plain neighbouring mip.
     For BC6H, score full pixels with the decoded indices, not endpoints
     alone.
  2. Textures stored only as BCPack: alternate decoding and fitting
     against the decoded neighbour mips in both directions.
  3. Child mip: compare 2 × 2 block means with the next smaller mip.
  4. Edge continuity (see its limit above).
  5. Do not freeze a slot because it has a plain reference: those blocks
     only constrain the fields their modes read. Fit jointly, with the
     plain references weighted higher.
  6. After any refit against a subset, check each changed field on every
     texture of the codec and keep the old value where it is better.
  7. Use the exact rules (anchor top bit clear, BC6H no-wrap, mode 1 p-bit
     agreement) as hard constraints. Do not fit BC6H delta presets by
     centring the deltas on zero without reference pixels: it lowered the
     plain-reference score from 87.7 to 83.8 %, so deltas are not
     zero-centred.
- **Fitted tables** (not game data): BC3 presets (256 slots, colour and
  alpha), its colour quadrant codebook (256 entries, BC1's layout with
  other entries) and alpha index-pair table (64 entries of two 3-bit
  indices); BC4 and BC5 presets (256 slots) and index-pair tables (64);
  BC6H endpoint presets (256 slots × 4 fields × 3 channels, 16-bit) and
  index presets (256 × 16, 4-bit); the BC7 header table (285 (E, O) pairs
  to mode, partition, rotation and index selection), endpoint presets
  (256 slots × 3 subsets × 2 endpoints × RGBA, 8-bit) and index presets
  (256 × 2 sets × 16, 4-bit). **Published** (2026-11-23; BC3, BC4, BC5
  and BC7 revised 2026-11-24 under the exact rules above, BC6H refitted
  2026-11-24 without the lighting store, with the same slots fitted) in
  [§6A.4](06a-bcpack-tables.md#6a4-fitted-bc3bc7-tables) and
  [`data/bcpack-fitted-tables.json`](data/bcpack-fitted-tables.json).
  Slots fitted: BC3 35 of 256, BC4 172, BC5 214, BC6H endpoints 226 and
  indices 251, BC7 endpoints 248 and indices 255; the rest are read by no
  chunk. BC3 is the least reliable (34.3 dB against 50.3 for plain BC3, on
  16 textures).
- **Final BC7 figures** (2026-11-23, after a second alpha-aware refit; mip
  check on 1,433 textures): colour mean 39.2 dB, median 38.1 dB, 4.6 %
  under 30 dB, 1.2 % (17 textures) under 20 dB; alpha mean 49.2 dB, 13.2 %
  under 30 dB. By format: median 37.6 dB on the 1,133 textures of DXGI
  format 98 and 44.4 dB on the 300 of format 99 (plain BC7 40.9 dB). BC6H
  unchanged: median 24.0 dB, mean 24.7 dB, 23.6 % under 20 dB (plain BC6H
  29.1 dB).
- **Anchor-consistent BC6H tables** (2026-11-23; tables medium-low): a
  joint fit of all slots on plain-reference and bootstrap chunks, then index
  presets made anchor-consistent: 992 slot / anchor entries (pixels 0, 2, 8,
  15), 720 pinned to one exact value, 235 to two, 0 violations left. Mip
  check on BCPack-only textures: median 19.6 → 22.1 dB, under 20 dB 51.7 →
  36.0 % (plain BC6H 29.1 dB); plain-reference check unchanged at 87.7 %,
  28.6 dB. BC6H is still not exact. (Blocks whose endpoint chain starts
  inside the chunk score lower on that check, but that comes from three
  high-contrast modes, the 7-bit / 6-bit-delta two-region mode and both
  untransformed modes, which score low against a downsampled reference
  wherever they occur; zero-distance blocks of the 10-bit untransformed
  mode match the reference per-channel ranges. No BC6H rule error is
  indicated.)
- **BC7:** the order of the mode 0 / 2 headers is fitted (medium).

**BC1/3/4/5:** Confirm the disputed or fitted endpoint presets and selector
mappings against an independent copy of the same blocks. Two software
implementations agreeing is useful but is not a hardware reference.

**BC6H/7:** Confirm the preset endpoint and selector histories and the
remaining BC7 signature mappings. The candidate predictors can be tested
against standard BC anchor rules. Those rules constrain some initial
selector values, but do not constrain texels that are never anchors. In
one tested BC7 vehicle chunk, changing only such a preset selector field
gave two structurally legal reconstructions with 26 different visible
pixels.

Using the candidate reconstruction, six complete mips (three mips each
in two BC7 vehicle chunks) render without reading any initial history.
Two different initial tables gave identical output on those mips. The
vehicle surfaces use the 64 KiB `SW_64K_D` order; using `SW_64K_S`
scrambles them despite identical tile sizes. Two detilers and two BC7
pixel renderers agree on the corrected previews. These checks isolate
the preset dependence and layout but do not prove the original BCPack
blocks byte for byte.

## 6.8 How to resume

1. Check the entropy decoder first. Derive counts from the chunk table,
   remove only the documented padding and compare all lane symbols
   before comparing BC blocks.
2. Compare raw tiled BC blocks before detiling or rendering. A useful
   reference is a readable, independent copy of the **same** blocks at
   the same mip level. A nearby mip, a valid DDS, or a plausible picture
   cannot establish exact bytes. Local plain-mip searches did not find
   byte-exact references for the tested BPTC targets.
3. For a disagreement, trace the affected fields to a specific history
   root or codebook symbol. Test the proposed value on held-out chunks
   separately from those used to derive it.

For extraction without verified reconstruction, use a raw or zlib mip
where available ([section 5](05-ttpf.md)) and label fitted outputs
provisional.

---

← [5. `ttpf` texture stores](05-ttpf.md) · [Index](README.md) · [7. Drawables, fragments and vehicles](07-models.md) →
