---
name: fhevm-types-ops
description: fhEVM encrypted types (euint8/64/ebool/eaddress), all FHE arithmetic/comparison/bitshift/select operations
type: reference
---

# fhEVM — Encrypted Types & FHE Operations

## Type sizes
| Type | Max value | Use case |
|------|-----------|----------|
| `euint8` | 255 | Votes, ratings, small enums |
| `euint16` | 65,535 | Percentages in bps, small amounts |
| `euint32` | ~4.3B | Timestamps, medium amounts |
| `euint64` | ~18.4 quintillion | Token amounts (6 decimals = up to ~18T USDC) |
| `euint128` | Very large | High precision amounts |
| `euint256` | Extremely large | Rare — use only when needed |
| `ebool` | true/false | Conditions, flags, health factors |
| `eaddress` | address | Private recipient addresses |

## Storage declarations
```solidity
// ✅ Correct — always private/internal
mapping(address => euint64) private encryptedBalances;
mapping(address => ebool) private isEligible;
euint64 private totalSupply;

// ❌ Never public — exposes the handle to everyone
mapping(address => euint64) public balances;
```

## Casting plaintext → encrypted
```solidity
euint64 amount = FHE.asEuint64(1000);
euint8  score  = FHE.asEuint8(50);
ebool   flag   = FHE.asEbool(true);

// ❌ Wrong — TFHE is outdated
euint64 amount = TFHE.asEuint64(1000);
```

## Arithmetic
```solidity
euint64 sum        = FHE.add(a, b);
euint64 difference = FHE.sub(a, b);      // wraps on underflow — see floor-at-zero below
euint64 product    = FHE.mul(a, b);
euint64 quotient   = FHE.div(a, 2);      // ✅ second arg MUST be plaintext uint64
euint64 remainder  = FHE.rem(a, 3);      // ✅ same — plaintext only

// ❌ Encrypted divisor — compile error
euint64 bad = FHE.div(a, b);
```

## Comparisons — return ebool
```solidity
ebool isEqual   = FHE.eq(a, b);
ebool notEqual  = FHE.ne(a, b);
ebool isGreater = FHE.gt(a, b);
ebool isGte     = FHE.gte(a, b);
ebool isLess    = FHE.lt(a, b);
ebool isLte     = FHE.lte(a, b);
```

## Boolean operations
```solidity
ebool result = FHE.and(condA, condB);
ebool result = FHE.or(condA, condB);
ebool result = FHE.not(cond);
ebool result = FHE.xor(condA, condB);
```

## FHE.select — encrypted ternary (critical pattern)
```solidity
// FHE.select(condition, ifTrue, ifFalse) — both branches computed, neither leaks
euint64 zero  = FHE.asEuint64(0);
ebool   isYes = FHE.eq(voteType, FHE.asEuint8(1));

yesVotes = FHE.add(yesVotes, FHE.select(isYes, weight, zero));
noVotes  = FHE.add(noVotes,  FHE.select(isYes, zero, weight));

// ❌ Never branch on encrypted booleans
if (isYes) { ... }  // compile error — ebool is not bool
```

## Bitshift — prefer over div for power-of-2
```solidity
euint64 halved    = FHE.shr(a, 1);   // a >> 1 = ÷2  (~80k gas vs ~150k for div)
euint64 quartered = FHE.shr(a, 2);   // a >> 2 = ÷4
euint64 doubled   = FHE.shl(a, 1);   // a << 1 = ×2
```

## Min / Max
```solidity
euint64 smaller = FHE.min(a, b);
euint64 larger  = FHE.max(a, b);
```

## Floor-at-zero subtraction (prevent underflow)
```solidity
// FHE.sub wraps on underflow — use select to clamp
euint64 result = FHE.select(FHE.gte(a, b), FHE.sub(a, b), FHE.asEuint64(0));
// Alternative: FHE.max(FHE.sub(a, b), FHE.asEuint64(0))
```

## Random encrypted value
```solidity
euint64 rand = FHE.rand(type(uint64).max);   // unpredictable to all parties
euint8  die  = FHE.rand(type(uint8).max);

// ⚠️ Cannot be read without decryption — use FHE.select to branch without revealing
```

