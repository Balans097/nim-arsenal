# nim-arsenal

## Curated Nim library references: annotated API summaries, usage notes, and examples for the most powerful packages in the ecosystem.

A personal reference collection for notable Nim libraries — annotated summaries of APIs, practical usage notes, and working examples gathered while exploring the ecosystem.

Each reference is a self-contained Markdown file covering the core API, common patterns, gotchas, and annotated code snippets. The goal is a fast-lookup companion, not a replacement for official documentation.

🇷🇺 Russian version: [README_RU.md](README_RU.md)

## Contents

### SDL3 — graphics / input

> Repository: https://github.com/nim-lang/sdl3

| Reference | Description |
|---|---|
| [Quick reference](SDL3/sdl3_nim_reference_EN.md) | Core API: init/shutdown, windows, renderer, events, input, audio, timers, filesystem |
| [Extended reference](SDL3/sdl3_nim_reference_extended_EN.md) | Same coverage plus error-handling notes and extra detail |

### Constantine — cryptography

> Repository: https://github.com/mratsim/constantine

| Reference | Description |
|---|---|
| [EIP-7594 PeerDAS](Constantine/En/eth_eip7594_peerdas_EN.md) | `eth_eip7594_peerdas.nim` — Data Availability Sampling |
| [Ethereum BLS Signatures](Constantine/En/ethereum-bls-signatures_EN.md) | `ethereum_bls_signatures.nim` — BLS12-381 signatures for the Ethereum consensus layer |
| [Ethereum EVM Precompiles](Constantine/En/ethereum_evm_precompiles_en.md) | `ethereum_evm_precompiles.nim` — hashing and crypto precompiles for the EVM |

### Russian-only references

The following references currently exist in Russian only, under [`Справочники/`](Справочники):

| Library | Reference | Repository |
|---|---|---|
| bigints | [bigints_reference_ru.md](Справочники/bigints_reference_ru.md) | https://github.com/nim-lang/bigints |
| malebolgia | [malebolgia_reference_ru.md](Справочники/malebolgia_reference_ru.md) | https://github.com/Araq/malebolgia |
| uirelays | [uirelays-guide.md](Справочники/uirelays-guide.md) (guide), [uirelays_api_reference_ru.md](Справочники/uirelays_api_reference_ru.md) (API reference) | https://github.com/nim-lang/uirelays |

Contributions and corrections are welcome via issues or pull requests.
