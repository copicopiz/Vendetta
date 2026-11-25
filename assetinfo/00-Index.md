# Structure notes — index

Observations about the on-disk layout of RAGE 9 (Xbox Series) archives, recorded as byte-level
facts with the test that established each one. No tooling, no build references, no speculation
about intent.

| File | Contents |
|---|---|
| `01-rpf9-container.md` | archive header, encrypted table of contents, payload stream layout, chain rule, nested archives, header variants |
| `02-resources-textures.md` | resource page layout, texture descriptor, size law, texture-store descriptors |
| `03-audio-awc.md` | `ADAT` container, chunk table, Opus packet framing |
| `04-pso-objectgraph.md` | `PSIN` section walk, `LSNS` framing, the serialised object graph |
| `05-ttpf-stores.md` | `RDS5`/`ETPF` texture stores, chunk table arithmetic, codec ids |
| `06-models-shaders.md` | meshlet geometry layout, drawable name pointer, shader cache layout |

## Confidence labels

Every statement is marked:

- **OBSERVED** — measured directly from file bytes, reproducible with the stated arithmetic.
- **INFERRED** — reasoned from observed bytes, with the reasoning stated.
- **UNKNOWN** — not established; listed rather than guessed.

## Cross-reference method

Where an independent public specification exists, disagreements are recorded rather than
resolved by preference — see the conflicts section of each file. A claim that does not close
arithmetically is a hypothesis, and any conclusion drawn from it is void until it does.
