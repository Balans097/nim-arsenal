# Ethereum BLS Signatures (Constantine)

*Модуль `constantine/ethereum_bls_signatures` — реализация BLS-подписей (Boneh-Lynn-Schacham) на кривой BLS12-381 для протокола Ethereum (consensus layer).*

> Репозиторий библиотеки: https://github.com/mratsim/constantine
> Файл: `constantine/ethereum_bls_signatures.nim`

---

## О модуле

Модуль реализует схему BLS-подписей поверх кривой BLS12-381 (Barreto-Lynn-Scott), используемую в Ethereum consensus layer (Beacon Chain). Используется схема **proof-of-possession (PoP)**: подразумевается, что для каждого публичного ключа уже есть депозит-пруф, поэтому `PopProve`/`PopVerify` из IETF-спецификации не реализованы.

**Криптонабор (ciphersuite):**

| Компонент | Описание |
|---|---|
| Секретные ключи | поле `Fr` (32 байта) |
| Публичные ключи | группа `G1` (48 байт сжатый формат, 96 байт несжатый) |
| Подписи | группа `G2` (96 байт сжатый формат, 192 байта несжатый) |
| Domain Separation Tag | `BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_` |
| Хэш-функция | SHA-256 |

**Спецификации:**

- https://github.com/ethereum/consensus-specs/blob/v1.2.0/specs/phase0/beacon-chain.md#bls-signatures
- https://github.com/ethereum/consensus-specs/blob/v1.2.0/specs/altair/bls.md
- https://www.ietf.org/archive/id/draft-irtf-cfrg-bls-signature-05.html
- Тестовые векторы: https://github.com/ethereum/bls12-381-tests

---

## Подключение

```nim
import constantine/ethereum_bls_signatures
```

Модуль помечен `{.checks: off.}` — в криптографическом ядре исключения отключены намеренно (защита от side-channel через исключения).

---

## Основные типы

```nim
type
  SecretKey* = object
    ## Секретный ключ BLS12-381 (обёртка над Fr[BLS12_381])

  PublicKey* = object
    ## Публичный ключ на G1 (48/96 байт)

  Signature* = object
    ## Подпись на G2 (96/192 байта)

  BatchSigAccumulator* = object
    ## Аккумулятор для потокового batch-verify

  cttEthBlsStatus* = enum
    cttEthBls_Success
    cttEthBls_VerificationFailure
    cttEthBls_InputsLengthsMismatch
    cttEthBls_ZeroLengthAggregation
    cttEthBls_PointAtInfinity
```

`SecretKey`, `PublicKey` и `Signature` объявлены с `{.byref.}` — передаются по ссылке, не копируются бездумно.

Статусы кодеков (`CttCodecScalarStatus`, `CttCodecEccStatus`) реэкспортируются из `serialization/codecs_status_codes`.

---

## Сравнение и проверка на ноль

```nim
func pubkey_is_zero*(pubkey: PublicKey): bool
func signature_is_zero*(sig: Signature): bool
func pubkeys_are_equal*(a, b: PublicKey): bool
func signatures_are_equal*(a, b: Signature): bool
```

Пример (в стиле проекта):

```nim
var
  pk1, pk2: PublicKey

if pubkeys_are_equal(pk1, pk2):
  echo "Ключи совпадают"

if pubkey_is_zero(pk1):
  echo "Ключ нулевой — использовать нельзя"
```

---

## Валидация

```nim
func validate_seckey*(secret_key: SecretKey): CttCodecScalarStatus
func validate_pubkey*(public_key: PublicKey): CttCodecEccStatus
func validate_signature*(signature: Signature): CttCodecEccStatus
```

- `validate_seckey` — дешёвая операция, утечка тайминга возможна **только** для невалидного ключа (ноль или превышение порядка группы).
- `validate_pubkey` / `validate_signature` — **дорогие** операции (subgroup check), результат имеет смысл кэшировать.

```nim
let status = validate_pubkey(pk1)
if status != cttCodecEcc_Success:
  echo "Публичный ключ невалиден"
```

---

## Сериализация / десериализация (codecs)

```nim
proc serialize_seckey*(dst: var array[32, byte], secret_key: SecretKey)
func serialize_pubkey_compressed*(dst: var array[48, byte], public_key: PublicKey): CttCodecEccStatus
func serialize_signature_compressed*(dst: var array[96, byte], signature: Signature): CttCodecEccStatus

func deserialize_seckey*(dst: var SecretKey, src: array[32, byte]): CttCodecScalarStatus
func deserialize_pubkey_compressed_unchecked*(dst: var PublicKey, src: array[48, byte]): CttCodecEccStatus
func deserialize_pubkey_compressed*(dst: var PublicKey, src: array[48, byte]): CttCodecEccStatus
func deserialize_signature_compressed_unchecked*(dst: var Signature, src: array[96, byte]): CttCodecEccStatus
func deserialize_signature_compressed*(dst: var Signature, src: array[96, byte]): CttCodecEccStatus
```

