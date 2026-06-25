# Reference: `eth_eip7594_peerdas.nim`

> Constantine Library · EIP-7594 PeerDAS — Data Availability Sampling

---

## Table of Contents

1. [Overview and Concepts](#1-overview-and-concepts)
2. [Imported Dependencies](#2-imported-dependencies)
3. [Constants](#3-constants)
4. [Data Types](#4-data-types)
5. [Internal Serialization Functions](#5-internal-serialization-functions)
6. [Helper Functions](#6-helper-functions)
7. [Public API](#7-public-api)
8. [Algorithms and Optimizations](#8-algorithms-and-optimizations)
9. [Error Codes](#9-error-codes)
10. [Full Data Flow Diagram](#10-full-data-flow-diagram)
11. [Usage Examples](#11-usage-examples)

---

## 1. Overview and Concepts

This module implements the Ethereum-specific serialization layer for **PeerDAS** (Peer Data Availability Sampling) as specified in [EIP-7594](https://eips.ethereum.org/EIPS/eip-7594).

### Key Concepts

| Concept | Size | Description |
|---------|------|-------------|
| **Blob** | 128 KB | 4096 field elements × 32 bytes — raw data payload |
| **Cell** | 2 048 bytes | 64 field elements — minimum unit of sampling |
| **Extended Blob** | 256 KB | 8192 field elements after Reed-Solomon extension (2× redundancy) |
| **Cells per Blob** | 64 | 4096 ÷ 64 |
| **Cells per Extended Blob** | 128 | 8192 ÷ 64 |
| **KZG Proof** | 48 bytes | Compressed G1 point — cryptographic availability proof |
| **KZG Commitment** | 48 bytes | Compressed G1 point — polynomial commitment |

### Blob Lifecycle in PeerDAS

```
Blob (128 KB)
  │
  ▼ blob_to_field_polynomial()
Polynomial (Lagrange form, bit-reversed, 4096 points)
  │
  ├──[IFFT]──▶ Monomial form (coefficient form, 4096 coefficients)
  │                │
  │                ├──▶ FK20: compute 128 KZG proofs
  │                │
  │                └──▶ Shift + FFT: compute cells 64–127
  │
  └──[direct copy]──▶ cells 0–63  (identical to the original blob!)
        │
        ▼
128 Cell[2048 bytes] + 128 KZGProof[48 bytes]
```

---

## 2. Imported Dependencies

```nim
import
  # Algebraic structures and named curves
  constantine/named/algebras,
  # Elliptic curves, extension fields, polynomials, and FFT over fields
  constantine/math/[ec_shortweierstrass, extension_fields,
                    polynomials/polynomials, polynomials/fft_fields],
  # Finite field arithmetic and Montgomery limbs
  constantine/math/arithmetic/[finite_fields, limbs_montgomery],
  # I/O helpers for BigInts and finite field elements
  constantine/math/io/[io_bigints, io_fields],
  # Platform primitives, array views, aligned heap allocators
  constantine/platforms/[primitives, views, allocs],
  # Ethereum KZG Structured Reference String (SRS / trusted setup)
  constantine/commitments_setups/ethereum_kzg_srs,
  # EIP-4844 KZG internals — {.all.} exposes private symbols
  constantine/ethereum_eip4844_kzg {.all.},
  # Serialization status codes and BLS12-381 codecs
  constantine/serialization/[codecs_status_codes, codecs_bls12_381, endians],
  # Generic PeerDAS implementation (polynomial recovery etc.)
  constantine/data_availability_sampling/eth_peerdas,
  # Multi-point KZG proof generation (FK20 algorithm)
  constantine/commitments/kzg_multiproofs,
  # Hash functions (SHA-256 for Fiat-Shamir)
  constantine/hashes

import
  # Compile-time only: type introspection
  std/typetraits
```

> **Note:** `{.all.}` when importing `ethereum_eip4844_kzg` grants access to private symbols — in particular `blob_to_field_polynomial` and `bls_field_to_bytes`.

---

## 3. Constants

```nim
const
  # Domain separator for the Fiat-Shamir transcript used in batch verification.
  # Exactly 16 ASCII bytes — fits in one SHA-256 block update.
  RANDOM_CHALLENGE_KZG_CELL_BATCH_DOMAIN* = asBytes"RCKZGCBATCH__V1_"
```

### Re-exported Constants from `ethereum_kzg_srs`

| Name | Value | Description |
|------|-------|-------------|
| `FIELD_ELEMENTS_PER_CELL` | 64 | Fr field elements per cell |
| `CELLS_PER_EXT_BLOB` | 128 | Cells in the extended blob |
| `BYTES_PER_CELL` | 2 048 | Bytes per cell (64 × 32) |
| `CELLS_PER_BLOB` | 64 | Cells in the original (non-extended) blob |

These are re-exported via:
```nim
export ethereum_kzg_srs.FIELD_ELEMENTS_PER_CELL
export ethereum_kzg_srs.CELLS_PER_EXT_BLOB
export ethereum_kzg_srs.BYTES_PER_CELL
export ethereum_kzg_srs.CELLS_PER_BLOB
```

---

## 4. Data Types

### `Cell`

```nim
type Cell* = array[BYTES_PER_CELL, byte]
  ## The fundamental unit of Data Availability Sampling.
  ## Contains 64 field elements (2048 bytes) in big-endian serialized form.
  ## Each cell is verifiable with a single KZG proof.
  ##
  ## Memory layout:
  ##   [elem_0_byte_0 .. elem_0_byte_31 | elem_1_byte_0 .. elem_1_byte_31 | ... | elem_63_byte_31]
  ##    ◄──────── 32 bytes ────────────► ◄──────── 32 bytes ────────────►       ◄── 32 bytes ──►
  ##   total: 64 × 32 = 2048 bytes
```

### `CellIndex`

```nim
type CellIndex* = uint64
  ## Index of a cell within the extended blob.
  ## Valid range: [0, CELLS_PER_EXT_BLOB)  →  [0, 128)
  ## Used in the public API to identify the position of a cell.
```

### `CosetEvals`

```nim
type CosetEvals* = array[FIELD_ELEMENTS_PER_CELL, Fr[BLS12_381]]
  ## Internal representation of a cell's data:
  ## 64 polynomial evaluations at the points of a coset.
  ##
  ## A coset is a shifted subset of roots of unity:
  ##   { ω_shift · ω^0,  ω_shift · ω^1, ...,  ω_shift · ω^63 }
  ## where ω is a primitive 64th root of unity
  ##       ω_shift is the coset shift factor for this specific cell.
  ##
  ## Fr[BLS12_381] — element of the BLS12-381 scalar field:
  ##   r = 52435875175126190479447740508185965837690552500527637822603658699938581184513
```

### `KZGProofBytes`

```nim
type KZGProofBytes* = array[BYTES_PER_PROOF, byte]
  ## Serialized KZG proof: 48 bytes of a compressed G1 affine point.
  ##
  ## Format: first byte = 0x80 (compression flag), then 47 bytes of x-coordinate.
  ## Used at I/O boundaries and FFI interfaces.
  ##
  ## Conversion:
  ##   bytes → KZGProof : deserialize_g1_compressed(proof, bytes)
  ##   KZGProof → bytes : serialize_g1_compressed(bytes, proof)
  ##
  ## Internally the library uses EC_ShortW_Aff[Fp[BLS12_381], G1] (96 bytes).
```

### `Coset`

```nim
type Coset* = array[FIELD_ELEMENTS_PER_CELL, Fr[BLS12_381]]
  ## The evaluation domain for one cell — 64 coset points (roots of unity).
  ## Semantically equivalent to CosetEvals; used as an FFT domain type.
```

---

## 5. Internal Serialization Functions

These functions are not exported and form the `bytes ↔ Fr` conversion layer.

---

### `bytesToBlsField`

```nim
func bytesToBlsField(
    dst: var Fr[BLS12_381],   # [out] result — validated field element
    src: array[32, byte]      # [in]  32 bytes big-endian
): CttCodecScalarStatus
```

**Purpose:** Convert 32 untrusted bytes into a validated BLS12-381 scalar field element.

**Algorithm:**
1. Deserialize bytes into a BigInt via `deserialize_scalar`
2. Verify BigInt < r (curve order)
3. Convert to Montgomery form via `fromBig`

**Return status codes:**

| Status | Meaning |
|--------|---------|
| `cttCodecScalar_Success` | Success |
| `cttCodecScalar_Zero` | Success (zero element) |
| Others | Failure: bytes ≥ r |

**Example:**
```nim
var fieldElem: Fr[BLS12_381]
var rawBytes: array[32, byte]
# ... fill rawBytes ...
let status = bytesToBlsField(fieldElem, rawBytes)
if status != cttCodecScalar_Success:
  # handle error — bytes do not represent a valid scalar
  discard
```

---

### `cellToCosetEvals`

```nim
func cellToCosetEvals(
    evals: var array[FIELD_ELEMENTS_PER_CELL, Fr[BLS12_381]],  # [out] 64 field elements
    cell: openArray[byte]                                        # [in]  2048-byte cell
): cttEthKzgStatus
```

**Purpose:** Deserialize a cell (bytes) into an array of field elements (CosetEvals).

**Input data layout:**
```
cell[0..31]      → evals[0]   (32 bytes big-endian → Fr)
cell[32..63]     → evals[1]
...
cell[2016..2047] → evals[63]
```

**Return values:**
- `cttEthKzg_Success` — success
- `cttEthKzg_ScalarLargerThanCurveOrder` — one of the 32-byte chunks encodes a value ≥ r

---

### `cosetEvalsToCell`

```nim
func cosetEvalsToCell(
    cell: var openArray[byte],                                    # [out] 2048 bytes
    evals: array[FIELD_ELEMENTS_PER_CELL, Fr[BLS12_381]]        # [in]  64 field elements
)
```

**Purpose:** Serialize an array of field elements into cell bytes. The inverse of `cellToCosetEvals`.

**Algorithm:**
```
for each i in [0, 63]:
  marshal(chunk[32], evals[i], bigEndian)   # Montgomery form → big-endian bytes
  cell[i*32 .. i*32+31] := chunk
```

---

## 6. Helper Functions

### Template `?` (early return on error)

```nim
template `?`(status: cttEthKzgStatus): untyped {.dirty.}
  ## Analogous to Rust's ? operator.
  ## If status ≠ cttEthKzg_Success — immediately return status from the calling function.
  ## Enables clean chaining of fallible operations without nested if-blocks.
```

**Usage pattern:**
```nim
# Instead of:
let s1 = operationA()
if s1 != cttEthKzg_Success: return s1
let s2 = operationB()
if s2 != cttEthKzg_Success: return s2

# Write:
?operationA()
?operationB()
```

---

### `deduplicateCommitments`

```nim
func deduplicateCommitments(
    commitmentIdx: var openArray[int],                              # [out] unique commitment index per cell
    commitments: openArray[array[BYTES_PER_COMMITMENT, byte]],     # [in]  N commitments (possibly with duplicates)
    firstOccurrence: var openArray[int]                            # [out] index of first occurrence of each unique
): int  # returns the number of unique commitments
```

**Purpose:** Deduplicate KZG commitments before deserialization. Operates on raw bytes — significantly faster than working with deserialized EC points.

**Complexity:** O(N × M), where N = number of cells, M = number of unique blobs (≤ 64 per EIP-7594).

**Comparison with the Ethereum specification algorithm:**

| Approach | Complexity | Comparisons for N=1000, M=64 |
|----------|-----------|-------------------------------|
| Spec (Python `list.index`) | O(N²) | ~1 064 000 |
| This implementation | O(N × M) | ~32 000 |
| Speedup | — | ~33× |

**Invariants after the call:**
```
commitmentIdx[i]    ∈ [0, numUnique-1]         for each cell i
firstOccurrence[j]  ∈ [0, N-1]                 first-occurrence index of unique j
uniqueBuffer[0..numUnique-1] contains all distinct commitments
```

**Example:**
```nim
# Input:  commitments = [A, A, B, B]  (N = 4)
# Output: numUnique = 2
#         commitmentIdx    = [0, 0, 1, 1]
#         firstOccurrence  = [0, 2]
```

**Why byte-level comparison is fast:**

Cryptographic commitments have uniform byte distribution.  
The probability that two distinct commitments share the first byte is 1/256, so `memcmp` exits after just 1–2 bytes on average for non-matching entries.

---

### `compute_verify_cell_kzg_proof_batch_challenge`

```nim
func compute_verify_cell_kzg_proof_batch_challenge(
    commitments:       openArray[array[BYTES_PER_COMMITMENT, byte]],  # unique commitments
    commitment_indices: openArray[int],                                # which unique commitment each cell belongs to
    cell_indices:      openArray[CellIndex],                           # cell indices
    cosets_evals:      openArray[array[FIELD_ELEMENTS_PER_CELL, Fr[BLS12_381]]],  # cell data
    proofs:            openArray[KZGProofBytes]                        # proofs
): Fr[BLS12_381]  # returns random scalar r (Fiat-Shamir challenge)
```

**Purpose:** Compute the Fiat-Shamir challenge `r` for batch verification using a SHA-256 transcript.

**Algorithm (SHA-256 transcript):**
```
transcript.init()
transcript.update(RANDOM_CHALLENGE_KZG_CELL_BATCH_DOMAIN)  # "RCKZGCBATCH__V1_"
transcript.update(uint64(FIELD_ELEMENTS_PER_BLOB), bigEndian)
transcript.update(uint64(FIELD_ELEMENTS_PER_CELL), bigEndian)
transcript.update(uint64(numCommitments), bigEndian)
transcript.update(uint64(numCells), bigEndian)
for each commitment:
    transcript.update(commitment[48 bytes])
for each cell k:
    transcript.update(uint64(commitment_indices[k]), bigEndian)
    transcript.update(uint64(cell_indices[k]), bigEndian)
    for each eval in cosets_evals[k]:
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

**Purpose:** Core implementation for computing all 128 cells. Accepts the polynomial in two forms and fills all cells.

**Algorithm (half-FFT optimization):**

```
Step 1: Cells 0–63  → direct copy from poly_eval_brp  (O(1)!)
        The first half of the bit-reversed extended domain equals the original blob.

Step 2: Cells 64–127 → shift + FFT
        poly_coef_shifted := poly_coef_nat
        multiply coefficient k by w_8192^k   (shift in frequency domain)
        FFT(poly_coef_shifted) → cells[64..127]

Step 3: Serialization
        for each cell i: cosetEvalsToCell(cells[i], cells_evals[i])
```

**Complexity:**
- Cells 0–63:   O(1) — `copyMem` only
- Cells 64–127: O(N log N) — one FFT of size 4096
- Total: approximately **2× faster** than a full FFT(8192)

---

## 7. Public API

### `compute_cells`

```nim
func compute_cells*(
    ctx: ptr EthereumKZGContext,                    # Context with SRS and FFT descriptors
    cells: var array[CELLS_PER_EXT_BLOB, Cell],     # [out] 128 cells × 2048 bytes each
    blob: Blob                                       # [in]  128 KB blob
): cttEthKzgStatus
```

**Purpose:** Compute all 128 cells of the extended blob. Does **not** compute KZG proofs.

**Algorithm:**
1. `blob_to_field_polynomial` → `poly_eval_brp` (Lagrange, bit-reversed, heap, ~128 KB)
2. `lagrangeInterpolate` → `poly_coef_nat` (coefficient form, heap, ~128 KB)
3. `compute_cells_impl` → fill all 128 cells

> Both intermediate allocations are made on the heap (`allocHeapAligned`) rather than the stack, to avoid stack overflow (128 KB = 10% of Windows default 1 MB stack limit).

**Return values:** see [Error Codes](#9-error-codes).

**Example:**
```nim
import constantine/eth_eip7594_peerdas

let ctx: ptr EthereumKZGContext = ...  # initialize context
var blob: Blob                          # fill blob with data
var cells: array[CELLS_PER_EXT_BLOB, Cell]

let status = compute_cells(ctx, cells, blob)
if status != cttEthKzg_Success:
  echo "Cell computation error: ", status
```

---

### `compute_cells_and_kzg_proofs`

```nim
func compute_cells_and_kzg_proofs*(
    ctx: ptr EthereumKZGContext,
    cells: ptr UncheckedArray[Cell],           # [out] 128 cells
    proofs: ptr UncheckedArray[KZGProofBytes], # [out] 128 KZG proofs × 48 bytes
    blob: Blob                                  # [in]  128 KB blob
): cttEthKzgStatus
  {.libPrefix: prefix_eth_kzg, raises: [].}
```

**Purpose:** Compute all 128 cells **and** all 128 KZG proofs for a blob. This is the primary operation when publishing a blob.

**Algorithm (FK20):**

```
1. Deserialization
   blob_to_field_polynomial → poly_lagrange (Lagrange, bit-reversed)

2. IFFT: Lagrange → monomial form
   lagrangeInterpolate(poly_lagrange) → poly_monomial (4096 coefficients)

3. Cell computation
   compute_cells_impl(poly_lagrange, poly_monomial) → cells[0..127]

4. FK20: compute 128 proofs
   kzg_coset_prove(poly_monomial.coefs[0..4095], SRS) → proofsAff[128]
   bit_reversal_permutation(proofsAff)    # EIP-7594 ordering convention

5. Proof serialization
   for each i: serialize_g1_compressed(proofs[i], proofsAff[i])
```

**Preconditions:**
- `ctx`, `cells`, `proofs` must not be `nil`

**SRS precompute branching:**
```nim
case ctx.polyphaseSpectrumBank.kind:
of kNoPrecompute:
  # Use raw SRS points (slower, less memory)
  kzg_coset_prove(..., ctx.polyphaseSpectrumBank.rawPoints)
of kPrecompute:
  # Use precomputed polyphase spectrum bank (faster)
  kzg_coset_prove(..., ctx.polyphaseSpectrumBank.precompPoints)
```

**Example:**
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
    recovered_cells: ptr UncheckedArray[Cell],           # [out] all 128 cells
    recovered_proofs: ptr UncheckedArray[KZGProofBytes], # [out] all 128 proofs
    cell_indices: ptr UncheckedArray[CellIndex],          # [in]  known cell indices (strictly ascending)
    cells: ptr UncheckedArray[Cell],                      # [in]  known cells
    n: int                                                # [in]  number of known cells
): cttEthKzgStatus
  {.libPrefix: prefix_eth_kzg, raises: [].}
```

**Purpose:** Recover all 128 cells and 128 proofs from ≥ 64 known cells (50% threshold).

**Preconditions:**
- `n >= CELLS_PER_EXT_BLOB div 2` (i.e., ≥ 64 cells)
- `n <= CELLS_PER_EXT_BLOB` (no more than 128)
- `cell_indices` must be strictly ascending: `cell_indices[i-1] < cell_indices[i]`
- All indices in range `[0, CELLS_PER_EXT_BLOB)`
- All pointers must not be `nil`

**Algorithm:**

```
1. Validation: check n, uniqueness, and ordering of indices

2. Deserialization:
   for each of the n cells: cellToCosetEvals(cells[i]) → cosets_evals[i]

3. Polynomial coefficient recovery:
   recoverPolynomialCoeff(
     domain = 8192 roots of unity,
     coset_shift = SCALE_FACTOR = 5
   ) → poly_coeff[8192]

4. FFT(poly_coeff) → cells_evals[128][64]
   (size 8192: 128 cells × 64 elements each)

5. Cell serialization:
   for each i: cosetEvalsToCell(recovered_cells[i], cells_evals[i])

6. FK20: proof computation
   kzg_coset_prove(poly_coeff.coefs[0..4095]) → proofsAff[128]
   bit_reversal_permutation(proofsAff)
   serialize_g1_compressed(recovered_proofs[i], proofsAff[i])
```

**Error codes:**
- `cttEthKzg_InputsLengthsMismatch` — n out of range, nil pointers, or invalid indices
- `cttEthKzg_CellIndicesNotAscending` — indices are not strictly ascending
- `cttEthKzg_ScalarLargerThanCurveOrder` — cell bytes contain an invalid scalar

---

### `verify_cell_kzg_proof_batch`

```nim
func verify_cell_kzg_proof_batch*(
    ctx: ptr EthereumKZGContext,
    commitments_bytes: ptr UncheckedArray[array[BYTES_PER_COMMITMENT, byte]],  # [in] N commitments
    cell_indices: ptr UncheckedArray[CellIndex],                               # [in] N cell indices
    cells: ptr UncheckedArray[Cell],                                           # [in] N cells
    proofs_bytes: ptr UncheckedArray[KZGProofBytes],                          # [in] N proofs
    n: int,                                                                    # [in] batch size
    secureRandomBytes: array[32, byte]                                         # [in] 32 random bytes
): cttEthKzgStatus
  {.libPrefix: prefix_eth_kzg, raises: [].}
```

**Purpose:** Batch verification of N (commitment, cell, proof) triples. Implements the [universal DAS verification equation](https://ethresear.ch/t/a-universal-verification-equation-for-data-availability-sampling/13240).

**Algorithm:**

```
Edge cases:
  n < 0     → InputsLengthsMismatch
  n == 0    → Success (trivially true)

1. Index validation: cell_indices[i] < CELLS_PER_EXT_BLOB

2. Commitment deduplication (on raw bytes, fast):
   deduplicateCommitments → commitmentIdx[N], firstOccurrence[M], M unique

3. Deserialization of M unique commitments only:
   deserialize_g1_compressed(uniqueCommitments[j], commitments_bytes[firstOccurrence[j]])

4. Deserialization of N cells and N proofs:
   cellToCosetEvals(cosets_evals[i], cells[i])
   deserialize_g1_compressed(proofs[i], proofs_bytes[i])

5. Fiat-Shamir challenge r:
   if secureRandomBytes are non-zero → use them directly (for testing)
   else → compute_verify_cell_kzg_proof_batch_challenge(...)

6. Powers of r: [r^1, r^2, ..., r^N]
   computePowers(r, n, skipOne=true)

7. Batch verification:
   kzg_coset_verify_batch(
     uniqueCommitments, commitmentIdx,
     proofs, cosets_evals, evalsCols,
     domain = fft_desc_ext,
     linearIndepRandNumbers = rPowers,
     powers_of_tau = SRS,
     tau_pow_L_g2 = SRS_G2[64],
     N = FIELD_ELEMENTS_PER_EXT_BLOB
   ) → bool
```

**`secureRandomBytes` parameter:**
- All zeros `[0; 32]` → library computes `r` via Fiat-Shamir (production mode)
- Non-zero value → used directly as randomness source (for testing determinism)

**Return values:**
- `cttEthKzg_Success` — all proofs are valid
- `cttEthKzg_VerificationFailure` — at least one proof is invalid
- Other codes — input data errors

---

## 8. Algorithms and Optimizations

### Half-FFT Optimization in `compute_cells_impl`

**Mathematical background:**

A full extended blob has 8192 field elements. The naive approach requires an FFT of size 8192. However:

- After bit-reversal of indices 0..8191 (13 bits): even indices → first 4096 positions, odd indices → second 4096 positions
- Even roots: `w_8192^(2k) = w_4096^k` — these are the same evaluation points as the original blob
- Therefore: **cells 0–63 = the original blob data** (O(1), `copyMem` only)

For cells 64–127 with odd roots `w_8192^(2k+1) = w_8192 · w_4096^k`:
- Multiply coefficient `c_k` by `w_8192^k` (frequency domain shift)
- Apply FFT(4096) to the shifted coefficients

**Resulting complexity:**
- Full FFT(8192): ≈ 8192 × 13 = 106,496 multiplications
- Half-FFT:       ≈ 4096 × 12 × 2 = 98,304 multiplications (IFFT + FFT)
- Plus O(4096) for the shift
- **~2× speedup** over full FFT, **~370× speedup** over naive O(N²)

### Commitment Deduplication: Byte-Level Comparison

Working on 48-byte compressed G1 points rather than deserialized EC points:
- Size: 48 bytes vs ~96 bytes → more cache-efficient
- Cryptographic property: uniform byte distribution → P(first byte match) = 1/256 → `memcmp` exits after ~1–2 bytes on average for non-matching commitments
- Zero deserialization overhead for duplicate commitments

### Memory Management

All large buffers are allocated on the heap with 64-byte alignment:

```nim
# Typical allocation pattern (inside API functions):
let buffer = allocHeapAligned(SomeType, alignment = 64)
defer: freeHeapAligned(buffer)  # guaranteed release on any exit path
```

Rationale:
- 128 KB on the stack = 10% of Windows default stack limit (1 MB)
- 64-byte alignment = CPU cache line size → optimal SIMD/AVX performance

### Safety Pragmas

```nim
{.push raises: [].}   # No exceptions allowed (mandatory for cryptographic code)
{.push checks: off.}  # Disable int overflow and array bounds checking
```

---

## 9. Error Codes

Type `cttEthKzgStatus` (from `constantine/ethereum_eip4844_kzg`):

| Code | Meaning |
|------|---------|
| `cttEthKzg_Success` | Operation completed successfully |
| `cttEthKzg_VerificationFailure` | Verification failed (invalid proof) |
| `cttEthKzg_InputsLengthsMismatch` | Input length mismatch or nil pointers |
| `cttEthKzg_ScalarLargerThanCurveOrder` | Bytes encode a scalar ≥ r |
| `cttEthKzg_CellIndicesNotAscending` | Cell indices are not strictly ascending |

Scalar serialization status codes `CttCodecScalarStatus` (from `codecs_status_codes`):

| Code | Meaning |
|------|---------|
| `cttCodecScalar_Success` | Success |
| `cttCodecScalar_Zero` | Success (zero scalar) |
| Others | Deserialization error |

---

## 10. Full Data Flow Diagram

```
                      ┌────────────────────────────────────┐
                      │            Blob (128 KB)            │
                      │     array[4096, array[32, byte]]    │
                      └───────────────┬────────────────────┘
                                      │ blob_to_field_polynomial()
                                      ▼
                 ┌────────────────────────────────────────────┐
                 │  PolynomialEval[4096, Fr, kBitReversed]    │
                 │  (Lagrange form, bit-reversed)              │
                 └──────────┬─────────────────────────────────┘
                            │
            ┌───────────────┴──────────────────┐
            │ lagrangeInterpolate (IFFT)         │
            ▼                                    │
┌─────────────────────────────┐                  │
│  PolynomialCoef[4096, Fr]   │                  │ (direct copy)
│  (monomial / coefficient    │                  │
│   form)                     │                  ▼
└─────┬──────────────┬────────┘     cells 0–63 = blob
      │              │
 shift+FFT        FK20 multiproof
      │              │
      ▼              ▼
cells 64–127    128 × G1Aff
                      │
                 bit_reversal_permutation
                      │
                 serialize_g1_compressed
                      │
                      ▼
               128 × KZGProofBytes[48]
```

---

## 11. Usage Examples

### Full cycle: publishing a blob

```nim
import
  constantine/commitments_setups/ethereum_kzg_srs,
  constantine/eth_eip7594_peerdas

# Initialize context once (expensive operation)
var ctx: EthereumKZGContext
ctx.trusted_setup_load(trustedSetupFile, kReferenceCKzg)

# Blob data (128 KB)
var blob: Blob
# ... fill blob ...

# Compute all cells and proofs
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

### Batch verification of cells

```nim
# n = number of (commitment, cell_index, cell, proof) tuples to verify
let n = 256

var secureRandom: array[32, byte]
# ... fill secureRandom with random bytes ...

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
  echo "Verification failed: invalid sample detected"
elif status != cttEthKzg_Success:
  echo "Input error: ", status
```

### Recovery from partial data

```nim
# Suppose cells with indices 0..63 are available (exactly 50%)
var
  knownIndices: array[64, CellIndex]
  knownCells: array[64, Cell]

for i in 0 ..< 64:
  knownIndices[i] = CellIndex(i)
  # ... fill knownCells[i] ...

var
  recoveredCells: array[CELLS_PER_EXT_BLOB, Cell]
  recoveredProofs: array[CELLS_PER_EXT_BLOB, KZGProofBytes]

let status = recover_cells_and_kzg_proofs(
  addr ctx,
  cast[ptr UncheckedArray[Cell]](addr recoveredCells[0]),
  cast[ptr UncheckedArray[KZGProofBytes]](addr recoveredProofs[0]),
  cast[ptr UncheckedArray[CellIndex]](addr knownIndices[0]),
  cast[ptr UncheckedArray[Cell]](addr knownCells[0]),
  64  # number of known cells
)
doAssert status == cttEthKzg_Success
# recoveredCells[0..127] and recoveredProofs[0..127] now contain complete data
```

### Cells only (without proofs)

```nim
# compute_cells is faster when proofs are not needed
var cells: array[CELLS_PER_EXT_BLOB, Cell]

let status = compute_cells(addr ctx, cells, blob)
doAssert status == cttEthKzg_Success

# Access individual cells:
let cell42 = cells[42]   # array[2048, byte] — bytes of cell #42
```

---

## References

- [EIP-7594](https://eips.ethereum.org/EIPS/eip-7594) — PeerDAS specification
- [Ethereum Consensus Spec (polynomial-commitments-sampling)](https://github.com/ethereum/consensus-specs/blob/v1.4.0-beta.1/specs/fulu/polynomial-commitments-sampling.md)
- [FK20 Paper](https://eprint.iacr.org/2023/033) — multi-point KZG proof generation algorithm
- [Universal DAS Verification Equation](https://ethresear.ch/t/a-universal-verification-equation-for-data-availability-sampling/13240)
- [Constantine: Zero-overhead cryptography](https://github.com/mratsim/constantine)
