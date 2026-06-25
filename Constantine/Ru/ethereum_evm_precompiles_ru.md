# Справочник: `ethereum_evm_precompiles` — Constantine

> **Библиотека:** [mratsim/constantine](https://github.com/mratsim/constantine)  
> **Файл источника:** `constantine/ethereum_evm_precompiles.nim`  
> **Язык:** Nim  
> **Лицензия:** MIT / Apache 2.0

---

## Содержание

1. [Назначение модуля](#назначение-модуля)
2. [Импорт и зависимости](#импорт-и-зависимости)
3. [Тип статуса: `CttEVMStatus`](#тип-статуса-cttevmstatus)
4. [Хэш-функции и вспомогательная криптография](#хэш-функции-и-вспомогательная-криптография)
   - [eth_evm_sha256](#eth_evm_sha256)
   - [eth_evm_ripemd160](#eth_evm_ripemd160)
5. [Модульное возведение в степень](#модульное-возведение-в-степень)
   - [eth_evm_modexp_result_size](#eth_evm_modexp_result_size)
   - [eth_evm_modexp](#eth_evm_modexp)
6. [Кривая BN254 (alt_bn128)](#кривая-bn254-altbn128)
   - [eth_evm_bn254_g1add](#eth_evm_bn254_g1add)
   - [eth_evm_bn254_g1mul](#eth_evm_bn254_g1mul)
   - [eth_evm_bn254_ecpairingcheck](#eth_evm_bn254_ecpairingcheck)
7. [Кривая BLS12-381 (EIP-2537)](#кривая-bls12-381-eip-2537)
   - [eth_evm_bls12381_g1add](#eth_evm_bls12381_g1add)
   - [eth_evm_bls12381_g2add](#eth_evm_bls12381_g2add)
   - [eth_evm_bls12381_g1mul](#eth_evm_bls12381_g1mul)
   - [eth_evm_bls12381_g2mul](#eth_evm_bls12381_g2mul)
   - [eth_evm_bls12381_g1msm](#eth_evm_bls12381_g1msm)
   - [eth_evm_bls12381_g2msm](#eth_evm_bls12381_g2msm)
   - [eth_evm_bls12381_pairingcheck](#eth_evm_bls12381_pairingcheck)
   - [eth_evm_bls12381_map_fp_to_g1](#eth_evm_bls12381_map_fp_to_g1)
   - [eth_evm_bls12381_map_fp2_to_g2](#eth_evm_bls12381_map_fp2_to_g2)
8. [KZG и протоколы EIP-4844](#kzg-и-протоколы-eip-4844)
   - [eth_evm_kzg_point_evaluation](#eth_evm_kzg_point_evaluation)
9. [ECDSA: восстановление публичного ключа](#ecdsa-восстановление-публичного-ключа)
   - [eth_evm_ecrecover](#eth_evm_ecrecover)
10. [Реэкспортируемые типы и API](#реэкспортируемые-типы-и-api)
11. [Внутренние вспомогательные функции](#внутренние-вспомогательные-функции)
12. [Таблица всех функций](#таблица-всех-функций)
13. [Общие замечания по безопасности](#общие-замечания-по-безопасности)

---

## Назначение модуля

Этот модуль реализует **предкомпилированные контракты Ethereum EVM** на языке Nim с использованием криптографической библиотеки Constantine. Предкомпилированные контракты — это особые адреса в EVM, выполняющие вычислительно тяжёлые операции (хэширование, арифметика эллиптических кривых, спаривания) напрямую в нативном коде, а не через байткод EVM.

Модуль реализует:
- хэш-функции SHA-256 и RIPEMD-160 (адреса 2, 3 EVM);
- модульное возведение в степень — MODEXP (адрес 5, EIP-198);
- операции на кривой BN254/alt_bn128: ECADD, ECMUL, ECPAIRING (адреса 6, 7, 8, EIP-196/197);
- операции на кривой BLS12-381 (EIP-2537): G1/G2 сложение, умножение, MSM, спаривание, маппинг;
- верификацию KZG-доказательства для EIP-4844 (прото-данкшардинг);
- восстановление публичного ключа ECDSA — ECRecover (адрес 1).

**Все функции объявлены с `{.push raises: [].}`** — исключения не генерируются. Ошибки возвращаются через код статуса `CttEVMStatus`.

---

## Импорт и зависимости

```nim
import
  ./hashes,                                        # SHA-256, RIPEMD-160, Keccak
  ./platforms/abstractions,                        # Низкоуровневые абстракции платформы
  ./serialization/io_limbs,                        # Сериализация лимбов BigInt
  constantine/named/algebras,                      # Именованные алгебры: BN254_Snarks, BLS12_381, Secp256k1
  ./math/[arithmetic, extension_fields],           # Арифметика полей Fp, Fr, Fp2, Fp12
  ./math/arithmetic/limbs_montgomery,              # Преобразование Монтгомери (getMont)
  ./math/ec_shortweierstrass,                      # Арифметика точек кривых Вейерштрасса
  ./math/elliptic/ec_multi_scalar_mul,             # Мультискалярное умножение (MSM)
  ./math/pairings/[pairings_generic,               # Общая реализация билинейного спаривания
                   miller_accumulators],            # Аккумулятор алгоритма Миллера
  ./named/zoo_subgroups,                           # Проверка принадлежности подгруппе
  ./math/io/[io_bigints, io_fields],               # Ввод/вывод BigInt и элементов поля
  ./math_arbitrary_precision/arithmetic/bigints_views, # Представления BigInt произвольной точности
  ./hash_to_curve/hash_to_curve,                   # Хэширование на кривую (hash-to-curve)
  ./ethereum_eip4844_kzg,                          # KZG для EIP-4844
  ./ethereum_ecdsa_signatures                      # ECDSA для ECRecover
```

---

## Тип статуса: `CttEVMStatus`

```nim
type
  CttEVMStatus* = enum
    cttEVM_Success               # Операция выполнена успешно
    cttEVM_InvalidInputSize      # Размер входного буфера неверен
    cttEVM_InvalidOutputSize     # Размер выходного буфера неверен
    cttEVM_IntLargerThanModulus  # Целое число >= модуль поля (не является элементом поля)
    cttEVM_PointNotOnCurve       # Точка не лежит на эллиптической кривой
    cttEVM_PointNotInSubgroup    # Точка не принадлежит нужной подгруппе простого порядка
    cttEVM_VerificationFailure   # Криптографическая проверка не прошла
    cttEVM_MalformedSignature    # Подпись имеет неверный формат
```

**Каждая публичная функция возвращает `CttEVMStatus`.** Вызывающий код обязан проверять возвращаемое значение перед использованием результата.

```nim
# Пример проверки статуса
var output: array[32, byte]
let data = "hello world"

let status = eth_evm_sha256(output, cast[openArray[byte]](data))
if status != cttEVM_Success:
  echo "Ошибка: ", status
else:
  echo "SHA-256 вычислен успешно"
```

---

## Хэш-функции и вспомогательная криптография

### `eth_evm_sha256`

**Адрес EVM:** 2  
**Сигнатура:**
```nim
func eth_evm_sha256*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 32 байта
  inputs: openArray[byte]        # Произвольные входные данные для хэширования
): CttEVMStatus
```

**Описание:** Вычисляет SHA-256 хэш входных данных.

**Требования к буферам:**

| Буфер    | Размер       | Описание            |
|----------|--------------|---------------------|
| `r`      | ровно 32 байт | Выходной дайджест  |
| `inputs` | произвольный  | Входное сообщение  |

**Возможные статусы:**
- `cttEVM_Success` — хэш успешно вычислен
- `cttEVM_InvalidOutputSize` — `r` не равен 32 байтам

**Пример использования:**
```nim
import constantine/ethereum_evm_precompiles

# Объявляем буфер для результата (32 байта = 256 бит)
var digest: array[32, byte]

# Входное сообщение в байтовом представлении
let message: seq[byte] = @[0x61'u8, 0x62, 0x63]  # "abc"

# Вызов: функция первой, все аргументы в скобках
let status = eth_evm_sha256(digest, message)

if status == cttEVM_Success:
  echo "Дайджест SHA-256 вычислен"
```

---

### `eth_evm_ripemd160`

**Адрес EVM:** 3  
**Сигнатура:**
```nim
func eth_evm_ripemd160*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 32 байта
  inputs: openArray[byte]        # Произвольные входные данные
): CttEVMStatus
```

**Описание:** Вычисляет RIPEMD-160 хэш входных данных. Результат — 20 байт дайджеста, **выровненные вправо** в 32-байтовом буфере: первые 12 байт равны нулю, последние 20 байт содержат хэш.

**Схема выходного буфера:**
```
[00 00 00 00 00 00 00 00 00 00 00 00 | XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX]
 ← 12 нулевых байт (padding)        → ← 20 байт хэша RIPEMD-160                               →
```

**Требования к буферам:**

| Буфер    | Размер        | Описание            |
|----------|---------------|---------------------|
| `r`      | ровно 32 байта | Выходной дайджест (12 + 20 байт) |
| `inputs` | произвольный  | Входное сообщение  |

**Возможные статусы:**
- `cttEVM_Success`
- `cttEVM_InvalidOutputSize`

---

## Модульное возведение в степень

### `eth_evm_modexp_result_size`

**Сигнатура:**
```nim
func eth_evm_modexp_result_size*(
  size:   var uint64,          # Выход: требуемый размер буфера для результата (в байтах)
  inputs: openArray[byte]      # Входные данные в формате MODEXP
): CttEVMStatus {.noInline, tags:[Alloca, Vartime].}
```

**Описание:** Вспомогательная функция. Вычисляет требуемый размер буфера для результата `eth_evm_modexp` **до** его вызова. Размер определяется как `modulusLen` — длина модуля в байтах, извлечённая из первых 96 байт входных данных.

**Формат первых 96 байт `inputs`:**
```
inputs[0..31]  — baseLen    (32 байта, big-endian uint256): длина основания
inputs[32..63] — exponentLen (32 байта, big-endian uint256): длина показателя
inputs[64..95] — modulusLen  (32 байта, big-endian uint256): длина модуля → это и есть требуемый размер
```

**Пример использования:**
```nim
var resultSize: uint64
let inputData: seq[byte] = buildModexpInput(base, exponent, modulus)

# Шаг 1: узнать нужный размер буфера
let sizeStatus = eth_evm_modexp_result_size(resultSize, inputData)
if sizeStatus != cttEVM_Success:
  echo "Ошибка размера: ", sizeStatus
  return

# Шаг 2: выделить буфер нужного размера
var resultBuf = newSeq[byte](resultSize)

# Шаг 3: выполнить вычисление
let status = eth_evm_modexp(resultBuf, inputData)
```

---

### `eth_evm_modexp`

**Адрес EVM:** 5 (EIP-198)  
**Сигнатура:**
```nim
func eth_evm_modexp*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО размером modulusLen байт
  inputs: openArray[byte]        # Закодированные входные данные
): CttEVMStatus {.noInline, tags:[Alloca, Vartime].}
```

**Описание:** Вычисляет `base ^ exponent mod modulus`. Реализует EIP-198 Yellow Paper Appendix E.

**Полный формат входных данных `inputs`:**
```
inputs[0..31]                         — baseLen     (32 байта, big-endian uint256)
inputs[32..63]                        — exponentLen (32 байта, big-endian uint256)
inputs[64..95]                        — modulusLen  (32 байта, big-endian uint256)
inputs[96 .. 96+baseLen-1]            — base        (baseLen байт, big-endian)
inputs[96+baseLen .. ... +expLen-1]   — exponent    (exponentLen байт, big-endian)
inputs[... .. ...+modulusLen-1]       — modulus     (modulusLen байт, big-endian)
```

> Если входные данные короче ожидаемого, они неявно дополняются нулями справа.

**Специальные случаи (возвращает нули):**
- `modulusLen == 0` → результат пустой
- `exponentLen == 0` → результат = `1` (x⁰ = 1, 0⁰ = 1)
- `baseLen == 0` → результат = `0`
- все данные после длин равны нулю → модуль равен нулю → результат = `0`

**Требования к буферам:**

| Буфер    | Размер                         |
|----------|-------------------------------|
| `r`      | ровно `modulusLen` байт        |
| `inputs` | произвольный (с padding нулями)|

**Возможные статусы:**
- `cttEVM_Success`
- `cttEVM_InvalidInputSize` — длины слишком велики для адресного пространства CPU
- `cttEVM_InvalidOutputSize` — `r.len != modulusLen`

> **Замечание по безопасности:** функция выделяет память на стеке объёмом до `(16+1) * modulusLen` байт. EVM должен ограничивать стоимость газа для предотвращения переполнения стека при больших входных данных.

---

## Кривая BN254 (alt_bn128)

Кривая BN254 (также известна как alt_bn128 или bn256 в Ethereum) — это скрученная кривая Бейт-Нарайяна, используемая в Ethereum с момента Byzantium hard fork. Параметры: 254-битный порядок группы.

**Кодирование координат BN254:** каждая координата — 32-байтовое целое big-endian в диапазоне `[0, p)`, где `p` — простой модуль поля.

### `eth_evm_bn254_g1add`

**Адрес EVM:** 6 (EIP-196)  
**Сигнатура:**
```nim
func eth_evm_bn254_g1add*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 64 байта
  inputs: openArray[byte]        # Входные данные: [Px(32) | Py(32) | Qx(32) | Qy(32)]
): CttEVMStatus
```

**Описание:** Сложение двух точек на кривой BN254 G1: `R = P + Q`.

**Формат входных данных:**
```
inputs[0..31]   — Px: координата X точки P (32 байта, big-endian)
inputs[32..63]  — Py: координата Y точки P (32 байта, big-endian)
inputs[64..95]  — Qx: координата X точки Q (32 байта, big-endian)
inputs[96..127] — Qy: координата Y точки Q (32 байта, big-endian)
```
Итого: 128 байт. Если `inputs.len < 128`, недостающие байты считаются нулями. Если `inputs.len > 128`, лишние байты игнорируются.

**Формат выходных данных:**
```
r[0..31]  — Rx: координата X результата R (32 байта, big-endian)
r[32..63] — Ry: координата Y результата R (32 байта, big-endian)
```

> **Подгруппа:** для BN254 G1 кофактор равен 1, поэтому проверка принадлежности подгруппе не выполняется.

> **Точка на бесконечности** кодируется как `(0, 0)`.

**Возможные статусы:**
- `cttEVM_Success`
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`
- `cttEVM_PointNotOnCurve`

**Пример:**
```nim
# Входные данные: точки P и Q в конкатенированном виде
var inputBuf: array[128, byte]
# ... заполнить координаты P и Q ...

var resultBuf: array[64, byte]
let status = eth_evm_bn254_g1add(resultBuf, inputBuf)
if status == cttEVM_Success:
  # resultBuf содержит координаты R = P + Q
  echo "Сложение точек выполнено"
```

---

### `eth_evm_bn254_g1mul`

**Адрес EVM:** 7 (EIP-196)  
**Сигнатура:**
```nim
func eth_evm_bn254_g1mul*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 64 байта
  inputs: openArray[byte]        # Входные данные: [Px(32) | Py(32) | s(32)]
): CttEVMStatus
```

**Описание:** Скалярное умножение точки на кривой BN254 G1: `R = [s]P`.

**Формат входных данных:**
```
inputs[0..31]  — Px: координата X точки P (32 байта, big-endian)
inputs[32..63] — Py: координата Y точки P (32 байта, big-endian)
inputs[64..95] — s:  скаляр в диапазоне [0, 2²⁵⁶) (32 байта, big-endian)
```
Итого: 96 байт. Дополнение нулями при нехватке, усечение при излишке.

> **Оптимизация:** скаляр `s` может быть больше порядка группы `r`. Реализация выполняет редукцию `s mod r` и использует оконный метод с эндоморфизмом (на 31.5% быстрее чистого оконного умножения) через низкоуровневое преобразование Монтгомери (`getMont`).

**Возможные статусы:**
- `cttEVM_Success`
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`
- `cttEVM_PointNotOnCurve`

---

### `eth_evm_bn254_ecpairingcheck`

**Адрес EVM:** 8 (EIP-197, EIP-1108)  
**Сигнатура:**
```nim
func eth_evm_bn254_ecpairingcheck*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 32 байта
  inputs: openArray[byte]        # Массив пар точек [(P0,Q0), (P1,Q1), ..., (Pk,Qk)]
): CttEVMStatus
```

**Описание:** Проверка равенства билинейного спаривания:
```
e(P₀, Q₀) · e(P₁, Q₁) · ... · e(Pₖ, Qₖ) == 1
```
Возвращает `1` (как uint256 big-endian) если равенство выполнено, иначе `0`.

**Формат одной пары (192 байта):**
```
[Px(32) | Py(32) | Qx_im(32) | Qx_re(32) | Qy_im(32) | Qy_re(32)]
```

> **Важно:** в соответствии с EIP-197 координаты Fp2 закодированы **в обратном порядке**: `(a, b)` означает `a·𝑖 + b` вместо стандартного `a + 𝑖·b`. Это отличает BN254 от BLS12-381.

**Входные данные:** `N` пар по 192 байта каждая. `inputs.len` должен делиться на 192.

> **Пустой ввод:** по спецификации пустой входной массив является допустимым и возвращает `1`.

**Алгоритм:**
1. Парсинг и валидация каждой пары (Pᵢ, Qᵢ).
2. Накопление через `MillerAccumulator` (алгоритм Миллера).
3. Финальная экспонента.
4. Сравнение с единицей в GT (группа цели спаривания Fp12).

**Возможные статусы:**
- `cttEVM_Success`
- `cttEVM_InvalidInputSize`
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`
- `cttEVM_PointNotOnCurve`
- `cttEVM_PointNotInSubgroup`

---

## Кривая BLS12-381 (EIP-2537)

BLS12-381 — 381-битная кривая Бейт-Любашевского-Скотта с максимальным уровнем встроенности 12. Используется в Ethereum 2.0 (Beacon Chain) для BLS-подписей. Имеет две подгруппы:

- **G1**: точки над базовым полем `Fp` (381-битный простой модуль `p`), каждая координата кодируется в 64 байтах (48 значащих байт + 16 нулевых байт старшего разряда).
- **G2**: точки над расширением поля `Fp2 = Fp[√-1]`, каждая координата — пара `(c0, c1)` элементов Fp, кодируется в 128 байтах.

**Кодирование координат BLS12-381 (EIP-2537):**
```
64-байтовый блок = [00...00 (16 байт) | координата (48 байт, big-endian)]
                    ← старшие байты → ← значащие данные →
```
Первые 16 байт **обязательно** должны быть нулевыми.

### `eth_evm_bls12381_g1add`

**EIP-2537**  
**Сигнатура:**
```nim
func eth_evm_bls12381_g1add*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 128 байт
  inputs: openArray[byte]        # ОБЯЗАТЕЛЬНО 256 байт: [Px(64) | Py(64) | Qx(64) | Qy(64)]
): CttEVMStatus
```

**Описание:** Сложение двух точек G1 кривой BLS12-381: `R = P + Q`.

**Формат входных данных (256 байт):**
```
inputs[0..63]    — Px (64 байта: 16 нулей + 48 байт координаты)
inputs[64..127]  — Py (64 байта)
inputs[128..191] — Qx (64 байта)
inputs[192..255] — Qy (64 байта)
```

**Формат выходных данных (128 байт):**
```
r[0..63]   — Rx (64 байта)
r[64..127] — Ry (64 байта)
```

> **Примечание:** для операции сложения проверка принадлежности подгруппе **не выполняется** (по спецификации). Используются формулы Якоби (vartime), так как полные проективные формулы могут давать некорректный результат вне подгруппы.

**Возможные статусы:**
- `cttEVM_Success`
- `cttEVM_InvalidInputSize`
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`
- `cttEVM_PointNotOnCurve`

---

### `eth_evm_bls12381_g2add`

**EIP-2537**  
**Сигнатура:**
```nim
func eth_evm_bls12381_g2add*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 256 байт
  inputs: openArray[byte]        # ОБЯЗАТЕЛЬНО 512 байт
): CttEVMStatus
```

**Описание:** Сложение двух точек G2 кривой BLS12-381: `R = P + Q`. Точки G2 имеют координаты в поле `Fp2 = Fp[𝑖]`, где `𝑖 = √-1`. Каждая координата `(a + 𝑖·b)` кодируется двумя 64-байтовыми блоками.

**Формат входных данных (512 байт):**
```
inputs[0..63]    — Px.c0 = Re(Px) — вещественная часть X координаты P
inputs[64..127]  — Px.c1 = Im(Px) — мнимая часть X координаты P
inputs[128..191] — Py.c0 = Re(Py)
inputs[192..255] — Py.c1 = Im(Py)
inputs[256..319] — Qx.c0
inputs[320..383] — Qx.c1
inputs[384..447] — Qy.c0
inputs[448..511] — Qy.c1
```

**Формат выходных данных (256 байт):** аналогично первой точке входа.

> Проверка подгруппы не выполняется (как и для G1ADD).

---

### `eth_evm_bls12381_g1mul`

**EIP-2537**  
**Сигнатура:**
```nim
func eth_evm_bls12381_g1mul*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 128 байт
  inputs: openArray[byte]        # ОБЯЗАТЕЛЬНО 160 байт: [Px(64) | Py(64) | s(32)]
): CttEVMStatus
```

**Описание:** Скалярное умножение точки G1: `R = [s]P`.

**Формат входных данных (160 байт):**
```
inputs[0..63]    — Px (64 байта)
inputs[64..127]  — Py (64 байта)
inputs[128..159] — s:  скаляр [0, 2²⁵⁶) (32 байта, big-endian)
```

> **Оптимизация:** скаляр редуцируется по порядку группы (`s mod r`) через `getMont`, затем используется оконный метод с эндоморфизмом для ускорения. Для G1 выполняется **проверка подгруппы** (`checkSubgroup = true`).

---

### `eth_evm_bls12381_g2mul`

**EIP-2537**  
**Сигнатура:**
```nim
func eth_evm_bls12381_g2mul*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 256 байт
  inputs: openArray[byte]        # ОБЯЗАТЕЛЬНО 288 байт: [Px(128) | Py(128) | s(32)]
): CttEVMStatus
```

**Описание:** Скалярное умножение точки G2: `R = [s]P`.

**Формат входных данных (288 байт):**
```
inputs[0..63]    — Px.c0
inputs[64..127]  — Px.c1
inputs[128..191] — Py.c0
inputs[192..255] — Py.c1
inputs[256..287] — s (32 байта, big-endian)
```

> Как и для G1MUL, выполняется проверка подгруппы и редукция скаляра.

---

### `eth_evm_bls12381_g1msm`

**EIP-2537**  
**Сигнатура:**
```nim
func eth_evm_bls12381_g1msm*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 128 байт
  inputs: openArray[byte]        # N пар [(Pᵢx(64) | Pᵢy(64) | sᵢ(32))], N≥1, кратно 160
): CttEVMStatus
```

**Описание:** Мультискалярное умножение (MSM) на G1: `R = Σ[sᵢ]Pᵢ`. Эффективнее, чем N отдельных вызовов G1MUL, за счёт алгоритма Пиппенджера.

**Формат входных данных:** кратно 160 байтам. Каждые 160 байт — одна пара `(Pᵢ, sᵢ)`:
```
i*160 + 0  .. i*160 + 63  — Pᵢx
i*160 + 64 .. i*160 + 127 — Pᵢy
i*160 + 128.. i*160 + 159 — sᵢ
```

**Управление памятью:** точки и коэффициенты размещаются на **куче** (`allocHeapArrayAligned` с выравниванием 64 байта для SIMD). Память освобождается через `freeHeapAligned` после вычисления.

**Возможные статусы:**
- `cttEVM_Success`
- `cttEVM_InvalidInputSize` — пустой ввод или не кратен 160
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`
- `cttEVM_PointNotOnCurve`

---

### `eth_evm_bls12381_g2msm`

**EIP-2537**  
**Сигнатура:**
```nim
func eth_evm_bls12381_g2msm*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 256 байт
  inputs: openArray[byte]        # N пар [(Pᵢ(256 байт) | sᵢ(32 байта))], кратно 288
): CttEVMStatus
```

**Описание:** Мультискалярное умножение на G2: `R = Σ[sᵢ]Pᵢ`. Каждая пара — 288 байт.

**Формат входных данных:** кратно 288 байтам. Каждые 288 байт:
```
i*288 +   0 .. i*288 +  63 — Pᵢx.c0
i*288 +  64 .. i*288 + 127 — Pᵢx.c1
i*288 + 128 .. i*288 + 191 — Pᵢy.c0
i*288 + 192 .. i*288 + 255 — Pᵢy.c1
i*288 + 256 .. i*288 + 287 — sᵢ
```

---

### `eth_evm_bls12381_pairingcheck`

**EIP-2537**  
**Сигнатура:**
```nim
func eth_evm_bls12381_pairingcheck*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 32 байта
  inputs: openArray[byte]        # N пар [(Pᵢ(128) | Qᵢ(256))], кратно 384, N≥1
): CttEVMStatus
```

**Описание:** Проверка спаривания для BLS12-381:
```
∏ e(Pᵢ, Qᵢ) == 1  →  r = [0...01]
∏ e(Pᵢ, Qᵢ) ≠ 1  →  r = [0...00]
```

**Формат одной пары (384 байта):**
```
pos +   0 .. pos +  63 — Pᵢx    (G1, 64 байта)
pos +  64 .. pos + 127 — Pᵢy    (G1, 64 байта)
pos + 128 .. pos + 191 — Qᵢx.c0 (G2, 64 байта)
pos + 192 .. pos + 255 — Qᵢx.c1 (G2, 64 байта)
pos + 256 .. pos + 319 — Qᵢy.c0 (G2, 64 байта)
pos + 320 .. pos + 383 — Qᵢy.c1 (G2, 64 байта)
```

> **Отличие от BN254:** в BLS12-381 координаты Fp2 кодируются как `(c0 + 𝑖·c1)` в прямом порядке, а не в обратном как в EIP-197.

> **Пустой ввод недопустим** (в отличие от BN254). Вернёт `cttEVM_InvalidInputSize`.

> **Проверка подгруппы выполняется** для обеих точек P (G1) и Q (G2).

**Возможные статусы:**
- `cttEVM_Success`
- `cttEVM_InvalidInputSize`
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`
- `cttEVM_PointNotOnCurve`
- `cttEVM_PointNotInSubgroup`

---

### `eth_evm_bls12381_map_fp_to_g1`

**EIP-2537**  
**Сигнатура:**
```nim
func eth_evm_bls12381_map_fp_to_g1*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 128 байт
  inputs: openArray[byte]        # ОБЯЗАТЕЛЬНО 64 байта: элемент поля Fp
): CttEVMStatus
```

**Описание:** Детерминированное отображение элемента базового поля `Fp` в точку на G1 методом Simplified SWU (`mapToCurve_sswu`). Результат — точка в надлежащей подгруппе (после очистки кофактора `clearCofactor`).

**Формат входных данных (64 байта):**
```
inputs[0..15]  — нули (16 байт, должны быть 0x00)
inputs[16..63] — элемент поля u ∈ [0, p), big-endian (48 байт)
```

**Возможные статусы:**
- `cttEVM_Success`
- `cttEVM_InvalidInputSize`
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`

---

### `eth_evm_bls12381_map_fp2_to_g2`

**EIP-2537**  
**Сигнатура:**
```nim
func eth_evm_bls12381_map_fp2_to_g2*(
  r:      var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 256 байт
  inputs: openArray[byte]        # ОБЯЗАТЕЛЬНО 128 байт: элемент поля Fp2
): CttEVMStatus
```

**Описание:** Отображение элемента расширенного поля `Fp2` в точку G2. Входной элемент `u = u.c0 + 𝑖·u.c1` — пара элементов Fp.

**Формат входных данных (128 байт):**
```
inputs[0..15]   — нули (16 байт)
inputs[16..63]  — u.c0 (48 байт, big-endian): Re(u)
inputs[64..79]  — нули (16 байт)
inputs[80..127] — u.c1 (48 байт, big-endian): Im(u)
```

---

## KZG и протоколы EIP-4844

### `eth_evm_kzg_point_evaluation`

**EIP-4844** (прото-данкшардинг)  
**Сигнатура:**
```nim
func eth_evm_kzg_point_evaluation*(
  ctx:   ptr EthereumKZGContext,  # Указатель на инициализированный KZG-контекст (доверенная настройка)
  r:     var openArray[byte],     # Выходной буфер: ОБЯЗАТЕЛЬНО 64 байта
  input: openArray[byte]          # ОБЯЗАТЕЛЬНО 192 байта
): CttEVMStatus
```

**Описание:** Верифицирует KZG-доказательство вычисления многочлена в точке: `p(z) = y`, где `p(x)` — многочлен, соответствующий KZG-обязательству. Дополнительно проверяет совпадение обязательства с `versioned_hash`.

**Формат входных данных (192 байта):**
```
input[0..31]    — versioned_hash (32 байта): SHA-256 обязательства с версией 0x01
input[32..63]   — z (32 байта, big-endian): точка вычисления (элемент поля Fr[BLS12_381])
input[64..95]   — y (32 байта, big-endian): ожидаемое значение многочлена в точке z
input[96..143]  — commitment (48 байт): KZG-обязательство (точка G1 в сжатом виде)
input[144..191] — proof (48 байт): KZG-доказательство (точка G1 в сжатом виде)
```

**Вычисление versioned_hash:**
```
versioned_hash = 0x01 || SHA256(commitment)[1:]
```
Первый байт SHA-256 дайджеста заменяется на версию `0x01`.

**Формат выходных данных (64 байта):**
```
r[0..31]  — FIELD_ELEMENTS_PER_BLOB (uint256, big-endian): число элементов поля в блобе
r[32..63] — BLS12-381 модуль поля Fr (uint256, big-endian)
```

**Порядок верификации:**
1. Проверка размеров буферов.
2. Восстановление `versioned_hash` из `commitment` и сравнение с входным.
3. Вызов `verify_kzg_proof(commitment, z, y, proof)`.
4. Запись результирующих констант в `r`.

**Инициализация контекста:**

```nim
import constantine/ethereum_evm_precompiles

# Реэкспортируемые типы (см. раздел ниже)
# Создать контекст из файла доверенной настройки
var ctx: ptr EthereumKZGContext
let status = new(ctx, "trusted_setup.json", TrustedSetupFormat.json)

# ... использовать ctx для вызовов ...

# Освободить контекст
delete(ctx)
```

---

## ECDSA: восстановление публичного ключа

### `eth_evm_ecrecover`

**Адрес EVM:** 1  
**Сигнатура:**
```nim
func eth_evm_ecrecover*(
  r:     var openArray[byte],   # Выходной буфер: ОБЯЗАТЕЛЬНО 32 байта
  input: openArray[byte]        # ОБЯЗАТЕЛЬНО 128 байт
): CttEVMStatus
```

**Описание:** Восстанавливает Ethereum-адрес (последние 20 байт keccak256 публичного ключа) из хэша сообщения и подписи ECDSA на кривой Secp256k1.

**Формат входных данных (128 байт):**
```
input[0..31]   — msgHash: keccak256-хэш подписанного сообщения (32 байта)
input[32..63]  — v: чётность Y-координаты восстановленной точки (32 байта)
input[64..95]  — r: компонент r подписи, скаляр Fr[Secp256k1] (32 байта, big-endian)
input[96..127] — s: компонент s подписи, скаляр Fr[Secp256k1] (32 байта, big-endian)
```

**Допустимые значения `v`:**
| Значение | Смысл                      |
|----------|---------------------------|
| `0`      | чётная Y-координата       |
| `1`      | нечётная Y-координата     |
| `27`     | чётная Y (legacy Ethereum) |
| `28`     | нечётная Y (legacy Ethereum) |

> Байты `input[32..62]` (первые 31 байт блока v) должны быть нулями. Иначе — `cttEVM_MalformedSignature`.

**Формат выходных данных (32 байта):**
```
r[0..11]  — нули (12 байт)
r[12..31] — Ethereum-адрес (последние 20 байт keccak256 от несжатого публичного ключа)
```

**Алгоритм:**
1. Построение `msgHash` как скаляра `Fr[Secp256k1]`.
2. Валидация `v` (первые 31 байт = 0, последний байт ∈ {0, 1, 27, 28}).
3. Парсинг `r`, `s` в `Signature`.
4. Восстановление публичного ключа через `recoverPubkeyFromDigest`.
5. Сериализация несжатого ключа `(x, y)` в 64 байта.
6. keccak256-хэш → усечение до последних 20 байт.

**Совместимость:** реализация следует поведению Geth (go-ethereum).

**Пример:**
```nim
var inputBuf: array[128, byte]
# Заполнить msgHash, v, r, s ...

var addrBuf: array[32, byte]
let status = eth_evm_ecrecover(addrBuf, inputBuf)

if status == cttEVM_Success:
  # addrBuf[12..31] содержит Ethereum-адрес (20 байт)
  echo "Адрес восстановлен"
elif status == cttEVM_MalformedSignature:
  echo "Некорректный формат подписи"
```

---

## Реэкспортируемые типы и API

Модуль реэкспортирует следующие символы из зависимостей:

```nim
# Из ethereum_eip4844_kzg:
export EthereumKZGContext    # Тип: непрозрачный контекст доверенной настройки KZG
export TrustedSetupFormat    # Enum: формат файла доверенной настройки (json, binary)
export TrustedSetupStatus    # Enum: статус загрузки настройки
export new                   # proc new(ctx: var ptr EthereumKZGContext, ...): TrustedSetupStatus
export new_with_precompute   # proc: создать контекст с предвычислением
export delete                # proc delete(ctx: ptr EthereumKZGContext)

# Из hashes (технически не предкомпиляция, но реэкспортируется):
export hashes                # Модуль: sha256, keccak256, ripemd160 и их методы
```

**Использование KZG-контекста:**
```nim
import constantine/ethereum_evm_precompiles

var ctx: ptr EthereumKZGContext

# Вариант 1: из файла JSON (стандартная доверенная настройка Ethereum)
let loadStatus = new(ctx, "trusted_setup.json", TrustedSetupFormat.json)

# Вариант 2: с предвычислением (быстрее верификация, больше памяти)
let loadStatus2 = new_with_precompute(ctx, "trusted_setup.json", TrustedSetupFormat.json)

# После использования — освободить
delete(ctx)
```

---

## Внутренние вспомогательные функции

Следующие функции используются внутри модуля и не являются частью публичного API:

### `parseEip2537`

```nim
proc parseEip2537(
  dst: var Fp[BLS12_381],
  src: openArray[byte]   # ОБЯЗАТЕЛЬНО 64 байта
): CttEvmStatus {.inline.}
```

Парсинг координаты по правилам EIP-2537: 16 нулевых байт + 48 байт значения. Проверяет, что значение строго меньше простого модуля поля.

### `parseRawUint` (обобщённая)

```nim
func parseRawUint(dst: var Fp[BLS12_381], src: openarray[byte]): CttEVMStatus
func parseRawUint(dst: var Fp[BN254_Snarks], src: openarray[byte]): CttEVMStatus
```

Специализированный парсинг в зависимости от кривой. Для BN254 — прямой `unmarshal`; для BLS12-381 — делегирует `parseEip2537`.

### `fromRawCoords` (обобщённые)

Семейство перегруженных обобщённых функций для парсинга точек из сырых координат:

```nim
# Аффинная точка G1 (Fp)
func fromRawCoords[Name, G](
  dst: var EC_ShortW_Aff[Fp[Name], G],
  x, y: openarray[byte],
  checkSubgroup: bool): CttEVMStatus

# Аффинная точка G2 (Fp2)
func fromRawCoords[Name](
  dst: var EC_ShortW_Aff[Fp2[Name], G2],
  x0, x1, y0, y1: openarray[byte],
  checkSubgroup: bool): CttEVMStatus

# Проективная/Якобиева точка G1 (Fp) — конвертирует через аффинную
func fromRawCoords[Name, G](
  dst: var EC_ShortW_Jac[Fp[Name], G],
  x, y: openarray[byte],
  checkSubgroup: bool): CttEVMStatus

# Проективная/Якобиева точка G2 (Fp2)
func fromRawCoords[Name, G](
  dst: var EC_ShortW_Jac[Fp2[Name], G],
  x0, x1, y0, y1: openarray[byte],
  checkSubgroup: bool): CttEVMStatus
```

Порядок валидации внутри:
1. Парсинг координат через `parseRawUint`.
2. Обработка точки на бесконечности `(0, 0)`.
3. Проверка принадлежности кривой через `isOnCurve`.
4. Опционально: проверка подгруппы через `isInSubgroup`.

### `kzg_to_versioned_hash` (внутренняя)

```nim
proc kzg_to_versioned_hash(
  r: var array[32, byte],
  commitment_bytes: array[48, byte])
```

Вычисляет `versioned_hash = 0x01 || SHA256(commitment)[1:]` для EIP-4844.

---

## Таблица всех функций

| Функция | Адрес EVM | EIP | Вход (байт) | Выход (байт) |
|---------|-----------|-----|------------|--------------|
| `eth_evm_sha256` | 2 | — | произв. | 32 |
| `eth_evm_ripemd160` | 3 | — | произв. | 32 |
| `eth_evm_modexp_result_size` | — | 198 | произв. | `uint64` (через `var`) |
| `eth_evm_modexp` | 5 | 198 | произв. | `modulusLen` |
| `eth_evm_bn254_g1add` | 6 | 196 | 128 (pad) | 64 |
| `eth_evm_bn254_g1mul` | 7 | 196 | 96 (pad) | 64 |
| `eth_evm_bn254_ecpairingcheck` | 8 | 197/1108 | N×192 | 32 |
| `eth_evm_bls12381_g1add` | — | 2537 | 256 | 128 |
| `eth_evm_bls12381_g2add` | — | 2537 | 512 | 256 |
| `eth_evm_bls12381_g1mul` | — | 2537 | 160 | 128 |
| `eth_evm_bls12381_g2mul` | — | 2537 | 288 | 256 |
| `eth_evm_bls12381_g1msm` | — | 2537 | N×160 | 128 |
| `eth_evm_bls12381_g2msm` | — | 2537 | N×288 | 256 |
| `eth_evm_bls12381_pairingcheck` | — | 2537 | N×384, N≥1 | 32 |
| `eth_evm_bls12381_map_fp_to_g1` | — | 2537 | 64 | 128 |
| `eth_evm_bls12381_map_fp2_to_g2` | — | 2537 | 128 | 256 |
| `eth_evm_kzg_point_evaluation` | — | 4844 | 192 | 64 |
| `eth_evm_ecrecover` | 1 | — | 128 | 32 |

---

## Общие замечания по безопасности

**Constant-time:** большинство криптографических операций реализованы в постоянном времени (не зависящем от секретных данных) для защиты от атак по времени. Функции с суффиксом `_vartime` — вариативного времени и не должны использоваться с секретными данными.

**Исключения отключены:** директива `{.push raises: [].}` гарантирует, что ни одна функция не генерирует исключений. Все ошибки передаются через `CttEVMStatus`.

**C FFI:** все публичные функции экспортируются с префиксом `ctt_` через прагму `{.libPrefix: prefix_ffi.}`. Например, `eth_evm_sha256` доступна из C как `ctt_eth_evm_sha256`.

**Валидация входных данных:** всегда проверяйте `CttEVMStatus` перед использованием выходного буфера. При ошибке содержимое `r` не определено.

**Версия Nim:** требуется Nim 2.2.0 или новее.

**Компилятор:** рекомендуется Clang 14+ для оптимальной производительности и корректной генерации кода с Intel-синтаксисом ассемблера.