⚠️ **Важно:** варианты `*_unchecked` пропускают дорогую проверку подгруппы (subgroup check). Это открывает протокол для атак на малую подгруппу (small subgroup attack). Использовать их можно только тогда, когда валидация будет выполнена отдельно (например, через `validate_pubkey`/`validate_signature`).

Пример сериализации/десериализации в принятом стиле кода:

```nim
var
  buf: array[48, byte]
  pk: PublicKey

let status = serialize_pubkey_compressed(buf, pk)
if status != cttCodecEcc_Success:
  echo "Не удалось сериализовать ключ"

var pk2: PublicKey
if deserialize_pubkey_compressed(pk2, buf) == cttCodecEcc_Success:
  echo "Ключ успешно прочитан и провалидирован"
```

---

## Деривация ключей

```nim
func derive_pubkey*(public_key: var PublicKey, secret_key: SecretKey)
```

Выводит публичный ключ из секретного. **Предусловие:** `secret_key` должен быть провалидирован заранее (`validate_seckey`).

```nim
var
  sk: SecretKey
  pk: PublicKey

discard deserialize_seckey(sk, secretBytes)
derive_pubkey(pk, sk)
```

---

## Подпись одного сообщения

```nim
func sign*(signature: var Signature, secret_key: SecretKey, message: openArray[byte])
```

Подписывает `message` секретным ключом `secret_key`. Domain Separation Tag фиксирован: `BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_`. `secret_key` должен быть провалидирован.

```nim
var sig: Signature

sign(sig, sk, toOpenArray(messageBytes, 0, len(messageBytes) - 1))
```

---

## Верификация одного сообщения

```nim
func verify*(public_key: PublicKey, message: openArray[byte], signature: Signature): cttEthBlsStatus
```

Проверяет подпись `signature` для `message` под ключом `public_key`. Предполагается, что ключ и подпись уже on-curve и subgroup-checked (через деривацию/десериализацию или явную валидацию).

Отдельно обрабатывается случай, когда ключ или подпись оказались нулевыми (point at infinity) — тогда возвращается `cttEthBls_PointAtInfinity`, а не падение в верификации.

```nim
let status = verify(pk, message, sig)

case status
of cttEthBls_Success:
  echo "Подпись верна"
of cttEthBls_PointAtInfinity:
  echo "Ключ или подпись — точка на бесконечности"
else:
  echo "Подпись неверна"
```

---

## Агрегация (unstable API)

```nim
func aggregate_pubkeys_unstable_api*(aggregate_pubkey: var PublicKey, pubkeys: openArray[PublicKey])
func aggregate_signatures_unstable_api*(aggregate_sig: var Signature, signatures: openArray[Signature])
```

Агрегируют коллекцию ключей/подписей в один объект. Предполагается, что входные элементы уже провалидированы (десериализацией или явно). Пустой массив на входе даёт нейтральный элемент (точку на бесконечности), а не ошибку.

```nim
var aggPk: PublicKey

aggregate_pubkeys_unstable_api(aggPk, pubkeysSeq)
```

---

## Fast Aggregate Verify

```nim
func fast_aggregate_verify*(pubkeys: openArray[PublicKey], message: openArray[byte], aggregate_sig: Signature): cttEthBlsStatus
```

Проверка **одной** подписи `aggregate_sig` против **одного** сообщения `message`, но под агрегатом нескольких публичных ключей. Используется, когда все участники подписывали один и тот же блок/аттестацию.

Проверки по спецификации IETF:

- пустой `pubkeys` → `cttEthBls_ZeroLengthAggregation`;
- нулевая подпись или хотя бы один нулевой ключ → `cttEthBls_PointAtInfinity`.

```nim
let status = fast_aggregate_verify(pubkeysSeq, message, aggSig)
```

---

## Aggregate Verify (разные сообщения)

```nim
func aggregate_verify*[Msg](pubkeys: openArray[PublicKey], messages: openArray[Msg], aggregate_sig: cttEthBlsStatus): cttEthBlsStatus
```

Версия для случая, когда у каждого ключа — своё сообщение (пары `(pubkey, message)`).

⚠️ **Защита от rogue-key / splitting-zeros атак — ответственность вызывающего:**

1. Публичные ключи, подписывающие одно и то же сообщение, должны быть заранее агрегированы и проверены на ноль.
2. Для каждого ключа должна использоваться augmentation или proof-of-possession.

```nim
let status = aggregate_verify(pubkeysSeq, messagesSeq, aggSig)

if status == cttEthBls_InputsLengthsMismatch:
  echo "Количество ключей и сообщений не совпадает"
```

Есть также низкоуровневый C FFI вариант с сырыми указателями (`ptr UncheckedArray[PublicKey]`, `ptr UncheckedArray[View[byte]]`, `len: int`) — используется при вызове из C/C++, в чистом Nim-коде обычно не нужен.

---

## Batch Verify

```nim
func batch_verify*[Msg](pubkeys: openArray[PublicKey], messages: openArray[Msg], signatures: openArray[Signature], secureRandomBytes: array[32, byte]): cttEthBlsStatus
```

Проверяет сразу множество независимых троек `(pubkey, message, signature)`. Возвращает успех, только если валидны **все** тройки.

