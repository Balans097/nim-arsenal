# 📚 nim-arsenal

*Подборка справочников по библиотекам Nim: аннотированные сводки API, заметки по использованию и примеры для наиболее мощных пакетов экосистемы.*

---

## О проекте

Личная коллекция справочников по примечательным библиотекам Nim — аннотированные сводки API, практические заметки по использованию и рабочие примеры, собранные в процессе изучения экосистемы.

Каждый справочник представляет собой самодостаточный Markdown-файл, охватывающий:

- **Основной API** — ключевые типы, процедуры и макросы
- **Типичные паттерны** — идиоматичное использование и практические примеры
- **Подводные камни** — неочевидное поведение, заметки по версиям, известные особенности
- **Аннотированные примеры** — рабочий код с построчными комментариями

Цель — **быстрый справочник под рукой**, а не замена официальной документации.

🇬🇧 English version: [README.md](README.md)

---

## Содержимое

### SDL3 — графика / ввод

> Репозиторий: https://github.com/nim-lang/sdl3

| Справочник | Описание |
|---|---|
| [Краткий справочник](SDL3/sdl3_nim_reference_RU.md) | Основной API: инициализация, окна, рендерер, события, ввод, аудио, таймеры, файловая система |
| [Расширенный справочник](SDL3/sdl3_nim_reference_extended_RU.md) | То же самое плюс заметки по обработке ошибок и дополнительные детали |

### Constantine — криптография

> Репозиторий: https://github.com/mratsim/constantine

| Справочник | Описание |
|---|---|
| [EIP-7594 PeerDAS](Constantine/Ru/eth_eip7594_peerdas_RU.md) | `eth_eip7594_peerdas.nim` — выборка доступности данных (Data Availability Sampling) |
| [Ethereum BLS Signatures](Constantine/Ru/ethereum-bls-signatures_RU.md) | `ethereum_bls_signatures.nim` — подписи BLS12-381 для консенсус-уровня Ethereum |
| [Ethereum EVM Precompiles](Constantine/Ru/ethereum_evm_precompiles_ru.md) | `ethereum_evm_precompiles.nim` — криптографические прекомпилированные контракты EVM |

### Справочники, доступные только на русском

Следующие справочники пока существуют только на русском языке, в папке [`Справочники/`](Справочники):

| Библиотека | Справочник | Репозиторий |
|---|---|---|
| bigints | [bigints_reference_ru.md](Справочники/bigints_reference_ru.md) | https://github.com/nim-lang/bigints |
| malebolgia | [malebolgia_reference_ru.md](Справочники/malebolgia_reference_ru.md) | https://github.com/Araq/malebolgia |
| uirelays | [uirelays-guide.md](Справочники/uirelays-guide.md) (руководство), [uirelays_api_reference_ru.md](Справочники/uirelays_api_reference_ru.md) (справочник по API) | https://github.com/nim-lang/uirelays |

---

## Участие в проекте

Исправления и дополнения приветствуются через **issues** или **pull requests**.