## Type conversion
```solidity
euint8  small = FHE.asEuint8(100);
euint64 big   = FHE.asEuint64(small);  // upcast — always safe
euint8  back  = FHE.asEuint8(big);     // downcast — truncates high bits
```

## Score-gated tiered selection (lending pattern)
```solidity
// Two-tier FHE.select — set interest rate / collateral ratio based on encrypted credit score
// Avoids revealing which tier the user is in
euint64 ratio = FHE.asEuint64(150); // default

if (address(scoreContract) != address(0) && scoreContract.hasScore(borrower)) {
    ebool isPremium  = scoreContract.meetsThreshold(borrower, 800); // euint64 >= 800
    ebool isStandard = scoreContract.meetsThreshold(borrower, 600); // euint64 >= 600

    // premium → 110%, standard → 130%, else → 150%
    ratio = FHE.select(isPremium,
                FHE.asEuint64(110),
                FHE.select(isStandard,
                    FHE.asEuint64(130),
                    FHE.asEuint64(150)));
}

// Apply to health factor: isLiquidatable = collateral*100 < debt*ratio
euint64 collateralScaled = FHE.mul(pos.collateral, FHE.asEuint64(100));
euint64 debtScaled       = FHE.mul(pos.totalDebt,  ratio);
ebool   isLiq            = FHE.lt(collateralScaled, debtScaled);
```

## FHE.not(FHE.lt(a,b)) as gte (floor-at-zero repayment)
```solidity
// ✅ gte via not+lt — no FHE.gte in v0.11
euint64 newDebt = FHE.select(
    FHE.not(FHE.lt(pos.totalDebt, encRepay)),  // totalDebt >= encRepay
    FHE.sub(pos.totalDebt, encRepay),
    FHE.asEuint64(0)
);
// ✅ Also works: lte via not+lt: lte(a,b) = not(lt(b,a))
euint64 safeDiscount = FHE.select(
    FHE.not(FHE.lt(maxDiscount, discount)),    // maxDiscount >= discount
    discount,
    maxDiscount
);
```

## Interest accrual (mul then div plaintext)
```solidity
// interest = totalDebt * interestRate / 10000 (interestRate is in bps — e.g. 500 = 5%)
euint64 interest = FHE.div(FHE.mul(pos.totalDebt, pos.interestRate), 10000);
euint64 newDebt  = FHE.add(pos.totalDebt, interest);
// ⚠️ FHE.mul(euint64, euint64) can overflow if both are large — keep operands bounded
// pos.interestRate is in bps (≤ 1000), pos.totalDebt ≤ ~18.4e18 → safe for lending amounts
```

## ⚠️ uint64 overflow warning with ETH-wei
```
ETH amounts in wei can exceed uint64 max (18.4e18).
uint64 max ≈ 18.4 × 10^18 wei ≈ 18.4 ETH
→ Safe for lending (collateral in wei up to ~18 ETH per position)
→ For larger portfolios, use euint128 for collateral
```

## Gas cost reference
| Operation | ~Gas | Notes |
|-----------|------|-------|
| `FHE.asEuint64(n)` | 50k | Constant → encrypted |
| `FHE.fromExternal(handle, proof)` | 80k | Input proof validation |
| `FHE.add / FHE.sub` | 100k | |
| `FHE.mul` | 200k | |
| `FHE.div(a, plaintext)` | 150k | |
| `FHE.shr / FHE.shl` | 80k | Prefer over div |
| `FHE.eq / FHE.lt / FHE.gt` | 120k | → ebool |
| `FHE.select` | 150k | |
| `FHE.allowThis` | 30k | |
| `FHE.allow(val, addr)` | 30k | |
| `FHE.makePubliclyDecryptable` | 40k | |
| `FHE.requestDecryption` | 80k | |
| `FHE.checkSignatures` | 120k | |

**Design guidelines:** batch ACL grants at function end; prefer `FHE.shr` over `FHE.div`; avoid chaining >3-4 FHE ops per function (gas hits 1-2M).
