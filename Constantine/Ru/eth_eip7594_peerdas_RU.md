# Справочник: `eth_eip7594_peerdas.nim`

> Модуль Constantine · EIP-7594 PeerDAS — выборка данных о доступности (Data Availability Sampling)

---

## Оглавление

1. [Обзор и концепции](#1-обзор-и-концепции)
2. [Импортируемые зависимости](#2-импортируемые-зависимости)
3. [Константы](#3-константы)
4. [Типы данных](#4-типы-данных)
5. [Внутренние функции сериализации](#5-внутренние-функции-сериализации)
6. [Вспомогательные функции](#6-вспомогательные-функции)
7. [Публичный API](#7-публичный-api)
8. [Алгоритмы и оптимизации](#8-алгоритмы-и-оптимизации)
9. [Коды ошибок](#9-коды-ошибок)
10. [Полная схема потока данных](#10-полная-схема-потока-данных)
11. [Примеры использования](#11-примеры-использования)

---

## 1. Обзор и концепции

Модуль реализует уровень Ethereum-специфической сериализации для **PeerDAS** (Peer Data Availability Sampling) согласно [EIP-7594](https://eips.ethereum.org/EIPS/eip-7594).

### Ключевые понятия

| Понятие | Размер | Описание |
|---------|--------|----------|
| **Blob** (блоб) | 128 KB | 4096 элементов поля × 32 байта — исходный блок данных |
| **Cell** (ячейка) | 2 048 байт | 64 элемента поля — минимальная единица выборки |
| **Extended Blob** | 256 KB | 8192 элементов поля после расширения Рида–Соломона (2× избыточность) |
| **Cells per Blob** | 64 | 4096 ÷ 64 |
| **Cells per Extended Blob** | 128 | 8192 ÷ 64 |
| **KZG Proof** | 48 байт | Доказательство KZG в сжатом формате G1 |
| **KZG Commitment** | 48 байт | Обязательство на многочлен (тоже сжатый G1) |

### Жизненный цикл блоба в PeerDAS

```
Blob (128 KB)
  │
  ▼ blob_to_field_polynomial()
Polynomial (Лагранжева форма, bit-reversed, 4096 точек)
  │
  ├──[IFFT]──▶ Monomial form (коэффициентная форма, 4096 коэффициентов)
  │                │
  │                ├──▶ FK20: вычислить 128 KZG-доказательств
  │                │
  │                └──▶ Shift + FFT: вычислить ячейки 64–127
  │
  └──[direct copy]──▶ ячейки 0–63 (совпадают с исходным блобом!)
        │
        ▼
128 Cell[2048 byte] + 128 KZGProof[48 byte]
```

---

## 2. Импортируемые зависимости

```nim
import
  # Алгебраические структуры и кривые
  constantine/named/algebras,
  # Кривые, расширения полей, многочлены, FFT
  constantine/math/[ec_shortweierstrass, extension_fields,
                    polynomials/polynomials, polynomials/fft_fields],
  # Арифметика в конечных полях и числа Монтгомери
  constantine/math/arithmetic/[finite_fields, limbs_montgomery],
  # Ввод/вывод для BigInt и полей
  constantine/math/io/[io_bigints, io_fields],
  # Платформенные примитивы, представления массивов, аллокаторы кучи
  constantine/platforms/[primitives, views, allocs],
  # Доверенные параметры SRS (Structured Reference String) для Ethereum KZG
  constantine/commitments_setups/ethereum_kzg_srs,
  # EIP-4844 KZG (импорт с {.all.} — включая приватные символы)
  constantine/ethereum_eip4844_kzg {.all.},
  # Статус-коды сериализации и кодеки BLS12-381
  constantine/serialization/[codecs_status_codes, codecs_bls12_381, endians],
  # Внутренняя реализация PeerDAS
  constantine/data_availability_sampling/eth_peerdas,
  # Многоточечные KZG-доказательства (FK20)
  constantine/commitments/kzg_multiproofs,
  # Хеш-функции (SHA-256 для Fiat-Shamir)
  constantine/hashes

import
  # Только для компиляции: typetraits нужен для introspection типов
  std/typetraits
```

> **Примечание:** `{.all.}` при импорте `ethereum_eip4844_kzg` открывает доступ к приватным символам модуля — в частности к `blob_to_field_polynomial` и `bls_field_to_bytes`.

---

## 3. Константы

```nim
const
  # Доменный разделитель для Fiat-Shamir heuristic при пакетной верификации.
  # Ровно 16 байт ASCII — заполняет один блок SHA-256 за одно обращение.
  RANDOM_CHALLENGE_KZG_CELL_BATCH_DOMAIN* = asBytes"RCKZGCBATCH__V1_"
```

### Реэкспортируемые константы из `ethereum_kzg_srs`

| Имя | Значение | Описание |
|-----|----------|----------|
| `FIELD_ELEMENTS_PER_CELL` | 64 | Элементов поля Fr в одной ячейке |
| `CELLS_PER_EXT_BLOB` | 128 | Ячеек в расширенном блобе |
| `BYTES_PER_CELL` | 2 048 | Байт в одной ячейке (64 × 32) |
| `CELLS_PER_BLOB` | 64 | Ячеек в исходном (нерасширенном) блобе |

Эти константы реэкспортируются директивами:
```nim
export ethereum_kzg_srs.FIELD_ELEMENTS_PER_CELL
export ethereum_kzg_srs.CELLS_PER_EXT_BLOB
export ethereum_kzg_srs.BYTES_PER_CELL
export ethereum_kzg_srs.CELLS_PER_BLOB
```

---

## 4. Типы данных

### `Cell`

```nim
type Cell* = array[BYTES_PER_CELL, byte]
  ## Фундаментальная единица выборки данных.
  ## Содержит 64 элемента поля (2048 байт) в big-endian сериализации.
  ## Каждая ячейка верифицируется одним KZG-доказательством.
  ##
  ## Структура памяти:
  ##   [elem_0_byte_0 .. elem_0_byte_31 | elem_1_byte_0 .. elem_1_byte_31 | ... | elem_63_byte_31]
  ##    ◄──────── 32 байта ────────────► ◄──────── 32 байта ────────────►       ◄── 32 байта ──►
  ##   итого: 64 × 32 = 2048 байт
```

### `CellIndex`

```nim
type CellIndex* = uint64
  ## Индекс ячейки в расширенном блобе.
  ## Допустимый диапазон: [0, CELLS_PER_EXT_BLOB)  →  [0, 128)
  ## Используется в API как идентификатор позиции ячейки.
```

### `CosetEvals`

```nim
type CosetEvals* = array[FIELD_ELEMENTS_PER_CELL, Fr[BLS12_381]]
  ## Внутреннее представление данных ячейки:
  ## 64 значения многочлена, вычисленные в точках косета.
  ##
  ## Косет — это смещённое подмножество корней из единицы:
  ##   { ω_shift · ω^0,  ω_shift · ω^1, ...,  ω_shift · ω^63 }
  ## где ω — примитивный 64-й корень из единицы,
  ##       ω_shift — множитель смещения для конкретной ячейки.
  ##
  ## Fr[BLS12_381] — элемент простого поля скаляров кривой BLS12-381:
  ##   r = 52435875175126190479447740508185965837690552500527637822603658699938581184513
```

### `KZGProofBytes`

```nim
type KZGProofBytes* = array[BYTES_PER_PROOF, byte]
  ## Сериализованное KZG-доказательство: 48 байт сжатого G1-аффинного элемента.
  ##
  ## Формат: первый байт = 0x80 (бит сжатия), затем 47 байт x-координаты.
  ## Используется на границах I/O и FFI.
  ##
  ## Конвертация:
  ##   bytes → KZGProof : deserialize_g1_compressed(proof, bytes)
  ##   KZGProof → bytes : serialize_g1_compressed(bytes, proof)
  ##
  ## Внутри библиотеки используется EC_ShortW_Aff[Fp[BLS12_381], G1] (96 байт).
```

### `Coset`

```nim
type Coset* = array[FIELD_ELEMENTS_PER_CELL, Fr[BLS12_381]]
  ## Домен вычисления для одной ячейки — 64 точки (корни из единицы в косете).
  ## Аналог CosetEvals, используется как тип домена в FFT-операциях.
```

---

## 5. Внутренние функции сериализации

Эти функции не экспортируются и составляют слой конвертации `bytes ↔ Fr`.

---

### `bytesToBlsField`

```nim
func bytesToBlsField(
    dst: var Fr[BLS12_381],   # [out] Результат — элемент поля
    src: array[32, byte]      # [in]  32 байта big-endian
): CttCodecScalarStatus
```

**Назначение:** Конвертировать 32 недоверенных байта в проверенный скалярный элемент поля Fr BLS12-381.

**Алгоритм:**
1. Десериализовать байты в BigInt через `deserialize_scalar`
2. Проверить, что BigInt < r (порядок кривой)
3. Конвертировать в форму Монтгомери через `fromBig`

**Возвращаемые статусы:**

| Статус | Описание |
|--------|----------|
| `cttCodecScalar_Success` | Успех |
| `cttCodecScalar_Zero` | Успех (нулевой элемент) |
| Прочие | Ошибка: байты ≥ r |

**Пример:**
```nim
var fieldElem: Fr[BLS12_381]
var rawBytes: array[32, byte]
# ... заполнить rawBytes ...
let status = bytesToBlsField(fieldElem, rawBytes)
if status != cttCodecScalar_Success:
  # обработать ошибку — байты не являются валидным скаляром
  discard
```

---

### `cellToCosetEvals`

```nim
func cellToCosetEvals(
    evals: var array[FIELD_ELEMENTS_PER_CELL, Fr[BLS12_381]],  # [out] 64 элемента поля
    cell: openArray[byte]                                        # [in]  2048 байт ячейки
): cttEthKzgStatus
```

**Назначение:** Десериализовать ячейку (bytes) в массив элементов поля (CosetEvals).

**Структура входных данных:**
```
cell[0..31]   → evals[0]   (32 байта big-endian → Fr)
cell[32..63]  → evals[1]
...
cell[2016..2047] → evals[63]
```

**Возвращаемые значения:**
- `cttEthKzg_Success` — успех
- `cttEthKzg_ScalarLargerThanCurveOrder` — один из 32-байтовых чанков ≥ r

---

### `cosetEvalsToCell`

```nim
func cosetEvalsToCell(
    cell: var openArray[byte],                                    # [out] 2048 байт
    evals: array[FIELD_ELEMENTS_PER_CELL, Fr[BLS12_381]]        # [in]  64 элемента поля
)
```

**Назначение:** Сериализовать массив элементов поля в байты ячейки. Обратная операция к `cellToCosetEvals`.

**Алгоритм:**
```
для каждого i в [0, 63]:
  marshal(chunk[32], evals[i], bigEndian)   # из формы Монтгомери → big-endian bytes
  cell[i*32 .. i*32+31] := chunk
```

---

## 6. Вспомогательные функции

### Шаблон `?` (ранний возврат при ошибке)

```nim
template `?`(status: cttEthKzgStatus): untyped {.dirty.}
  ## Аналог оператора ? в Rust.
  ## Если status ≠ cttEthKzg_Success — немедленно вернуть status из вызывающей функции.
  ## Используется для цепочек операций без вложенных if-блоков.
```

**Пример применения:**
```nim
# Вместо:
let s1 = operationA()
if s1 != cttEthKzg_Success: return s1
let s2 = operationB()
if s2 != cttEthKzg_Success: return s2

# Используется:
?operationA()
?operationB()
```

---

### `deduplicateCommitments`

```nim
func deduplicateCommitments(
    commitmentIdx: var openArray[int],                              # [out] индекс уникального commitment для каждой ячейки
    commitments: openArray[array[BYTES_PER_COMMITMENT, byte]],     # [in]  N commitments (возможно с дублями)
    firstOccurrence: var openArray[int]                            # [out] индекс первого вхождения каждого уникального
): int  # возвращает количество уникальных commitments
```

**Назначение:** Дедупликация KZG-обязательств перед десериализацией. Работает на сырых байтах — быстрее, чем на десериализованных EC-точках.

**Сложность:** O(N × M), где N — число ячеек, M — число уникальных блобов (≤ 64 по EIP-7594).

**Сравнение с алгоритмом спецификации Ethereum:**

| Подход | Сложность | Сравнений при N=1000, M=64 |
|--------|-----------|---------------------------|
| Spec (Python `list.index`) | O(N²) | ~1 064 000 |
| Данная реализация | O(N × M) | ~32 000 |
| Ускорение | — | ~33× |

**Инварианты после вызова:**
```
commitmentIdx[i]    ∈ [0, numUnique-1]         для каждой ячейки i
firstOccurrence[j]  ∈ [0, N-1]                 индекс первого вхождения уникального j
uniqueBuffer[0..numUnique-1] содержит все различные commitments
```

**Пример:**
```nim
# Вход:  commitments = [A, A, B, B]  (N = 4)
# Выход: numUnique = 2
#        commitmentIdx     = [0, 0, 1, 1]
#        firstOccurrence   = [0, 2]
```

---

### `compute_verify_cell_kzg_proof_batch_challenge`

```nim
func compute_verify_cell_kzg_proof_batch_challenge(
    commitments:       openArray[array[BYTES_PER_COMMITMENT, byte]],  # уникальные commitments
    commitment_indices: openArray[int],                                # к какому уникальному относится каждая ячейка
    cell_indices:      openArray[CellIndex],                           # индексы ячеек
    cosets_evals:      openArray[array[FIELD_ELEMENTS_PER_CELL, Fr[BLS12_381]]],  # данные ячеек
    proofs:            openArray[KZGProofBytes]                        # доказательства
): Fr[BLS12_381]  # возвращает случайный скаляр r (Fiat-Shamir challenge)
```

**Назначение:** Вычислить Fiat-Shamir challenge `r` для пакетной верификации.

**Алгоритм (SHA-256 транскрипт):**
```
transcript.init()
transcript.update(RANDOM_CHALLENGE_KZG_CELL_BATCH_DOMAIN)   # "RCKZGCBATCH__V1_"
transcript.update(uint64(FIELD_ELEMENTS_PER_BLOB), bigEndian)
transcript.update(uint64(FIELD_ELEMENTS_PER_CELL), bigEndian)
transcript.update(uint64(numCommitments), bigEndian)
transcript.update(uint64(numCells), bigEndian)
для каждого commitment:
    transcript.update(commitment[48 bytes])
для каждой ячейки k:
    transcript.update(uint64(commitment_indices[k]), bigEndian)
    transcript.update(uint64(cell_indices[k]), bigEndian)
    для каждого eval в cosets_evals[k]:
        transcript.update(bls_field_to_bytes(eval))
    transcript.update(proofs[k])
r := Fr.fromDigest(SHA256(transcript))
```

---

### `compute_cells_impl`

```nim
func compute_cells_impl(
    ctx: ptr EthereumKZGContext,
    cells: var array[CELLS_PER_EXT_BLOB, Cell],
    poly_eval_brp: PolynomialEval[FIELD_ELEMENTS_PER_BLOB, Fr[BLS12_381], kBitReversed],
    poly_coef_nat: PolynomialCoef[FIELD_ELEMENTS_PER_BLOB, Fr[BLS12_381]]
): cttEthKzgStatus
```

**Назначение:** Основная реализация вычисления всех 128 ячеек. Принимает многочлен в двух формах и заполняет все ячейки.

**Алгоритм (оптимизация половинного FFT):**

```
Шаг 1: Ячейки 0–63  → прямое копирование из poly_eval_brp (O(1)!)
        Первая половина bit-reversed расширенного домена = исходный блоб.

Шаг 2: Ячейки 64–127 → смещение + FFT
        poly_coef_shifted := poly_coef_nat
        умножить коэффициент k на w_8192^k  (сдвиг в частотной области)
        FFT(poly_coef_shifted) → ячейки[64..127]

Шаг 3: Сериализация
        для каждой ячейки i: cosetEvalsToCell(cells[i], cells_evals[i])
```

**Сложность:**
- Ячейки 0–63: O(1) — только `copyMem`
- Ячейки 64–127: O(N log N) — один FFT размера 4096
- Итого: примерно в **2× быстрее** полного FFT(8192)

---

## 7. Публичный API

### `compute_cells`

```nim
func compute_cells*(
    ctx: ptr EthereumKZGContext,                    # Контекст с SRS и FFT-дескрипторами
    cells: var array[CELLS_PER_EXT_BLOB, Cell],     # [out] 128 ячеек по 2048 байт
    blob: Blob                                       # [in]  блоб 128 KB
): cttEthKzgStatus
```

**Назначение:** Вычислить все 128 ячеек расширенного блоба. Без вычисления KZG-доказательств.

**Алгоритм:**
1. `blob_to_field_polynomial` → `poly_eval_brp` (Лагранж, bit-reversed, куча, ~128 KB)
2. `lagrangeInterpolate` → `poly_coef_nat` (коэффициентная форма, куча, ~128 KB)
3. `compute_cells_impl` → заполнить все 128 ячеек

> Обе промежуточные аллокации выполняются в куче (`allocHeapAligned`), а не на стеке, чтобы избежать переполнения стека (128 KB — 10% от лимита стека Windows по умолчанию).

**Возвращаемые значения:** см. [Коды ошибок](#9-коды-ошибок).

**Пример:**
```nim
import constantine/eth_eip7594_peerdas

let ctx: ptr EthereumKZGContext = ...  # инициализировать контекст
var blob: Blob                          # заполнить blob данными
var cells: array[CELLS_PER_EXT_BLOB, Cell]

let status = compute_cells(ctx, cells, blob)
if status != cttEthKzg_Success:
  echo "Ошибка вычисления ячеек: ", status
```

---

### `compute_cells_and_kzg_proofs`

```nim
func compute_cells_and_kzg_proofs*(
    ctx: ptr EthereumKZGContext,
    cells: ptr UncheckedArray[Cell],          # [out] 128 ячеек
    proofs: ptr UncheckedArray[KZGProofBytes], # [out] 128 KZG-доказательств по 48 байт
    blob: Blob                                # [in]  блоб 128 KB
): cttEthKzgStatus
  {.libPrefix: prefix_eth_kzg, raises: [].}
```

**Назначение:** Вычислить все 128 ячеек **и** все 128 KZG-доказательств для блоба. Основная операция при публикации блоба.

**Алгоритм (FK20):**

```
1. Десериализация
   blob_to_field_polynomial → poly_lagrange (Лагранж, bit-reversed)

2. IFFT: Лагранж → мономиальная форма
   lagrangeInterpolate(poly_lagrange) → poly_monomial (4096 коэффициентов)

3. Вычисление ячеек
   compute_cells_impl(poly_lagrange, poly_monomial) → cells[0..127]

4. FK20: вычисление 128 доказательств
   kzg_coset_prove(poly_monomial.coefs[0..4095], SRS) → proofsAff[128]
   bit_reversal_permutation(proofsAff)       # перестановка по EIP-7594

5. Сериализация доказательств
   для каждого i: serialize_g1_compressed(proofs[i], proofsAff[i])
```

**Предусловия:**
- `ctx`, `cells`, `proofs` — не `nil`

**Ветвление по типу предвычислений SRS:**
```nim
case ctx.polyphaseSpectrumBank.kind:
of kNoPrecompute:
  # Использовать сырые точки SRS (медленнее, меньше памяти)
  kzg_coset_prove(..., ctx.polyphaseSpectrumBank.rawPoints)
of kPrecompute:
  # Использовать предвычисленный полифазный спектральный банк (быстрее)
  kzg_coset_prove(..., ctx.polyphaseSpectrumBank.precompPoints)
```

**Пример:**
```nim
var blob: Blob
var cells: array[CELLS_PER_EXT_BLOB, Cell]
var proofs: array[CELLS_PER_EXT_BLOB, KZGProofBytes]

let status = compute_cells_and_kzg_proofs(
  ctx,
  cast[ptr UncheckedArray[Cell]](addr cells[0]),
  cast[ptr UncheckedArray[KZGProofBytes]](addr proofs[0]),
  blob
)
```

---

### `recover_cells_and_kzg_proofs`

```nim
func recover_cells_and_kzg_proofs*(
    ctx: ptr EthereumKZGContext,
    recovered_cells: ptr UncheckedArray[Cell],          # [out] все 128 ячеек
    recovered_proofs: ptr UncheckedArray[KZGProofBytes], # [out] все 128 доказательств
    cell_indices: ptr UncheckedArray[CellIndex],         # [in]  индексы известных ячеек (строго возрастающие)
    cells: ptr UncheckedArray[Cell],                     # [in]  известные ячейки
    n: int                                               # [in]  количество известных ячеек
): cttEthKzgStatus
  {.libPrefix: prefix_eth_kzg, raises: [].}
```

**Назначение:** Восстановить все 128 ячеек и 128 доказательств из ≥ 64 известных ячеек (порог 50%).

**Предусловия:**
- `n >= CELLS_PER_EXT_BLOB div 2` (т.е. ≥ 64 ячеек)
- `n <= CELLS_PER_EXT_BLOB` (не более 128)
- `cell_indices` строго возрастающие: `cell_indices[i-1] < cell_indices[i]`
- Все индексы в диапазоне `[0, CELLS_PER_EXT_BLOB)`
- Все указатели не `nil`

**Алгоритм:**

```
1. Валидация:   проверить n, уникальность и порядок индексов

2. Десериализация:
   для каждой из n ячеек: cellToCosetEvals(cells[i]) → cosets_evals[i]

3. Восстановление коэффициентного многочлена:
   recoverPolynomialCoeff(
     domain=8192 корней из единицы,
     coset_shift=SCALE_FACTOR=5
   ) → poly_coeff[8192]

4. FFT(poly_coeff) → cells_evals[128][64]
   (размер 8192: 128 ячеек × 64 элемента)

5. Сериализация ячеек:
   для каждой i: cosetEvalsToCell(recovered_cells[i], cells_evals[i])

6. FK20: вычисление доказательств
   kzg_coset_prove(poly_coeff.coefs[0..4095]) → proofsAff[128]
   bit_reversal_permutation(proofsAff)
   serialize_g1_compressed(recovered_proofs[i], proofsAff[i])
```

**Ошибки:**
- `cttEthKzg_InputsLengthsMismatch` — n вне диапазона, nil-указатели, невалидные индексы
- `cttEthKzg_CellIndicesNotAscending` — индексы не строго возрастают
- `cttEthKzg_ScalarLargerThanCurveOrder` — байты ячейки содержат недопустимый скаляр

---

### `verify_cell_kzg_proof_batch`

```nim
func verify_cell_kzg_proof_batch*(
    ctx: ptr EthereumKZGContext,
    commitments_bytes: ptr UncheckedArray[array[BYTES_PER_COMMITMENT, byte]],  # [in] N обязательств
    cell_indices: ptr UncheckedArray[CellIndex],                               # [in] N индексов ячеек
    cells: ptr UncheckedArray[Cell],                                           # [in] N ячеек
    proofs_bytes: ptr UncheckedArray[KZGProofBytes],                          # [in] N доказательств
    n: int,                                                                    # [in] размер пакета
    secureRandomBytes: array[32, byte]                                         # [in] 32 случайных байта
): cttEthKzgStatus
  {.libPrefix: prefix_eth_kzg, raises: [].}
```

**Назначение:** Пакетная верификация N пар (ячейка, доказательство, обязательство). Реализует [универсальное уравнение верификации DAS](https://ethresear.ch/t/a-universal-verification-equation-for-data-availability-sampling/13240).

**Алгоритм:**

```
Граничные случаи:
  n < 0     → InputsLengthsMismatch
  n == 0    → Success (тривиально верно)

1. Валидация индексов: cell_indices[i] < CELLS_PER_EXT_BLOB

2. Дедупликация обязательств (на сырых байтах, быстро):
   deduplicateCommitments → commitmentIdx[N], firstOccurrence[M], M

3. Десериализация (только M уникальных обязательств):
   deserialize_g1_compressed(uniqueCommitments[j], commitments_bytes[firstOccurrence[j]])

4. Десериализация N ячеек и N доказательств:
   cellToCosetEvals(cosets_evals[i], cells[i])
   deserialize_g1_compressed(proofs[i], proofs_bytes[i])

5. Fiat-Shamir challenge r:
   если secureRandomBytes не нулевые → r из них (для тестов)
   иначе → compute_verify_cell_kzg_proof_batch_challenge(...)

6. Степени r: [r^1, r^2, ..., r^N]
   computePowers(r, n, skipOne=true)

7. Пакетная верификация:
   kzg_coset_verify_batch(
     uniqueCommitments, commitmentIdx,
     proofs, cosets_evals, evalsCols,
     domain=fft_desc_ext,
     linearIndepRandNumbers=rPowers,
     powers_of_tau=SRS,
     tau_pow_L_g2=SRS_G2[64],
     N=FIELD_ELEMENTS_PER_EXT_BLOB
   ) → bool
```

**Секция `secureRandomBytes`:**
- Если передать `[0; 32]` (нули) — библиотека вычислит `r` через Fiat-Shamir (стандартный режим)
- Ненулевое значение используется как источник случайности напрямую (для тестирования)

**Возвращаемые значения:**
- `cttEthKzg_Success` — все доказательства верны
- `cttEthKzg_VerificationFailure` — хотя бы одно доказательство неверно
- Прочие коды — ошибки входных данных

---

## 8. Алгоритмы и оптимизации

### Оптимизация половинного FFT в `compute_cells_impl`

**Математическое обоснование:**

Расширенный блоб имеет 8192 элемента поля. Наивное вычисление потребовало бы FFT размером 8192. Однако:

- Корни из единицы размера 8192 после bit-reversal: чётные индексы → первые 4096, нечётные → вторые 4096
- Чётные корни: `w_8192^(2k) = w_4096^k` — это те же точки, что и исходный блоб!
- Значит: **первые 64 ячейки = исходный блоб** (O(1), только copyMem)

Для вторых 64 ячеек нечётные корни `w_8192^(2k+1) = w_8192 · w_4096^k`:
- Умножить коэффициент `c_k` на `w_8192^k` (сдвиг коэффициентного представления)
- Применить FFT(4096) к сдвинутым коэффициентам

**Итоговая сложность:**
- Полный FFT(8192): ≈ 8192 × 13 = 106 496 умножений
- Половинный FFT: ≈ 4096 × 12 × 2 = 98 304 умножений (IFFT + FFT)
- Плюс O(4096) для сдвига
- **Ускорение ≈ 2×** по сравнению с полным FFT, **≈ 370×** по сравнению с наивным O(N²)

### Дедупликация обязательств: байтовое сравнение

Работа с 48-байтовыми сжатыми точками G1 вместо десериализованных EC-точек:
- Размер: 48 байт vs ~96 байт (более кэш-эффективно)
- Криптографическое свойство: равномерное распределение байтов → вероятность совпадения первого байта 1/256 → ранний выход из `memcmp` почти всегда
- Нет накладных расходов на десериализацию дубликатов

### Управление памятью

Все крупные буферы выделяются в куче с выравниванием 64 байта:

```nim
# Пример паттерна аллокации (внутри функций API):
let buffer = allocHeapAligned(SomeType, alignment = 64)
defer: freeHeapAligned(buffer)  # гарантированное освобождение при любом выходе
```

Причины:
- 128 KB на стеке = 10% лимита стека Windows (1 MB по умолчанию)
- Выравнивание 64 = размер кэш-линии → оптимальная работа с SIMD/AVX

### Прагмы безопасности

```nim
{.push raises: [].}   # Запрет на любые исключения (обязательно для крипто-кода)
{.push checks: off.}  # Отключение проверок переполнения int и выхода за границы массива
```

---

## 9. Коды ошибок

Тип `cttEthKzgStatus` (из `constantine/ethereum_eip4844_kzg`):

| Код | Значение |
|-----|----------|
| `cttEthKzg_Success` | Операция выполнена успешно |
| `cttEthKzg_VerificationFailure` | Верификация не пройдена (неверное доказательство) |
| `cttEthKzg_InputsLengthsMismatch` | Несоответствие длин входных данных или nil-указатели |
| `cttEthKzg_ScalarLargerThanCurveOrder` | Байты содержат скаляр ≥ r |
| `cttEthKzg_CellIndicesNotAscending` | Индексы ячеек не строго возрастают |

Статус-коды сериализации `CttCodecScalarStatus` (из `codecs_status_codes`):

| Код | Значение |
|-----|----------|
| `cttCodecScalar_Success` | Успех |
| `cttCodecScalar_Zero` | Успех (нулевой скаляр) |
| Прочие | Ошибка десериализации |

---

## 10. Полная схема потока данных

```
                         ┌────────────────────────────────────┐
                         │            Blob (128 KB)            │
                         │     array[4096, array[32, byte]]    │
                         └───────────────┬────────────────────┘
                                         │ blob_to_field_polynomial()
                                         ▼
                    ┌────────────────────────────────────────────┐
                    │  PolynomialEval[4096, Fr, kBitReversed]    │
                    │  (Лагранжева форма, bit-reversed)           │
                    └──────────┬─────────────────────────────────┘
                               │
               ┌───────────────┴──────────────────┐
               │ lagrangeInterpolate (IFFT)         │
               ▼                                    │
   ┌─────────────────────────────┐                  │ (прямое копирование)
   │  PolynomialCoef[4096, Fr]   │                  │
   │  (мономиальная форма)       │                  ▼
   └─────┬──────────────┬────────┘     ячейки 0–63 = blob
         │              │
    shift+FFT        FK20 multiproof
         │              │
         ▼              ▼
   ячейки 64–127    128 × G1Aff
                         │
                    bit_reversal_permutation
                         │
                    serialize_g1_compressed
                         │
                         ▼
                  128 × KZGProofBytes[48]
```

---

## 11. Примеры использования

### Полный цикл: публикация блоба

```nim
import
  constantine/commitments_setups/ethereum_kzg_srs,
  constantine/eth_eip7594_peerdas

# Инициализация контекста (один раз, дорогостоящая операция)
var ctx: EthereumKZGContext
ctx.trusted_setup_load(trustedSetupFile, kReferenceCKzg)

# Данные блоба (128 KB)
var blob: Blob
# ... заполнить blob ...

# Вычислить все ячейки и доказательства
var cells: array[CELLS_PER_EXT_BLOB, Cell]
var proofs: array[CELLS_PER_EXT_BLOB, KZGProofBytes]

let status = compute_cells_and_kzg_proofs(
  addr ctx,
  cast[ptr UncheckedArray[Cell]](addr cells[0]),
  cast[ptr UncheckedArray[KZGProofBytes]](addr proofs[0]),
  blob
)
doAssert status == cttEthKzg_Success
```

### Верификация пакета ячеек

```nim
# n = количество пар (commitment, cell_index, cell, proof) для проверки
let n = 256

var secureRandom: array[32, byte]
# ... заполнить secureRandom случайными байтами ...

let status = verify_cell_kzg_proof_batch(
  addr ctx,
  cast[ptr UncheckedArray[array[BYTES_PER_COMMITMENT, byte]]](addr commitments[0]),
  cast[ptr UncheckedArray[CellIndex]](addr cellIndices[0]),
  cast[ptr UncheckedArray[Cell]](addr cells[0]),
  cast[ptr UncheckedArray[KZGProofBytes]](addr proofs[0]),
  n,
  secureRandom
)

if status == cttEthKzg_VerificationFailure:
  echo "Верификация провалена: обнаружен невалидный сэмпл"
elif status != cttEthKzg_Success:
  echo "Ошибка входных данных: ", status
```

### Восстановление блоба из частичных данных

```nim
# Предположим, получены ячейки с индексами 0..63 (ровно 50%)
var
  knownIndices: array[64, CellIndex]
  knownCells: array[64, Cell]

for i in 0 ..< 64:
  knownIndices[i] = CellIndex(i)
  # ... заполнить knownCells[i] ...

var
  recoveredCells: array[CELLS_PER_EXT_BLOB, Cell]
  recoveredProofs: array[CELLS_PER_EXT_BLOB, KZGProofBytes]

let status = recover_cells_and_kzg_proofs(
  addr ctx,
  cast[ptr UncheckedArray[Cell]](addr recoveredCells[0]),
  cast[ptr UncheckedArray[KZGProofBytes]](addr recoveredProofs[0]),
  cast[ptr UncheckedArray[CellIndex]](addr knownIndices[0]),
  cast[ptr UncheckedArray[Cell]](addr knownCells[0]),
  64  # количество известных ячеек
)
doAssert status == cttEthKzg_Success
# recoveredCells[0..127] и recoveredProofs[0..127] теперь содержат полные данные
```

---

## Ссылки

- [EIP-7594](https://eips.ethereum.org/EIPS/eip-7594) — спецификация PeerDAS
- [Спецификация Ethereum (polynomial-commitments-sampling)](https://github.com/ethereum/consensus-specs/blob/v1.4.0-beta.1/specs/fulu/polynomial-commitments-sampling.md)
- [FK20 Paper](https://eprint.iacr.org/2023/033) — алгоритм многоточечных KZG-доказательств
- [Универсальное уравнение верификации DAS](https://ethresear.ch/t/a-universal-verification-equation-for-data-availability-sampling/13240)
- [Constantine: библиотека криптографии](https://github.com/mratsim/constantine)
