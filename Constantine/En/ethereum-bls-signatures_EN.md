# Ethereum BLS Signatures (Constantine)

*Module `constantine/ethereum_bls_signatures` — implementation of BLS (Boneh-Lynn-Schacham) signatures on the BLS12-381 curve for the Ethereum protocol (consensus layer).*

> Library repository: https://github.com/mratsim/constantine
> File: `constantine/ethereum_bls_signatures.nim`

---

## About the module

The module implements BLS signatures on top of the BLS12-381 curve (Barreto-Lynn-Scott), used by the Ethereum consensus layer (Beacon Chain). It uses the **proof-of-possession (PoP)** scheme: each public key is assumed to already be associated with a deposit proof, so `PopProve`/`PopVerify` from the IETF spec are not implemented.

**Ciphersuite:**

| Component | Description |
|---|---|
| Secret keys | `Fr` field (32 bytes) |
| Public keys | `G1` group (48 bytes compressed, 96 bytes uncompressed) |
| Signatures | `G2` group (96 bytes compressed, 192 bytes uncompressed) |
| Domain Separation Tag | `BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_` |
| Hash function | SHA-256 |

**Specs:**

- https://github.com/ethereum/consensus-specs/blob/v1.2.0/specs/phase0/beacon-chain.md#bls-signatures
- https://github.com/ethereum/consensus-specs/blob/v1.2.0/specs/altair/bls.md
- https://www.ietf.org/archive/id/draft-irtf-cfrg-bls-signature-05.html
- Test vectors: https://github.com/ethereum/bls12-381-tests

---

## Importing

```nim
import constantine/ethereum_bls_signatures
```

The module is marked `{.checks: off.}` — exceptions are deliberately disabled in the cryptographic core (a defense against side-channel leaks through exceptions).

---

## Core types

```nim
type
  SecretKey* = object
    ## A BLS12-381 secret key (wraps Fr[BLS12_381])

  PublicKey* = object
    ## A public key on G1 (48/96 bytes)

  Signature* = object
    ## A signature on G2 (96/192 bytes)

  BatchSigAccumulator* = object
    ## Accumulator for streaming batch verification

  cttEthBlsStatus* = enum
    cttEthBls_Success
    cttEthBls_VerificationFailure
    cttEthBls_InputsLengthsMismatch
    cttEthBls_ZeroLengthAggregation
    cttEthBls_PointAtInfinity
```

`SecretKey`, `PublicKey` and `Signature` are declared with `{.byref.}` — they are passed by reference and should not be copied carelessly.

Codec statuses (`CttCodecScalarStatus`, `CttCodecEccStatus`) are re-exported from `serialization/codecs_status_codes`.

---

## Comparison and zero checks

```nim
func pubkey_is_zero*(pubkey: PublicKey): bool
func signature_is_zero*(sig: Signature): bool
func pubkeys_are_equal*(a, b: PublicKey): bool
func signatures_are_equal*(a, b: Signature): bool
```