`secureRandomBytes` — обязательные криптографически случайные байты, не контролируемые атакующим. Без них схема уязвима к атаке через подмену нулей (см. https://ethresear.ch/t/fast-verification-of-multiple-bls-signatures/5407/14).

```nim
var randomBytes: array[32, byte]

# randomBytes должен быть заполнен криптографически стойким генератором

let status = batch_verify(pubkeysSeq, messagesSeq, signaturesSeq, randomBytes)
```

Аналогично `aggregate_verify`, есть низкоуровневый C FFI вариант с указателями вместо `openArray`.

---

## BatchSigAccumulator — потоковый batch-verify

Используется, когда тройки `(pubkey, message, signature)` поступают по одной (например, по мере приёма сообщений из сети), и нет необходимости держать всё в памяти одновременно.

```nim
func alloc_batch_sig_accumulator*(): ptr BatchSigAccumulator
proc free_batch_sig_accumulator*(p: ptr BatchSigAccumulator)

func init_batch_sig_accumulator*(ctx: var BatchSigAccumulator, secureRandomBytes: array[32, byte], accumSepTag: ptr UncheckedArray[byte], accumSepTagLen: int)

func update_batch_sig_accumulator*(ctx: var BatchSigAccumulator, pubkey: PublicKey, message: ptr UncheckedArray[byte], messageLen: int, signature: Signature): bool

func final_verify_batch_sig_accumulator*(ctx: var BatchSigAccumulator): bool
```

Жизненный цикл аккумулятора:

1. `alloc_batch_sig_accumulator()` — выделение памяти (выровненной под 64 байта).
2. `init_batch_sig_accumulator(...)` — инициализация с `secureRandomBytes` и опциональным тегом разделения (`accumSepTag`), полезным в многопоточном контексте — каждый поток получает свой независимый аккумулятор из одного источника случайности.
3. `update_batch_sig_accumulator(...)` — многократно, по одной тройке за раз. Возвращает `false`, если ключ или подпись — точка на бесконечности.
4. `final_verify_batch_sig_accumulator(...)` — завершающая проверка. Возвращает `false`, если в аккумулятор ничего не было добавлено, либо если проверка не прошла.
5. `free_batch_sig_accumulator(p)` — освобождение памяти.

```nim
var
  randomBytes: array[32, byte]
  ctx = alloc_batch_sig_accumulator()

init_batch_sig_accumulator(ctx[], randomBytes, nil, 0)

for triplet in triplets:
  let ok = update_batch_sig_accumulator(ctx[], triplet.pubkey, triplet.msgPtr, triplet.msgLen, triplet.signature)
  if not ok:
    echo "Тройка отброшена: нулевой ключ или подпись"

if final_verify_batch_sig_accumulator(ctx[]):
  echo "Все накопленные подписи валидны"
else:
  echo "Хотя бы одна подпись невалидна"

free_batch_sig_accumulator(ctx)
```

---

## Подводные камни

- **`*_unchecked` варианты десериализации** пропускают subgroup check — использовать только при наличии отдельной валидации, иначе протокол открыт для атак на малую подгруппу.
- **`fast_aggregate_verify` и `aggregate_verify`** требуют, чтобы вызывающий сам защитился от rogue-key атак — агрегация и проверка на ноль ключей, подписывающих одно сообщение, до вызова этих функций.
- **`batch_verify`** без качественного `secureRandomBytes` теряет защиту от атаки с подменой нулей — источник случайности обязан быть криптографически стойким и не контролируемым атакующим.
- **`aggregate_pubkeys_unstable_api` / `aggregate_signatures_unstable_api`** при пустом входном массиве не возвращают ошибку, а тихо записывают нейтральный элемент (point at infinity) — это нужно проверять явно (`pubkey_is_zero` / `signature_is_zero`), прежде чем использовать результат.
- **`verify`** отдельно обрабатывает point-at-infinity ещё до самой криптографической проверки — если ключ или подпись не инициализированы (zero-init по ошибке), вернётся `cttEthBls_PointAtInfinity`, а не `cttEthBls_VerificationFailure`. Стоит различать эти статусы в обработке ошибок.
- Domain Separation Tag (`BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_`) зашит в модуль и не настраивается — это специфика именно Ethereum-схемы (PoP), для другой схемы BLS нужен другой модуль/тег.
- Модуль скомпилирован с `{.checks: off.}` — в криптографическом ядре исключения не используются вообще, все ошибки идут через статус-коды (`cttEthBlsStatus`, `CttCodecScalarStatus`, `CttCodecEccStatus`), а не через `raise`.

---

## Сводная таблица типов ошибок

| Статус | Когда возникает |
|---|---|
| `cttEthBls_Success` | операция успешна |
| `cttEthBls_VerificationFailure` | подпись/агрегат не прошли криптографическую проверку |
| `cttEthBls_InputsLengthsMismatch` | длины массивов ключей/сообщений/подписей не совпадают |
| `cttEthBls_ZeroLengthAggregation` | передан пустой массив ключей там, где это запрещено спецификацией |
| `cttEthBls_PointAtInfinity` | ключ или подпись оказались нейтральным элементом (точкой на бесконечности) |
