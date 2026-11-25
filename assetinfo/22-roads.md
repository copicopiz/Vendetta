# 16. Road graph, junctions and flags

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

GTA V `nodes*.ynd` equivalent, held in meta resources in `levels.rpf` as
one global graph.

## Nodes and links

| Struct | Size | Count | Fields |
|---|---|---|---|
| `0c69a100` node | 60 B | 63,808 | +0x00 `Position`; **+0x0C `Flags`** (51-bit set, enum `0b354003`); **+0x14 own address {u16 area, u16 index}**; **+0x18 spatial key {area, index}** (junctions use it); +0x1C another key; **+0x20 `StreetName`**: joaat of a lower-case location key ([below](#street-names)); 403 distinct non-zero hashes plus 0; **+0x24 f16 `SpeedLimit` m/s**; +0x26 i16; +0x2A enum `0fe27c12` (8 values); **+0x2B enum `391d8bab`: road class**; +0x2C `NumLinks`; +0x2D `Density` (0–15); +0x38 u32 first link index |
| `0f2da4c2` link | 20 B | 145,518 | **+0x00 target address**; **+0x04 u16 `Distance` (m)**; **+0x06 / +0x07 u8 lanes to / from the target**; +0x08 u8 (0–7); +0x0A i16; **+0x0C `Flags`** (11-bit set, enum `c2804a82`); +0x10 u16 |

A node's links are `links[first(i) .. first(i+1)]`. Every link resolves;
only 8 lack a reverse link. Median link length 11.5 m. Speed limits come in
mph steps (4.47 / 13.4 / 17.9 / 22.35 m/s = 10 / 30 / 40 / 50 mph). Nodes at
z = 0.3 are boat paths. One-way links have 0 lanes in one direction.

**Road class** (+0x2B, inferred from speeds; names not recovered): 0
freeway (60–70 mph), 1 highway (40–60), 2 major road (40), 3 30–50 mph, 4
50 mph, 5 15 mph streets, 7 / 10 residential and car parks (10 mph), 9
airport.

## Node flags (`PNF_<Name>`; 34 of 51 named, confirmed)

| Bit | Name | Bit | Name |
|---|---|---|---|
| 0 | NoGps | 21 | BarrierAhead |
| 1 | SlipJunction | 22 | HighwayExitRoad |
| 2 | SplitJunction | 23 | HasBikeLane |
| 3 | WaterNode | 24 | IsRaceTrack |
| 4 | Junction | 25 | AvoidGps |
| 5 | InTunnel | 26 | NoBigVehicles |
| 6 | GpsOnly | 27 | SmallWorkVehicles |
| 7 | SlipLane | 28 | GiveWay |
| 9 | LowBridge | 29 | Stop |
| 10 | River | 30 | Yield |
| 12 | NoThroughGps | 32 | Crossing |
| 13 | RiverCrossing | 38 | AirportTaxi |
| 14 | Bridge | 39 | AirportRunway |
| 17 | HighwaySlipRoad | 41 | Elevated |
| 18 | IsInInterior | 42 | RoadCrossing |
| 19 | TrainStation | 50 | UndergroundCarPark |

Bit 31 (8,152 nodes, all with 3+ links) is unnamed. Counts: Junction
10,775, Stop 4,412, Yield 124, InTunnel 180, Elevated 3,255.

## Link flags (`PNLF_<Name>`)

| Bit | Name | Count | Confidence |
|---|---|---|---|
| 0 | GpsCanGoBothWays (only on one-way links) | 5,284 | confirmed |
| 2 | ShortCut (unused here) | 0 | confirmed |
| 3 / 4 | LeadsToDeadEnd / LeadsFromDeadEnd | 4,056 each | confirmed |
| 5 | `[unpaved / track]`: never one-way, one lane each way, slow classes, often not on road collision | 34,694 | inferred |
| 6 | `[no ambient traffic?]`: set on every link touching a density-0 node and at dead ends | 102,939 | inferred (medium) |
| 10 | `[two-way road]`: on every two-way link, never on a forward one-way link | 90,739 | inferred |
| 1, 7 | too rare to read | 2, 4 | unknown |
| 8, 9 | unused | 0 | — |

## Junctions (list `eba8aebb`, 88 B each, confirmed)

1,546 junctions, each keyed by its node's spatial key (99.1 % are on
Junction-flag nodes). The junction resource's system page spans two zlib
streams; read the whole entry.

| Struct | Fields |
|---|---|
| entrance `abc0cc55` (80 B, 6,790) | +0x08 turn paths, +0x18 entrance node key (median 16 m from the junction), +0x46 lane, +0x47 `bIsStop`, +0x48 `bIsYield` (almost never set: stop and give-way are node flags) |
| turn path `badd6def` (176 B, 11,604) | +0x78 target node key, +0x8C `fPathLength`, +0x90 / +0x9C start / end points, +0xA8 lane, three bools |
| phase `aef55528` (24 B) | +0x00 `fDuration` (s), +0x08 lights `c47cb7b3` (16 B: u64 hash, u8 `boxId`) |

**369 junctions are signalised** (1,033 phases). A phase light hash is the
**64-bit hash at entity +0x08 of a placed traffic-light or pedestrian-
crossing prop** (99.6 % match); `boxId` picks the signal head.

## Street names

(Confirmed, 2026-11-22.) A node's `StreetName` is joaat of the
**lower-cased** `key` attribute of an `Item` in
`GameShape__Data__LocationNameLoopkup/LocationNames` (XML in `common.rpf`;
the misspelling is the game's). Keys are entity ids shaped like
`CustomEntity<n>_<m>`; only the lower-cased spelling matches. The item's
`<Name>` is a text label (`STREET_<…>_NAME`), and labels are GXT keys
([§17](17-other.md#text-gxt)), so:

```
node StreetName → location item (by key hash) → label → text
```

| Count | |
|---|---|
| nodes / named nodes | 63,808 / 21,049 |
| distinct non-zero hashes | 403 (the zero hash also occurs) |
| hashes resolving to an item key (2,345 items) | **393, 20,689 nodes = 98.3 %** |
| distinct labels behind them | 391 (two keys share a label), all with text, e.g. "SW 3rd Ave", "US-35", "I-97", "Catalan Blvd" |
| hashes of `customentity<a>_<b>` keys with no item | 10, 360 nodes |

The label hash itself, the street texts and case-sensitive keys match
nothing. One label is spelled `STEET_120_NAME` ("SR-120") in the game
files.

## Open questions

- Link flag bits 1, 5–10 names; node flag bit 31 and the other unnamed bits.
- Road class and the +0x2A enum names.
- The 10 street-name keys with no location-table item (360 nodes).
- Struct `527ce554` (24 B × 32,586: two node addresses and four floats).


---

← [15. Lights](15-lights.md) · [Index](README.md) · [17. Other formats](17-other.md) →
