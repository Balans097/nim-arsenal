# Reference: `ethereum_evm_precompiles` — Constantine

> **Library:** [mratsim/constantine](https://github.com/mratsim/constantine)  
> **Source file:** `constantine/ethereum_evm_precompiles.nim`  
> **Language:** Nim  
> **License:** MIT / Apache 2.0

---

## Table of Contents

1. [Module Purpose](#module-purpose)
2. [Imports and Dependencies](#imports-and-dependencies)
3. [Status Type: `CttEVMStatus`](#status-type-cttevmstatus)
4. [Hash Functions and General Cryptography](#hash-functions-and-general-cryptography)
   - [eth_evm_sha256](#eth_evm_sha256)
   - [eth_evm_ripemd160](#eth_evm_ripemd160)
5. [Modular Exponentiation](#modular-exponentiation)
   - [eth_evm_modexp_result_size](#eth_evm_modexp_result_size)
   - [eth_evm_modexp](#eth_evm_modexp)
6. [BN254 Curve (alt_bn128)](#bn254-curve-altbn128)
   - [eth_evm_bn254_g1add](#eth_evm_bn254_g1add)
   - [eth_evm_bn254_g1mul](#eth_evm_bn254_g1mul)
   - [eth_evm_bn254_ecpairingcheck](#eth_evm_bn254_ecpairingcheck)
7. [BLS12-381 Curve (EIP-2537)](#bls12-381-curve-eip-2537)
   - [eth_evm_bls12381_g1add](#eth_evm_bls12381_g1add)
   - [eth_evm_bls12381_g2add](#eth_evm_bls12381_g2add)
   - [eth_evm_bls12381_g1mul](#eth_evm_bls12381_g1mul)
   - [eth_evm_bls12381_g2mul](#eth_evm_bls12381_g2mul)
   - [eth_evm_bls12381_g1msm](#eth_evm_bls12381_g1msm)
   - [eth_evm_bls12381_g2msm](#eth_evm_bls12381_g2msm)
   - [eth_evm_bls12381_pairingcheck](#eth_evm_bls12381_pairingcheck)
   - [eth_evm_bls12381_map_fp_to_g1](#eth_evm_bls12381_map_fp_to_g1)
   - [eth_evm_bls12381_map_fp2_to_g2](#eth_evm_bls12381_map_fp2_to_g2)
8. [KZG and EIP-4844 Protocol](#kzg-and-eip-4844-protocol)
   - [eth_evm_kzg_point_evaluation](#eth_evm_kzg_point_evaluation)
9. [ECDSA: Public Key Recovery](#ecdsa-public-key-recovery)
   - [eth_evm_ecrecover](#eth_evm_ecrecover)
10. [Re-exported Types and API](#re-exported-types-and-api)
11. [Internal Helper Functions](#internal-helper-functions)
12. [Function Quick Reference Table](#function-quick-reference-table)
13. [Security Notes](#security-notes)

---

## Module Purpose

This module implements **Ethereum EVM precompiled contracts** in Nim using the Constantine cryptographic library. Precompiled contracts are special EVM addresses that execute computationally expensive operations (hashing, elliptic curve arithmetic, pairings) directly in native code rather than through EVM bytecode.

The module covers:
- SHA-256 and RIPEMD-160 hash functions (EVM addresses 2, 3);
- Modular exponentiation — MODEXP (address 5, EIP-198);
- BN254/alt_bn128 curve operations: ECADD, ECMUL, ECPAIRING (addresses 6, 7, 8, EIP-196/197);
- BLS12-381 curve operations (EIP-2537): G1/G2 addition, multiplication, MSM, pairing, field mapping;
- KZG proof verification for EIP-4844 (proto-danksharding);
- ECDSA public key recovery — ECRecover (address 1).

**All functions are declared with `{.push raises: [].}`** — no exceptions are thrown. Errors are communicated via `CttEVMStatus` return codes.

---

## Imports and Dependencies

```nim
import
  ./hashes,                                        # SHA-256, RIPEMD-160, Keccak
  ./platforms/abstractions,                        # Low-level platform abstractions
  ./serialization/io_limbs,                        # BigInt limb serialization
  constantine/named/algebras,                      # Named algebras: BN254_Snarks, BLS12_381, Secp256k1
  ./math/[arithmetic, extension_fields],           # Field arithmetic: Fp, Fr, Fp2, Fp12
  ./math/arithmetic/limbs_montgomery,              # Montgomery form conversion (getMont)
  ./math/ec_shortweierstrass,                      # Short Weierstrass curve point arithmetic
  ./math/elliptic/ec_multi_scalar_mul,             # Multi-scalar multiplication (MSM)
  ./math/pairings/[pairings_generic,               # Generic bilinear pairing implementation
                   miller_accumulators],            # Miller loop accumulator
  ./named/zoo_subgroups,                           # Subgroup membership checks
  ./math/io/[io_bigints, io_fields],               # BigInt and field element I/O
  ./math_arbitrary_precision/arithmetic/bigints_views, # Arbitrary-precision BigInt views
  ./hash_to_curve/hash_to_curve,                   # Hash-to-curve (SSWU method)
  ./ethereum_eip4844_kzg,                          # KZG for EIP-4844
  ./ethereum_ecdsa_signatures                      # ECDSA for ECRecover
```

---

## Status Type: `CttEVMStatus`

```nim
type
  CttEVMStatus* = enum
    cttEVM_Success               # Operation completed successfully
    cttEVM_InvalidInputSize      # Input buffer has wrong size
    cttEVM_InvalidOutputSize     # Output buffer has wrong size
    cttEVM_IntLargerThanModulus  # Integer >= field modulus (not a valid field element)
    cttEVM_PointNotOnCurve       # Point does not lie on the elliptic curve
    cttEVM_PointNotInSubgroup    # Point is not in the required prime-order subgroup
    cttEVM_VerificationFailure   # Cryptographic verification failed
    cttEVM_MalformedSignature    # Signature has an invalid format
```

**Every public function returns `CttEVMStatus`.** The caller must always check the returned value before using the result buffer.

```nim
# Example: status check
var output: array[32, byte]
let data: seq[byte] = @[0x61'u8, 0x62, 0x63]  # "abc"

# Function name first, all arguments inside parentheses
let status = eth_evm_sha256(output, data)
if status != cttEVM_Success:
  echo "Error: ", status
else:
  echo "SHA-256 computed successfully"
```

---

## Hash Functions and General Cryptography

### `eth_evm_sha256`

**EVM Address:** 2  
**Signature:**
```nim
func eth_evm_sha256*(
  r:      var openArray[byte],   # Output buffer: MUST be exactly 32 bytes
  inputs: openArray[byte]        # Arbitrary input data to hash
): CttEVMStatus
```

**Description:** Computes the SHA-256 hash of the input data.

**Buffer requirements:**

| Buffer   | Size           | Description          |
|----------|----------------|----------------------|
| `r`      | exactly 32 bytes | Output digest       |
| `inputs` | arbitrary       | Input message        |

**Possible status codes:**
- `cttEVM_Success` — digest computed successfully
- `cttEVM_InvalidOutputSize` — `r.len != 32`

**Usage example:**
```nim
import constantine/ethereum_evm_precompiles

# Output buffer: 32 bytes = 256 bits
var digest: array[32, byte]

# Input message as bytes
let message: seq[byte] = @[0x61'u8, 0x62, 0x63]  # "abc"

# Call: function name first, arguments inside parentheses
let status = eth_evm_sha256(digest, message)
if status == cttEVM_Success:
  echo "SHA-256 digest computed"
```

---

### `eth_evm_ripemd160`

**EVM Address:** 3  
**Signature:**
```nim
func eth_evm_ripemd160*(
  r:      var openArray[byte],   # Output buffer: MUST be exactly 32 bytes
  inputs: openArray[byte]        # Arbitrary input data
): CttEVMStatus
```

**Description:** Computes the RIPEMD-160 hash of the input. The 20-byte digest is **right-aligned** in the 32-byte output buffer: the first 12 bytes are zero, the last 20 bytes contain the hash.

**Output buffer layout:**
```
[00 00 00 00 00 00 00 00 00 00 00 00 | XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX]
 ← 12 zero bytes (padding)          → ← 20-byte RIPEMD-160 digest                               →
```

**Buffer requirements:**

| Buffer   | Size            | Description                      |
|----------|-----------------|----------------------------------|
| `r`      | exactly 32 bytes | Output digest (12 zeros + 20 bytes) |
| `inputs` | arbitrary        | Input message                    |

**Possible status codes:**
- `cttEVM_Success`
- `cttEVM_InvalidOutputSize`

---

## Modular Exponentiation

### `eth_evm_modexp_result_size`

**Signature:**
```nim
func eth_evm_modexp_result_size*(
  size:   var uint64,          # Output: required result buffer size (in bytes)
  inputs: openArray[byte]      # MODEXP-encoded input data
): CttEVMStatus {.noInline, tags:[Alloca, Vartime].}
```

**Description:** Helper function. Computes the required output buffer size for `eth_evm_modexp` **before** calling it. The size equals `modulusLen` — the modulus length in bytes, extracted from the first 96 bytes of the input.

**Layout of the first 96 bytes of `inputs`:**
```
inputs[0..31]  — baseLen     (32 bytes, big-endian uint256): base length in bytes
inputs[32..63] — exponentLen (32 bytes, big-endian uint256): exponent length in bytes
inputs[64..95] — modulusLen  (32 bytes, big-endian uint256): modulus length → required output size
```

**Usage example:**
```nim
var resultSize: uint64
let inputData: seq[byte] = buildModexpInput(base, exponent, modulus)

# Step 1: determine required output buffer size
let sizeStatus = eth_evm_modexp_result_size(resultSize, inputData)
if sizeStatus != cttEVM_Success:
  echo "Size error: ", sizeStatus
  return

# Step 2: allocate the result buffer
var resultBuf = newSeq[byte](resultSize)

# Step 3: perform the computation
let status = eth_evm_modexp(resultBuf, inputData)
```

---

### `eth_evm_modexp`

**EVM Address:** 5 (EIP-198)  
**Signature:**
```nim
func eth_evm_modexp*(
  r:      var openArray[byte],   # Output buffer: MUST be exactly modulusLen bytes
  inputs: openArray[byte]        # Encoded input data
): CttEVMStatus {.noInline, tags:[Alloca, Vartime].}
```

**Description:** Computes `base ^ exponent mod modulus`. Implements EIP-198 / Yellow Paper Appendix E.

**Full input format:**
```
inputs[0..31]                         — baseLen     (32 bytes, big-endian uint256)
inputs[32..63]                        — exponentLen (32 bytes, big-endian uint256)
inputs[64..95]                        — modulusLen  (32 bytes, big-endian uint256)
inputs[96 .. 96+baseLen-1]            — base        (baseLen bytes, big-endian)
inputs[96+baseLen .. ...+expLen-1]    — exponent    (exponentLen bytes, big-endian)
inputs[... .. ...+modulusLen-1]       — modulus     (modulusLen bytes, big-endian)
```

> If the input is shorter than expected, it is implicitly zero-padded on the right.

**Special cases (returns zeros or one):**
- `modulusLen == 0` → empty result
- `exponentLen == 0` → result = `1` (x⁰ = 1, 0⁰ = 1)
- `baseLen == 0` → result = `0`
- all bytes after the length headers are zero → modulus is zero → result = `0`

**Buffer requirements:**

| Buffer   | Size                              |
|----------|-----------------------------------|
| `r`      | exactly `modulusLen` bytes        |
| `inputs` | arbitrary (zero-padded as needed) |

**Possible status codes:**
- `cttEVM_Success`
- `cttEVM_InvalidInputSize` — lengths exceed CPU addressable range
- `cttEVM_InvalidOutputSize` — `r.len != modulusLen`

> **Security note:** the function stack-allocates up to `(16+1) * modulusLen` bytes. EVM gas pricing must limit large inputs to prevent stack overflow.

---

## BN254 Curve (alt_bn128)

BN254 (also known as alt_bn128 or bn256 in Ethereum) is a Barreto-Naehrig twisted curve introduced in Ethereum at the Byzantium hard fork. Group order: 254 bits.

**BN254 coordinate encoding:** each coordinate is a 32-byte big-endian integer in `[0, p)` where `p` is the prime field modulus.

### `eth_evm_bn254_g1add`

**EVM Address:** 6 (EIP-196)  
**Signature:**
```nim
func eth_evm_bn254_g1add*(
  r:      var openArray[byte],   # Output buffer: MUST be 64 bytes
  inputs: openArray[byte]        # [Px(32) | Py(32) | Qx(32) | Qy(32)]
): CttEVMStatus
```

**Description:** Adds two points on the BN254 G1 curve: `R = P + Q`.

**Input layout:**
```
inputs[0..31]   — Px: X coordinate of point P (32 bytes, big-endian)
inputs[32..63]  — Py: Y coordinate of point P (32 bytes, big-endian)
inputs[64..95]  — Qx: X coordinate of point Q (32 bytes, big-endian)
inputs[96..127] — Qy: Y coordinate of point Q (32 bytes, big-endian)
```
Total: 128 bytes. If `inputs.len < 128`, missing bytes are treated as zeros. Extra bytes beyond 128 are ignored.

**Output layout:**
```
r[0..31]  — Rx: X coordinate of result R (32 bytes, big-endian)
r[32..63] — Ry: Y coordinate of result R (32 bytes, big-endian)
```

> **Subgroup:** the BN254 G1 cofactor is 1, so no subgroup membership check is performed.

> **Point at infinity** is encoded as `(0, 0)`.

**Possible status codes:**
- `cttEVM_Success`
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`
- `cttEVM_PointNotOnCurve`

**Usage example:**
```nim
# Input: points P and Q concatenated
var inputBuf: array[128, byte]
# ... fill P and Q coordinates ...

var resultBuf: array[64, byte]
let status = eth_evm_bn254_g1add(resultBuf, inputBuf)
if status == cttEVM_Success:
  # resultBuf holds R = P + Q coordinates
  echo "Point addition succeeded"
```

---

### `eth_evm_bn254_g1mul`

**EVM Address:** 7 (EIP-196)  
**Signature:**
```nim
func eth_evm_bn254_g1mul*(
  r:      var openArray[byte],   # Output buffer: MUST be 64 bytes
  inputs: openArray[byte]        # [Px(32) | Py(32) | s(32)]
): CttEVMStatus
```

**Description:** Scalar multiplication on BN254 G1: `R = [s]P`.

**Input layout:**
```
inputs[0..31]  — Px: X coordinate of P (32 bytes, big-endian)
inputs[32..63] — Py: Y coordinate of P (32 bytes, big-endian)
inputs[64..95] — s:  scalar in [0, 2²⁵⁶) (32 bytes, big-endian)
```
Total: 96 bytes. Zero-padded at the right if shorter; truncated if longer.

> **Optimization:** the scalar `s` is allowed to exceed the group order `r`. The implementation reduces `s mod r` using low-level Montgomery conversion (`getMont`) and then applies the windowed endomorphism scalar multiplication method — 31.5% faster than plain windowed multiplication.

**Possible status codes:**
- `cttEVM_Success`
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`
- `cttEVM_PointNotOnCurve`

---

### `eth_evm_bn254_ecpairingcheck`

**EVM Address:** 8 (EIP-197, EIP-1108)  
**Signature:**
```nim
func eth_evm_bn254_ecpairingcheck*(
  r:      var openArray[byte],   # Output buffer: MUST be 32 bytes
  inputs: openArray[byte]        # Array of pairs [(P0,Q0), (P1,Q1), ..., (Pk,Qk)]
): CttEVMStatus
```

**Description:** Checks the bilinear pairing product equality:
```
e(P₀, Q₀) · e(P₁, Q₁) · ... · e(Pₖ, Qₖ) == 1
```
Returns `1` (as uint256 big-endian) if the equality holds, otherwise `0`.

**Single pair format (192 bytes):**
```
[Px(32) | Py(32) | Qx_im(32) | Qx_re(32) | Qy_im(32) | Qy_re(32)]
```

> **Important:** per EIP-197, Fp2 coordinates are encoded in **reversed order**: `(a, b)` means `a·𝑖 + b` instead of the standard `a + 𝑖·b`. This differs from BLS12-381.

**Input:** `N` pairs of 192 bytes each. `inputs.len` must be divisible by 192.

> **Empty input:** per the specification, an empty input is valid and returns `1`.

**Algorithm:**
1. Parse and validate each pair (Pᵢ, Qᵢ).
2. Accumulate via `MillerAccumulator` (Miller loop).
3. Apply final exponentiation.
4. Compare result with the GT identity element (one in Fp12).

**Possible status codes:**
- `cttEVM_Success`
- `cttEVM_InvalidInputSize`
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`
- `cttEVM_PointNotOnCurve`
- `cttEVM_PointNotInSubgroup`

---

## BLS12-381 Curve (EIP-2537)

BLS12-381 is a 381-bit Barreto-Lynn-Scott curve with embedding degree 12, used in Ethereum 2.0 (Beacon Chain) for BLS signatures. It has two subgroups:

- **G1**: points over the base field `Fp` (381-bit prime modulus `p`). Each coordinate is encoded in 64 bytes (48 significant bytes + 16 leading zero bytes).
- **G2**: points over the quadratic extension `Fp2 = Fp[√-1]`. Each coordinate is a pair `(c0, c1)` of Fp elements, encoded in 128 bytes.

**BLS12-381 coordinate encoding (EIP-2537):**
```
64-byte block = [00...00 (16 bytes) | coordinate (48 bytes, big-endian)]
                 ← high-order pad  → ← significant data →
```
The first 16 bytes **must** be zero.

### `eth_evm_bls12381_g1add`

**EIP-2537**  
**Signature:**
```nim
func eth_evm_bls12381_g1add*(
  r:      var openArray[byte],   # Output buffer: MUST be 128 bytes
  inputs: openArray[byte]        # MUST be 256 bytes: [Px(64) | Py(64) | Qx(64) | Qy(64)]
): CttEVMStatus
```

**Description:** Adds two G1 points on BLS12-381: `R = P + Q`.

**Input layout (256 bytes):**
```
inputs[0..63]    — Px (64 bytes: 16 zeros + 48 bytes)
inputs[64..127]  — Py (64 bytes)
inputs[128..191] — Qx (64 bytes)
inputs[192..255] — Qy (64 bytes)
```

**Output layout (128 bytes):**
```
r[0..63]   — Rx (64 bytes)
r[64..127] — Ry (64 bytes)
```

> **Note:** subgroup membership is **not checked** for addition (per spec). Jacobian (vartime) formulas are used because complete projective formulas may produce incorrect results outside the subgroup.

**Possible status codes:**
- `cttEVM_Success`
- `cttEVM_InvalidInputSize`
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`
- `cttEVM_PointNotOnCurve`

---

### `eth_evm_bls12381_g2add`

**EIP-2537**  
**Signature:**
```nim
func eth_evm_bls12381_g2add*(
  r:      var openArray[byte],   # Output buffer: MUST be 256 bytes
  inputs: openArray[byte]        # MUST be 512 bytes
): CttEVMStatus
```

**Description:** Adds two G2 points on BLS12-381: `R = P + Q`. G2 points have coordinates in `Fp2 = Fp[𝑖]` where `𝑖 = √-1`. Each coordinate `(a + 𝑖·b)` is encoded as two 64-byte blocks.

**Input layout (512 bytes):**
```
inputs[0..63]    — Px.c0 = Re(Px) — real part of P's X coordinate
inputs[64..127]  — Px.c1 = Im(Px) — imaginary part of P's X coordinate
inputs[128..191] — Py.c0 = Re(Py)
inputs[192..255] — Py.c1 = Im(Py)
inputs[256..319] — Qx.c0
inputs[320..383] — Qx.c1
inputs[384..447] — Qy.c0
inputs[448..511] — Qy.c1
```

**Output layout (256 bytes):** mirrors the first point's input structure.

> Subgroup check is not performed (same as G1ADD).

---

### `eth_evm_bls12381_g1mul`

**EIP-2537**  
**Signature:**
```nim
func eth_evm_bls12381_g1mul*(
  r:      var openArray[byte],   # Output buffer: MUST be 128 bytes
  inputs: openArray[byte]        # MUST be 160 bytes: [Px(64) | Py(64) | s(32)]
): CttEVMStatus
```

**Description:** Scalar multiplication on G1: `R = [s]P`.

**Input layout (160 bytes):**
```
inputs[0..63]    — Px (64 bytes)
inputs[64..127]  — Py (64 bytes)
inputs[128..159] — s: scalar in [0, 2²⁵⁶) (32 bytes, big-endian)
```

> **Optimization:** the scalar is reduced modulo the group order (`s mod r`) via `getMont`, then the windowed endomorphism method is applied. **Subgroup check is performed** (`checkSubgroup = true`) for G1 scalar multiplication.

---

### `eth_evm_bls12381_g2mul`

**EIP-2537**  
**Signature:**
```nim
func eth_evm_bls12381_g2mul*(
  r:      var openArray[byte],   # Output buffer: MUST be 256 bytes
  inputs: openArray[byte]        # MUST be 288 bytes: [Px(128) | Py(128) | s(32)]
): CttEVMStatus
```

**Description:** Scalar multiplication on G2: `R = [s]P`.

**Input layout (288 bytes):**
```
inputs[0..63]    — Px.c0
inputs[64..127]  — Px.c1
inputs[128..191] — Py.c0
inputs[192..255] — Py.c1
inputs[256..287] — s (32 bytes, big-endian)
```

> Subgroup check and scalar reduction are performed (same as G1MUL).

---

### `eth_evm_bls12381_g1msm`

**EIP-2537**  
**Signature:**
```nim
func eth_evm_bls12381_g1msm*(
  r:      var openArray[byte],   # Output buffer: MUST be 128 bytes
  inputs: openArray[byte]        # N pairs [(Pᵢx(64) | Pᵢy(64) | sᵢ(32))], N≥1, multiple of 160
): CttEVMStatus
```

**Description:** Multi-Scalar Multiplication (MSM) on G1: `R = Σ[sᵢ]Pᵢ`. More efficient than N separate G1MUL calls due to the Pippenger algorithm.

**Input layout:** multiple of 160 bytes. Each 160-byte chunk is one pair `(Pᵢ, sᵢ)`:
```
i*160 + 0  .. i*160 + 63  — Pᵢx
i*160 + 64 .. i*160 + 127 — Pᵢy
i*160 + 128.. i*160 + 159 — sᵢ
```

**Memory management:** points and coefficients are allocated on the **heap** (`allocHeapArrayAligned` with 64-byte alignment for SIMD). Memory is freed via `freeHeapAligned` after computation.

**Possible status codes:**
- `cttEVM_Success`
- `cttEVM_InvalidInputSize` — empty input or not a multiple of 160
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`
- `cttEVM_PointNotOnCurve`

---

### `eth_evm_bls12381_g2msm`

**EIP-2537**  
**Signature:**
```nim
func eth_evm_bls12381_g2msm*(
  r:      var openArray[byte],   # Output buffer: MUST be 256 bytes
  inputs: openArray[byte]        # N pairs [(Pᵢ(256 bytes) | sᵢ(32 bytes))], multiple of 288
): CttEVMStatus
```

**Description:** Multi-Scalar Multiplication on G2: `R = Σ[sᵢ]Pᵢ`. Each pair is 288 bytes.

**Input layout:** multiple of 288 bytes. Each 288-byte chunk:
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
**Signature:**
```nim
func eth_evm_bls12381_pairingcheck*(
  r:      var openArray[byte],   # Output buffer: MUST be 32 bytes
  inputs: openArray[byte]        # N pairs [(Pᵢ(128) | Qᵢ(256))], multiple of 384, N≥1
): CttEVMStatus
```

**Description:** Pairing check for BLS12-381:
```
∏ e(Pᵢ, Qᵢ) == 1  →  r = [0...01]
∏ e(Pᵢ, Qᵢ) ≠ 1  →  r = [0...00]
```

**Single pair format (384 bytes):**
```
pos +   0 .. pos +  63 — Pᵢx    (G1, 64 bytes)
pos +  64 .. pos + 127 — Pᵢy    (G1, 64 bytes)
pos + 128 .. pos + 191 — Qᵢx.c0 (G2, 64 bytes)
pos + 192 .. pos + 255 — Qᵢx.c1 (G2, 64 bytes)
pos + 256 .. pos + 319 — Qᵢy.c0 (G2, 64 bytes)
pos + 320 .. pos + 383 — Qᵢy.c1 (G2, 64 bytes)
```

> **Difference from BN254:** in BLS12-381, Fp2 coordinates are encoded as `(c0 + 𝑖·c1)` in natural order, not reversed as in EIP-197.

> **Empty input is invalid** (unlike BN254). Returns `cttEVM_InvalidInputSize`.

> **Subgroup membership is checked** for both P (G1) and Q (G2) points.

**Possible status codes:**
- `cttEVM_Success`
- `cttEVM_InvalidInputSize`
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`
- `cttEVM_PointNotOnCurve`
- `cttEVM_PointNotInSubgroup`

---

### `eth_evm_bls12381_map_fp_to_g1`

**EIP-2537**  
**Signature:**
```nim
func eth_evm_bls12381_map_fp_to_g1*(
  r:      var openArray[byte],   # Output buffer: MUST be 128 bytes
  inputs: openArray[byte]        # MUST be 64 bytes: a field element in Fp
): CttEVMStatus
```

**Description:** Deterministic mapping of a base field element `u ∈ Fp` to a G1 point using Simplified SWU (`mapToCurve_sswu`). The result lies in the correct subgroup (cofactor is cleared via `clearCofactor`).

**Input layout (64 bytes):**
```
inputs[0..15]  — zeros (16 bytes, must be 0x00)
inputs[16..63] — field element u ∈ [0, p), big-endian (48 bytes)
```

**Possible status codes:**
- `cttEVM_Success`
- `cttEVM_InvalidInputSize`
- `cttEVM_InvalidOutputSize`
- `cttEVM_IntLargerThanModulus`

---

### `eth_evm_bls12381_map_fp2_to_g2`

**EIP-2537**  
**Signature:**
```nim
func eth_evm_bls12381_map_fp2_to_g2*(
  r:      var openArray[byte],   # Output buffer: MUST be 256 bytes
  inputs: openArray[byte]        # MUST be 128 bytes: a field element in Fp2
): CttEVMStatus
```

**Description:** Maps an extension field element `u ∈ Fp2` to a G2 point. The input element `u = u.c0 + 𝑖·u.c1` is a pair of Fp elements.

**Input layout (128 bytes):**
```
inputs[0..15]   — zeros (16 bytes)
inputs[16..63]  — u.c0 (48 bytes, big-endian): Re(u)
inputs[64..79]  — zeros (16 bytes)
inputs[80..127] — u.c1 (48 bytes, big-endian): Im(u)
```

---

## KZG and EIP-4844 Protocol

### `eth_evm_kzg_point_evaluation`

**EIP-4844** (proto-danksharding)  
**Signature:**
```nim
func eth_evm_kzg_point_evaluation*(
  ctx:   ptr EthereumKZGContext,  # Pointer to an initialized KZG context (trusted setup)
  r:     var openArray[byte],     # Output buffer: MUST be 64 bytes
  input: openArray[byte]          # MUST be 192 bytes
): CttEVMStatus
```

**Description:** Verifies a KZG proof of polynomial evaluation: `p(z) = y`, where `p(x)` is the polynomial corresponding to the KZG commitment. Also verifies that the commitment matches the `versioned_hash`.

**Input layout (192 bytes):**
```
input[0..31]    — versioned_hash (32 bytes): 0x01 || SHA256(commitment)[1:]
input[32..63]   — z (32 bytes, big-endian): evaluation point in Fr[BLS12_381]
input[64..95]   — y (32 bytes, big-endian): expected polynomial value at z
input[96..143]  — commitment (48 bytes): KZG commitment (compressed G1 point)
input[144..191] — proof (48 bytes): KZG proof (compressed G1 point)
```

**Computing versioned_hash:**
```
versioned_hash = 0x01 || SHA256(commitment)[1:]
```
The first byte of the SHA-256 digest is replaced with the version tag `0x01`.

**Output layout (64 bytes):**
```
r[0..31]  — FIELD_ELEMENTS_PER_BLOB (uint256, big-endian): number of field elements per blob
r[32..63] — BLS12-381 Fr field modulus (uint256, big-endian)
```

**Verification order:**
1. Check buffer sizes.
2. Recompute `versioned_hash` from `commitment` and compare with the input value.
3. Call `verify_kzg_proof(commitment, z, y, proof)`.
4. Write the constant output to `r`.

**Context initialization:**

```nim
import constantine/ethereum_evm_precompiles

# EthereumKZGContext, TrustedSetupFormat, new, delete are re-exported by this module
var ctx: ptr EthereumKZGContext

# Load context from a standard Ethereum trusted setup JSON file
let loadStatus = new(ctx, "trusted_setup.json", TrustedSetupFormat.json)

# ... use ctx for verification calls ...

# Free the context when done
delete(ctx)
```

---

## ECDSA: Public Key Recovery

### `eth_evm_ecrecover`

**EVM Address:** 1  
**Signature:**
```nim
func eth_evm_ecrecover*(
  r:     var openArray[byte],   # Output buffer: MUST be 32 bytes
  input: openArray[byte]        # MUST be 128 bytes
): CttEVMStatus
```

**Description:** Recovers the Ethereum address (last 20 bytes of keccak256 of the public key) from a message hash and an ECDSA signature on the Secp256k1 curve.

**Input layout (128 bytes):**
```
input[0..31]   — msgHash: keccak256 hash of the signed message (32 bytes)
input[32..63]  — v: parity of the Y coordinate of the recovered point (32 bytes)
input[64..95]  — r: signature component r, scalar Fr[Secp256k1] (32 bytes, big-endian)
input[96..127] — s: signature component s, scalar Fr[Secp256k1] (32 bytes, big-endian)
```

**Valid values of `v`:**

| Value | Meaning                          |
|-------|----------------------------------|
| `0`   | even Y coordinate                |
| `1`   | odd Y coordinate                 |
| `27`  | even Y (legacy Ethereum encoding) |
| `28`  | odd Y (legacy Ethereum encoding)  |

> Bytes `input[32..62]` (the first 31 bytes of the `v` block) must be zero. Otherwise returns `cttEVM_MalformedSignature`.

**Output layout (32 bytes):**
```
r[0..11]  — zeros (12 bytes)
r[12..31] — Ethereum address: last 20 bytes of keccak256(uncompressed public key)
```

**Algorithm:**
1. Build `msgHash` as a scalar in `Fr[Secp256k1]`.
2. Validate `v` (first 31 bytes = 0, last byte ∈ {0, 1, 27, 28}).
3. Parse `r`, `s` into a `Signature` struct.
4. Recover public key via `recoverPubkeyFromDigest`.
5. Serialize the uncompressed key `(x, y)` as 64 bytes.
6. Compute keccak256 digest and take the last 20 bytes.

**Compatibility:** follows Geth (go-ethereum) behavior.

**Usage example:**
```nim
var inputBuf: array[128, byte]
# Fill msgHash, v, r, s fields ...

var addrBuf: array[32, byte]
let status = eth_evm_ecrecover(addrBuf, inputBuf)

if status == cttEVM_Success:
  # addrBuf[12..31] contains the recovered Ethereum address (20 bytes)
  echo "Address recovered"
elif status == cttEVM_MalformedSignature:
  echo "Malformed signature"
```

---

## Re-exported Types and API

The module re-exports the following symbols from its dependencies:

```nim
# From ethereum_eip4844_kzg:
export EthereumKZGContext    # Type: opaque KZG trusted setup context
export TrustedSetupFormat    # Enum: trusted setup file format (json, binary)
export TrustedSetupStatus    # Enum: trusted setup loading status
export new                   # proc new(ctx: var ptr EthereumKZGContext, ...): TrustedSetupStatus
export new_with_precompute   # proc: create context with precomputed values
export delete                # proc delete(ctx: ptr EthereumKZGContext)

# From hashes (not technically a precompile, but re-exported for convenience):
export hashes                # Module: sha256, keccak256, ripemd160 and their methods
```

**Using the KZG context:**
```nim
import constantine/ethereum_evm_precompiles

var ctx: ptr EthereumKZGContext

# Option 1: from JSON file (standard Ethereum trusted setup)
let loadStatus = new(ctx, "trusted_setup.json", TrustedSetupFormat.json)

# Option 2: with precomputation (faster verification, more memory)
let loadStatus2 = new_with_precompute(ctx, "trusted_setup.json", TrustedSetupFormat.json)

# Release the context when done
delete(ctx)
```

---

## Internal Helper Functions

The following functions are used internally and are not part of the public API:

### `parseEip2537`

```nim
proc parseEip2537(
  dst: var Fp[BLS12_381],
  src: openArray[byte]   # MUST be 64 bytes
): CttEvmStatus {.inline.}
```

Parses a field coordinate according to EIP-2537 rules: 16 mandatory zero bytes followed by 48 bytes of value in big-endian. Verifies the value is strictly less than the field prime.

### `parseRawUint` (overloaded)

```nim
func parseRawUint(dst: var Fp[BLS12_381],   src: openarray[byte]): CttEVMStatus
func parseRawUint(dst: var Fp[BN254_Snarks], src: openarray[byte]): CttEVMStatus
```

Curve-specific field element parsing. For BN254: direct `unmarshal` of 32 bytes. For BLS12-381: delegates to `parseEip2537`.

### `fromRawCoords` (generic family)

A family of overloaded generic functions for parsing curve points from raw byte coordinates:

```nim
# Affine G1 point (Fp)
func fromRawCoords[Name, G](
  dst: var EC_ShortW_Aff[Fp[Name], G],
  x, y: openarray[byte],
  checkSubgroup: bool): CttEVMStatus

# Affine G2 point (Fp2)
func fromRawCoords[Name](
  dst: var EC_ShortW_Aff[Fp2[Name], G2],
  x0, x1, y0, y1: openarray[byte],
  checkSubgroup: bool): CttEVMStatus

# Jacobian G1 point (Fp) — converts through affine
func fromRawCoords[Name, G](
  dst: var EC_ShortW_Jac[Fp[Name], G],
  x, y: openarray[byte],
  checkSubgroup: bool): CttEVMStatus

# Jacobian G2 point (Fp2)
func fromRawCoords[Name, G](
  dst: var EC_ShortW_Jac[Fp2[Name], G],
  x0, x1, y0, y1: openarray[byte],
  checkSubgroup: bool): CttEVMStatus
```

Validation sequence inside each variant:
1. Parse coordinates via `parseRawUint`.
2. Handle the point at infinity `(0, 0)`.
3. Check curve membership via `isOnCurve`.
4. Optionally: check subgroup membership via `isInSubgroup`.

### `kzg_to_versioned_hash` (internal)

```nim
proc kzg_to_versioned_hash(
  r: var array[32, byte],
  commitment_bytes: array[48, byte])
```

Computes `versioned_hash = 0x01 || SHA256(commitment)[1:]` as required by EIP-4844.

---

## Function Quick Reference Table

| Function | EVM Addr | EIP | Input (bytes) | Output (bytes) |
|----------|----------|-----|---------------|----------------|
| `eth_evm_sha256` | 2 | — | arbitrary | 32 |
| `eth_evm_ripemd160` | 3 | — | arbitrary | 32 |
| `eth_evm_modexp_result_size` | — | 198 | arbitrary | `uint64` (via `var`) |
| `eth_evm_modexp` | 5 | 198 | arbitrary | `modulusLen` |
| `eth_evm_bn254_g1add` | 6 | 196 | 128 (padded) | 64 |
| `eth_evm_bn254_g1mul` | 7 | 196 | 96 (padded) | 64 |
| `eth_evm_bn254_ecpairingcheck` | 8 | 197/1108 | N×192 | 32 |
| `eth_evm_bls12381_g1add` | — | 2537 | 256 | 128 |
| `eth_evm_bls12381_g2add` | — | 2537 | 512 | 256 |
| `eth_evm_bls12381_g1mul` | — | 2537 | 160 | 128 |
| `eth_evm_bls12381_g2mul` | — | 2537 | 288 | 256 |
| `eth_evm_bls12381_g1msm` | — | 2537 | N×160, N≥1 | 128 |
| `eth_evm_bls12381_g2msm` | — | 2537 | N×288, N≥1 | 256 |
| `eth_evm_bls12381_pairingcheck` | — | 2537 | N×384, N≥1 | 32 |
| `eth_evm_bls12381_map_fp_to_g1` | — | 2537 | 64 | 128 |
| `eth_evm_bls12381_map_fp2_to_g2` | — | 2537 | 128 | 256 |
| `eth_evm_kzg_point_evaluation` | — | 4844 | 192 | 64 |
| `eth_evm_ecrecover` | 1 | — | 128 | 32 |

---

## Security Notes

**Constant-time:** most cryptographic operations are implemented in constant time (independent of secret data) to resist timing side-channel attacks. Functions with a `_vartime` suffix run in variable time and must not be used with secret data.

**No exceptions:** the `{.push raises: [].}` directive guarantees that no function throws an exception. All errors are reported via `CttEVMStatus`.

**C FFI:** all public functions are exported with a `ctt_` prefix via the `{.libPrefix: prefix_ffi.}` pragma. For example, `eth_evm_sha256` is accessible from C as `ctt_eth_evm_sha256`.

**Input validation:** always check `CttEVMStatus` before reading from the output buffer. On error, the contents of `r` are undefined.

**Nim version:** requires Nim 2.2.0 or later.

**Compiler:** Clang 14+ is recommended for optimal performance and correct generation of Intel-syntax assembly code.

**Subgroup checks summary:**

| Operation       | Curve    | G1 subgroup check | G2 subgroup check |
|----------------|----------|-------------------|-------------------|
| G1ADD / G2ADD  | BN254    | no (cofactor = 1) | yes               |
| G1ADD / G2ADD  | BLS12-381 | no (per spec)    | no (per spec)     |
| G1MUL / G2MUL  | BLS12-381 | **yes**          | **yes**           |
| G1MSM / G2MSM  | BLS12-381 | **yes**          | **yes**           |
| PAIRINGCHECK   | BN254    | no (cofactor = 1) | **yes**           |
| PAIRINGCHECK   | BLS12-381 | **yes**          | **yes**           |
