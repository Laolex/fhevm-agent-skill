---
name: fhevm-hcu-costs
description: FHE Homomorphic Complexity Unit (HCU) cost table — every operation for ebool, euint8/16/32/64/128/256, global and depth limits, budget planning patterns. Use when diagnosing HCU revert errors or estimating transaction cost.
type: reference
---

# HCU Cost Table

Fully Homomorphic Complexity Units (HCU) meter the on-chain cost of FHE operations.
Every `FHE.*` call consumes HCU; exceeding the per-transaction budget reverts the tx.

**Current devnet limits:**
- Global HCU: **20,000,000** per transaction
- Depth HCU: **5,000,000** per transaction

**Scalar vs non-scalar:**
- **Scalar**: one operand is encrypted (`euintX`), the other is a plaintext literal (e.g. `FHE.add(a, 100)`)
- **Non-scalar**: both operands are encrypted ciphertexts (e.g. `FHE.add(a, b)`)

---

## `ebool`

| Operation | HCU (scalar) | HCU (non-scalar) |
|---|---|---|
| `FHE.and` | 22,000 | 25,000 |
| `FHE.or` | 22,000 | 24,000 |
| `FHE.xor` | 2,000 | 22,000 |
| `FHE.not` | — | **2** |
| `FHE.select` | — | 55,000 |
| `FHE.randEbool` | — | 19,000 |

---

## `euint8`

| Operation | HCU (scalar) | HCU (non-scalar) |
|---|---|---|
| `FHE.add` | 84,000 | 88,000 |
| `FHE.sub` | 84,000 | 91,000 |
| `FHE.mul` | 122,000 | 150,000 |
| `FHE.div` | 210,000 | — (plaintext divisor only) |
| `FHE.rem` | 440,000 | — |
| `FHE.and` | 31,000 | 31,000 |
| `FHE.or` | 30,000 | 30,000 |
| `FHE.xor` | 31,000 | 31,000 |
| `FHE.shr` | 32,000 | 91,000 |
| `FHE.shl` | 32,000 | 92,000 |
| `FHE.rotr` | 31,000 | 93,000 |
| `FHE.rotl` | 31,000 | 91,000 |
| `FHE.eq` | 55,000 | 55,000 |
| `FHE.ne` | 55,000 | 55,000 |
| `FHE.ge` | 52,000 | 63,000 |
| `FHE.gt` | 52,000 | 59,000 |
| `FHE.le` | 58,000 | 58,000 |
| `FHE.lt` | 52,000 | 59,000 |
| `FHE.min` | 84,000 | 119,000 |
| `FHE.max` | 89,000 | 121,000 |
| `FHE.neg` | — | 79,000 |
| `FHE.not` | — | **9** |
| `FHE.select` | — | 55,000 |
| `FHE.randEuint8` | — | 23,000 |

---

## `euint16`

| Operation | HCU (scalar) | HCU (non-scalar) |
|---|---|---|
| `FHE.add` | 93,000 | 93,000 |
| `FHE.sub` | 93,000 | 93,000 |
| `FHE.mul` | 193,000 | 222,000 |
| `FHE.div` | 302,000 | — |
| `FHE.rem` | 580,000 | — |
| `FHE.and/or/xor` | 31,000 | 31,000 |
| `FHE.shr/shl` | 32,000 | ~123,000 |
| `FHE.eq/ne` | 55,000 | 83,000 |
| `FHE.ge/gt/le/lt` | 55–58,000 | 83–84,000 |
| `FHE.min` | 88,000 | 146,000 |
| `FHE.max` | 89,000 | 145,000 |
| `FHE.neg` | — | 93,000 |
| `FHE.not` | — | **16** |
| `FHE.select` | — | 55,000 |

---

## `euint32`

