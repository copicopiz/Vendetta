# 3. Audio — `ADAT` containers

## Header

| Off | Size | Meaning |
|---|---|---|
| 0x00 | 4 | magic `ADAT` |
| 0x04 | 4 | flags (values observed: `0xFF020001`, `0xFF210001`, `0xFF250001`, `0xFF610001`) |
| 0x08 | 4 | stream count |
| 0x0C | 4 | header length — the chunk table ends exactly here |

Optional tables follow the fixed header, selected by flag bits:

| Flag | Table |
|---|---|
| `0x00400000` | one extra u32 word |
| `0x00010000` | u16 first-chunk index per stream |
| `0x00200000` | u32 × (stream_count + 1) |
| `0x00040000` | multi-channel / music variant (512 KiB block framing) |

Then, per stream, a u32 of the form `id:29 | chunk_count:3`, followed by the chunk tag table.

## Chunk tags

Each tag is a u64: `type:8 | size:28 | offset:28`. Type bytes are the low byte of a joaat of the
chunk name:

| Type | Chunk |
|---|---|
| 0x36 | peak |
| 0x48 | stream_format |
| 0x55 | data |
| 0x5C | animation |
| 0x76 | format |
| 0xA3 | seek_table |
| 0xB2 | metadata |
| 0xBD | markers |

## Data framing

Single-channel streams store **Opus packets in 2,048-byte blocks**. Each block is a sequence of
`u16 little-endian length + packet`, zero-padded to the block end. A zero length ends that block;
blocks are independent.

Consequence: `0x55` chunk sizes are exact multiples of 2,048 when the payload includes its block
header, which is why raw size statistics show a 256-byte period rather than a 2,048-byte one.

Music/multi-channel streams use large blocks (the size is in the `0x48` chunk, e.g. 512 KiB or
`0x1B9000`) with 2,048-byte sub-blocks, selected by the `0x00040000` flag. See
[Music streams](#music-streams-sub-streams) for the layout.

## Format chunk (`0x76` / `0xFA`)

20, 24 or 32 bytes:

| Off | Meaning |
|---|---|
| 0x00 | u32 sample count |
| 0x04 | u32 loop point |
| 0x08 | u16 sample rate |
| 0x0A | u16 headroom |
| 0x0C | u16 loop begin |
| 0x0E | u16 loop end |
| 0x10 | u16 play end |
| 0x12 | u8 play begin |
| 0x13 | u8 codec (0x00 = Opus): earlier reading, **superseded** for the 32-byte form by the decoding evidence below |
| 0x14 | u32 peak (24-byte form) |
| 0x1C | 32-byte form: u8 codec id, u8 channel count, u16 0 (see below) |

**32-byte form, codec and channels (confirmed by decoding):** the codec id is
the u8 at `0x1C` and the channel count the u8 at `0x1D` (1 mono, 2 stereo).

| Codec id | Codec | Check |
|---|---|---|
| 0 | PCM signed 16-bit LE | data bytes / samples = 2.000000 on every stream |
| 1 | ADPCM | not present in this build |
| 2 | PCM float32 LE | data bytes / samples = 4.000000 on every stream |
| 5 | XMA2 | every sampled stream decodes as XMA2; 32,000 Hz mono |
| 11 | Opus | packets start `0xF8` (mono) or `0xFC` (stereo), 20 ms each |

Over 1,606,240 streams: Opus is about 99.3 % of payload bytes, XMA2 0.45 %,
float32 PCM 0.26 %, PCM16 0.001 %. One container can hold streams of more than
one codec: per-codec container counts sum to 7,162 over 7,159 containers
(confirmed). By stream: 1,818 XMA2 (mono, 32 kHz), 227 float32 PCM and 19
PCM16 streams; the rest are Opus. Reading the codec from byte `0x13`
instead mislabels 514 of these non-Opus streams as Opus.

## Music stream-format chunk (`0x48`)

Long-form and music streams carry a `0x48` chunk instead: u32 block count, u32
block size (`0x00080000` = 512 KiB), u32 entry count (1–8), then 16-byte
entries:

| Off | Meaning |
|---|---|
| 0x00 | u32 stream id |
| 0x04 | u32 sample count, counting **both channels** of a stereo entry: frames = count / 2 for stereo (this is why it looked unusable as a duration) |
| 0x08 | u16 (gain-like) |
| 0x0A | u16 sample rate (48000) |
| 0x0C | u8 codec id (same ids as above; 11 = Opus on music) |
| 0x0D | u8 channel count |
| 0x0E | u16 `0xFFFF` |

### Music streams: sub-streams

(Confirmed by decoding all 3,008 music streams in `audio.rpf`, 2026-11-22.)
Each `0x48` entry is a **separate sub-stream**: its own mono or stereo Opus
stream, time-aligned with the others (stems of one piece). Entries per
stream: 1 in 2,958 streams, 2 in 24, 3 in 8, 4 in 13, 5 in 2, 8 in 3.

The `0x55` data is `block count` blocks of `block size` bytes; the last
block is shorter and the data ends exactly at its end (all 3,008 streams).
Each block:

1. 24 bytes per sub-stream: u32 `0xFFFFFFFF`, u32 count of 2 KiB blocks,
   u32 0, u32 samples (both channels counted, as above), u32 0, u32 0;
2. one u32 seek table per sub-stream: a sample position per 2 KiB block;
3. padding to 2 KiB;
4. each sub-stream's 2 KiB blocks in turn, packed like single-channel
   streams (`u16 length + packet`).

A sub-stream's packets run on from one block to the next. Example: stream
`7a7f000` of the minigame pack is 288 s long; reading the sample count as
frames gives 576 s.

## Markers chunk (`0xBD`)

16-byte records: u32 name hash (joaat of the lowercase cue name, e.g.
`trackid` `0xA6D93246`, `beat` `0xE89AE78C`, `intro` `0xDA8F5F53`, `outro`
`0x5D40B28D`), f32 value, u32 sample offset (48 kHz frames), u32 0. For
`trackid` the value is the song id stored as a float. Region cues use
`joaat("start")` `0x84DC271F` / `joaat("end")` `0xB0C485D7` as the value, in
start/end pairs.

## Non-Opus containers

Not every `ADAT` file carries Opus. Observed exceptions: containers with zero streams and a
20-byte header, and containers whose `0x55` chunk is uncompressed IEEE float32 PCM (a
163,840-byte chunk decodes as 40,960 samples × 4 bytes at 48 kHz). Any decoder must check the
codec byte rather than assume Opus.

## Open questions

- The meaning of `headroom` and of the `0x00400000` word.