Example (in the project's code style):

```nim
var
  pk1, pk2: PublicKey

if pubkeys_are_equal(pk1, pk2):
  echo "Keys match"

if pubkey_is_zero(pk1):
  echo "Key is zero — must not be used"
```

---

## Validation

```nim
func validate_seckey*(secret_key: SecretKey): CttCodecScalarStatus
func validate_pubkey*(public_key: PublicKey): CttCodecEccStatus
func validate_signature*(signature: Signature): CttCodecEccStatus
```

- `validate_seckey` is cheap; timing can leak **only** for an invalid key (zero or larger than the group order).
- `validate_pubkey` / `validate_signature` are **expensive** operations (subgroup check) — caching the result makes sense.

```nim
let status = validate_pubkey(pk1)
if status != cttCodecEcc_Success:
  echo "Public key is invalid"
```

---

## Serialization / deserialization (codecs)

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

⚠️ **Warning:** the `*_unchecked` variants skip the expensive subgroup check. This exposes the protocol to small-subgroup attacks. Use them only when validation will be performed separately (e.g. via `validate_pubkey`/`validate_signature`).

Serialization/deserialization example in the accepted code style:

```nim
var
  buf: array[48, byte]
  pk: PublicKey

let status = serialize_pubkey_compressed(buf, pk)
if status != cttCodecEcc_Success:
  echo "Failed to serialize the key"

var pk2: PublicKey
if deserialize_pubkey_compressed(pk2, buf) == cttCodecEcc_Success:
  echo "Key successfully read and validated"
```

---

## Key derivation

```nim
func derive_pubkey*(public_key: var PublicKey, secret_key: SecretKey)
```

Derives the public key matching a secret key. **Precondition:** `secret_key` MUST have been validated beforehand (`validate_seckey`).

```nim
var
  sk: SecretKey
  pk: PublicKey

discard deserialize_seckey(sk, secretBytes)
derive_pubkey(pk, sk)
```

---

## Signing a single message

```nim
func sign*(signature: var Signature, secret_key: SecretKey, message: openArray[byte])
```

Signs `message` with `secret_key`. The Domain Separation Tag is fixed: `BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_`. `secret_key` MUST have been validated.

```nim
var sig: Signature

sign(sig, sk, toOpenArray(messageBytes, 0, len(messageBytes) - 1))
```

---

## Verifying a single message

```nim
func verify*(public_key: PublicKey, message: openArray[byte], signature: Signature): cttEthBlsStatus
```

Checks that `signature` is valid for `message` under `public_key`. The key and signature are assumed to already be on-curve and subgroup-checked (via derivation/deserialization or explicit validation).

The case where the key or signature turns out to be zero (point at infinity) is handled separately — `cttEthBls_PointAtInfinity` is returned instead of falling through to verification.

```nim
let status = verify(pk, message, sig)

case status
of cttEthBls_Success:
  echo "Signature is valid"
of cttEthBls_PointAtInfinity:
  echo "Key or signature is the point at infinity"
else:
  echo "Signature is invalid"
```

---

## Aggregation (unstable API)

```nim
func aggregate_pubkeys_unstable_api*(aggregate_pubkey: var PublicKey, pubkeys: openArray[PublicKey])
func aggregate_signatures_unstable_api*(aggregate_sig: var Signature, signatures: openArray[Signature])
```

Aggregate a collection of keys/signatures into a single object. Input elements are assumed to already be validated (by deserialization or explicitly). An empty input array yields the neutral element (point at infinity) rather than an error.

```nim
var aggPk: PublicKey

aggregate_pubkeys_unstable_api(aggPk, pubkeysSeq)
```

---

## Fast Aggregate Verify

```nim
func fast_aggregate_verify*(pubkeys: openArray[PublicKey], message: openArray[byte], aggregate_sig: Signature): cttEthBlsStatus
```

Verifies a **single** signature `aggregate_sig` against a **single** message `message`, but under the aggregate of several public keys. Used when all participants signed the same block/attestation.

Checks per the IETF spec:

- empty `pubkeys` → `cttEthBls_ZeroLengthAggregation`;
- a zero signature or at least one zero key → `cttEthBls_PointAtInfinity`.

```nim
let status = fast_aggregate_verify(pubkeysSeq, message, aggSig)
```

---

## Aggregate Verify (distinct messages)

```nim
func aggregate_verify*[Msg](pubkeys: openArray[PublicKey], messages: openArray[Msg], aggregate_sig: Signature): cttEthBlsStatus
```

Version for the case where each key has its own message (pairs of `(pubkey, message)`).

⚠️ **Protection against rogue-key / splitting-zeros attacks is the caller's responsibility:**

1. Public keys signing the same message MUST be aggregated and checked for zero beforehand.
2. Augmentation or proof-of-possession must be used for each public key.

```nim
let status = aggregate_verify(pubkeysSeq, messagesSeq, aggSig)

if status == cttEthBls_InputsLengthsMismatch:
  echo "Number of keys and messages does not match"
```

There is also a low-level C FFI variant taking raw pointers (`ptr UncheckedArray[PublicKey]`, `ptr UncheckedArray[View[byte]]`, `len: int`) — used when calling from C/C++; usually not needed in plain Nim code.

---

## Batch Verify

```nim
func batch_verify*[Msg](pubkeys: openArray[PublicKey], messages: openArray[Msg], signatures: openArray[Signature], secureRandomBytes: array[32, byte]): cttEthBlsStatus
```

Verifies multiple independent `(pubkey, message, signature)` triplets at once. Returns success only if **all** triplets are valid.

`secureRandomBytes` must be cryptographically secure random bytes not under the attacker's control. Without them the scheme is vulnerable to a zero-splitting attack (see https://ethresear.ch/t/fast-verification-of-multiple-bls-signatures/5407/14).

```nim
var randomBytes: array[32, byte]

# randomBytes must be filled by a cryptographically secure RNG

let status = batch_verify(pubkeysSeq, messagesSeq, signaturesSeq, randomBytes)
```

As with `aggregate_verify`, there is a low-level C FFI variant using pointers instead of `openArray`.

---

## BatchSigAccumulator — streaming batch verify

Used when `(pubkey, message, signature)` triplets arrive one at a time (e.g. as messages come in from the network), avoiding the need to keep everything in memory at once.

```nim
func alloc_batch_sig_accumulator*(): ptr BatchSigAccumulator
proc free_batch_sig_accumulator*(p: ptr BatchSigAccumulator)

func init_batch_sig_accumulator*(ctx: var BatchSigAccumulator, secureRandomBytes: array[32, byte], accumSepTag: ptr UncheckedArray[byte], accumSepTagLen: int)

func update_batch_sig_accumulator*(ctx: var BatchSigAccumulator, pubkey: PublicKey, message: ptr UncheckedArray[byte], messageLen: int, signature: Signature): bool

func final_verify_batch_sig_accumulator*(ctx: var BatchSigAccumulator): bool
```

Accumulator lifecycle:

1. `alloc_batch_sig_accumulator()` — allocates memory (64-byte aligned).
2. `init_batch_sig_accumulator(...)` — initializes with `secureRandomBytes` and an optional separation tag (`accumSepTag`), useful in a multithreaded context — each thread gets its own independent accumulator seeded from a single source of randomness.
3. `update_batch_sig_accumulator(...)` — called repeatedly, one triplet at a time. Returns `false` if the key or signature is the point at infinity.
4. `final_verify_batch_sig_accumulator(...)` — the final check. Returns `false` if nothing was accumulated, or if verification fails.
5. `free_batch_sig_accumulator(p)` — frees the memory.

```nim
var
  randomBytes: array[32, byte]
  ctx = alloc_batch_sig_accumulator()

init_batch_sig_accumulator(ctx[], randomBytes, nil, 0)

for triplet in triplets:
  let ok = update_batch_sig_accumulator(ctx[], triplet.pubkey, triplet.msgPtr, triplet.msgLen, triplet.signature)
  if not ok:
    echo "Triplet discarded: zero key or signature"

if final_verify_batch_sig_accumulator(ctx[]):
  echo "All accumulated signatures are valid"
else:
  echo "At least one signature is invalid"

free_batch_sig_accumulator(ctx)
```

---

## Gotchas

- **`*_unchecked` deserialization variants** skip the subgroup check — use them only when validation is performed separately, otherwise the protocol is open to small-subgroup attacks.
- **`fast_aggregate_verify` and `aggregate_verify`** require the caller to defend against rogue-key attacks — aggregate and zero-check keys signing the same message before calling these functions.
- **`batch_verify`** without a high-quality `secureRandomBytes` loses its protection against the zero-splitting attack — the randomness source must be cryptographically secure and outside the attacker's control.
- **`aggregate_pubkeys_unstable_api` / `aggregate_signatures_unstable_api`** do not return an error on an empty input array, but silently write the neutral element (point at infinity) — this must be checked explicitly (`pubkey_is_zero` / `signature_is_zero`) before using the result.
- **`verify`** handles the point-at-infinity case separately, before the cryptographic check itself — if the key or signature were never properly initialized (accidentally zero-initialized), the result is `cttEthBls_PointAtInfinity`, not `cttEthBls_VerificationFailure`. Worth distinguishing between the two in error handling.
- The Domain Separation Tag (`BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_`) is hardcoded and not configurable — this is specific to the Ethereum (PoP) scheme; a different BLS scheme requires a different module/tag.
- The module is compiled with `{.checks: off.}` — exceptions are not used at all in the cryptographic core; all errors are communicated via status codes (`cttEthBlsStatus`, `CttCodecScalarStatus`, `CttCodecEccStatus`), not `raise`.

---

## Status code summary

| Status | When it occurs |
|---|---|
| `cttEthBls_Success` | operation succeeded |
| `cttEthBls_VerificationFailure` | signature/aggregate failed the cryptographic check |
| `cttEthBls_InputsLengthsMismatch` | key/message/signature array lengths do not match |
| `cttEthBls_ZeroLengthAggregation` | an empty key array was passed where the spec forbids it |
| `cttEthBls_PointAtInfinity` | a key or signature turned out to be the neutral element (point at infinity) |