| Operation | HCU (scalar) | HCU (non-scalar) |
|---|---|---|
| `FHE.add` | 95,000 | 125,000 |
| `FHE.sub` | 95,000 | 125,000 |
| `FHE.mul` | 265,000 | 328,000 |
| `FHE.div` | 438,000 | — |
| `FHE.rem` | 792,000 | — |
| `FHE.and/or/xor` | 32,000 | 32,000 |
| `FHE.shr/shl` | 32,000 | ~162,000 |
| `FHE.eq/ne` | 82–83,000 | 85–86,000 |
| `FHE.ge/gt/le/lt` | 84,000 | 117–118,000 |
| `FHE.min` | 117,000 | 182,000 |
| `FHE.max` | 117,000 | 180,000 |
| `FHE.neg` | — | 131,000 |
| `FHE.not` | — | **32** |
| `FHE.select` | — | 55,000 |

---

## `euint64` — standard for token amounts

| Operation | HCU (scalar) | HCU (non-scalar) |
|---|---|---|
| `FHE.add` | 133,000 | 162,000 |
| `FHE.sub` | 133,000 | 162,000 |
| `FHE.mul` | 365,000 | 596,000 |
| `FHE.div` | ~700,000 | — |
| `FHE.rem` | ~1,200,000 | — |
| `FHE.and/or/xor` | 33,000 | 33,000 |
| `FHE.shr/shl` | 33,000 | ~220,000 |
| `FHE.eq/ne` | ~110,000 | ~115,000 |
| `FHE.ge/gt/le/lt` | ~115,000 | ~165,000 |
| `FHE.min` | ~165,000 | ~250,000 |
| `FHE.max` | ~165,000 | ~250,000 |
| `FHE.neg` | — | ~190,000 |
| `FHE.not` | — | **64** |
| `FHE.select` | — | 55,000 |
| `FHE.randEuint64` | — | 25,000 |

---

## `euint128`

Arithmetic costs scale further from `euint64`. Use only when values may exceed
`uint64.max` (18.4 × 10¹⁸). For most token amounts with 18 decimals,
`euint64` is sufficient.

---

## `euint256`

Supported operations: `and`, `or`, `xor`, `shl`, `shr`, `rotl`, `rotr`,
`eq`, `ne`, `neg`, `not`, `select`, `rand`, `randBounded`. **No arithmetic**
(`add`, `sub`, `mul`, `div`). Use only for flag bitmasks or exceptional cases.

---

## Budget Planning

| Common pattern | Approximate HCU per tx |
|---|---|
| Counter increment (`euint64 add`) | ~162K global |
| Token transfer (`sub + add + eq + select`) | ~500–600K global |
| Auction bid update (`lt + select`) | ~220K global |
| Overflow-safe mint (`add + lt + select × 2`) | ~700K global |
| Complex scoring (`mul + add + compare`) | 1–2M global |

**Rule of thumb at 20M global limit:**
- ~120 `euint64` additions, OR
- ~33 `euint64` multiplications, OR
- ~10 complex scoring operations per transaction

## HCU Optimization Tips

- **Prefer scalar ops** where one operand is a known plaintext constant — scalar costs are lower.
- **`FHE.shr(x, 1)`** is cheaper than `FHE.div(x, 2)` — use shifts for powers of 2.
- **`euint8`/`euint16`** are cheaper than `euint64` for small-range values (e.g. credit tiers, vote counts).
- **`FHE.not` is nearly free** (cost equals the bit width: 2 for ebool, 64 for euint64).
- Avoid `FHE.mul` of two ciphertexts (596K for euint64) in hot paths — restructure to scalar if possible.
- Avoid `FHE.rem` (1.2M for euint64) — use bit-masking for power-of-2 moduli: `FHE.and(x, FHE.asEuint64(7))` for mod 8.
- **Batch operations** don't reduce per-op costs, but keep transaction count low (fewer Sepolia fees).

## Diagnosing HCU Reverts

```
Error: execution reverted: HCU limit exceeded
```

Steps:
1. Count FHE ops in the failing function — multiply by type-specific cost above.
2. If total > 5M: likely depth limit (deep nested selects/compares). Break into separate transactions.
3. If total > 20M: global limit. Reduce number of FHE ops per transaction or downsize types.
4. Use `euint8` instead of `euint64` where the value range allows.
