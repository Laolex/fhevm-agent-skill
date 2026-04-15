---
name: fhevm-architecture
description: FHEVM architecture overview — how FHE works on-chain, coprocessor model, network topology, key concepts every developer must understand
type: reference
---

# FHEVM — Architecture & How FHE Works On-Chain

## What is FHEVM?

FHEVM (Fully Homomorphic Encryption Virtual Machine) is a protocol that extends the EVM to support computation on encrypted state. Smart contracts can store, compute on, and selectively reveal encrypted values **without ever decrypting them on-chain**.

The core insight: operations run on ciphertext, not plaintext. The chain never sees the underlying values — only encrypted handles.

---

## Network topology

```
User Browser
│
│  encryptUint({ value, contractAddress, callerAddress })
│  → encrypted handle + ZK input proof
│
▼
Smart Contract (Sepolia)
│
│  FHE.fromExternal(handle, proof)  → euint64 (ciphertext handle stored on-chain)
│  FHE.add / FHE.mul / FHE.select  → new ciphertext handles (computed off-chain by coprocessor)
│  FHE.allowThis / FHE.allow       → ACL contract grants access to handle
│
▼
FHE Coprocessor (Zama Protocol)
│
│  Executes all FHE operations in TEE (Trusted Execution Environment)
│  Returns new encrypted handles to the chain
│  Holds the global FHE key — never exposed
│
▼
KMS (Key Management Service)
│
│  Handles re-encryption: user signs EIP-712 with ephemeral public key
│  KMS re-encrypts ciphertext under user's key → user decrypts locally
│  Also handles public decryption callbacks (oracle pattern)
```

---

## Key concepts

### Ciphertext handles
Every encrypted value on-chain is a `bytes32` handle — a pointer into the coprocessor's ciphertext store. The chain stores handles, not ciphertext. `euint64` is just a type wrapper around a `bytes32` handle.

### ACL (Access Control List)
The ACL is a singleton contract on Sepolia that tracks which addresses/contracts are permitted to use each ciphertext handle. **Without ACL permission, the coprocessor refuses to operate on a handle.**

- `FHE.allowThis(val)` — grants current contract permission (required after every computation)
- `FHE.allow(val, addr)` — grants a specific address permission (for re-encryption)
- `FHE.allowTransient(val, addr)` — one-time use permission (cleared after the tx)
- `FHE.allowPublic(val)` — anyone can compute on it (use carefully)

### Input proofs
When a user provides an encrypted value to a contract, they must also submit a ZK proof that:
1. The value is within the valid range for the type
2. The value was encrypted specifically for this contract+caller pair (prevents replay)

This is why `contractAddress` and `callerAddress` are required in `encryptUint()`.

### Coprocessor execution model
FHE operations are NOT executed synchronously in the EVM. The EVM emits an event, the coprocessor picks it up, executes the operation in the TEE, and the result is available in the next block (or same block depending on implementation). This is why:
- Gas costs for FHE ops are fixed estimates (not actual EVM computation)
- You cannot read the result of an FHE op in the same transaction

---

## Sepolia deployment addresses (v0.11+)

| Contract | Address |
|----------|---------|
| ACL | `0x339EcE85B9E11a3A3AA557582784a15d7F82AAf2` |
| KMS Verifier | `0x904Cda47dbFC1E355F9e58E3f3A89b45B57fF0cB` |
| Input Verifier | `0x69dE3158643e738a0724418b21a35FAA20CBb1c5` |
| FHEVM Executor | `0x596E6682c72946AF006B27C131793F2b62527A4B` |
| Decryption Oracle | `0x33347831500F1e73f0ccCBb95c9f86B94d7b1123` |
| cUSDT (ERC-7984) | `0x9Bc5A98A48742b51F74A9ED40bA9E96B56CD3e23` |

These are baked into `SepoliaConfig` — do not hardcode them manually.

---

## ZamaConfig.sol — local copy pattern

Contracts must inherit the correct config for their network. Always copy `ZamaConfig.sol` locally to avoid dependency issues:

```solidity
// contracts/ZamaConfig.sol — copy from node_modules, rename if needed
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import {ZamaFHEVMConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

abstract contract SepoliaZamaFHEVMConfig is ZamaFHEVMConfig {
    constructor() {
        // Sepolia addresses baked in by ZamaFHEVMConfig
    }
}
```

```solidity
// Your contract
contract MyContract is SepoliaZamaFHEVMConfig, AccessControl, ReentrancyGuard {
```

---

## Three decryption patterns

| Pattern | When to use | How |
|---------|-------------|-----|
| **Re-encryption** | User reads their own private data | EIP-712 sign → KMS re-encrypts under user key → user decrypts locally |
| **Oracle callback** | Public result after event (auction end, vote close) | `FHE.requestDecryption()` → coprocessor decrypts → callback function |
| **Public decrypt** | Admin/liquidation reveals a specific value | `FHE.makePubliclyDecryptable()` → `FHE.checkSignatures()` → plaintext on-chain |

---

## What FHEVM cannot do

- **No encrypted-to-encrypted division**: `FHE.div(a, b)` where `b` is encrypted — divisor must be plaintext
- **No branching on ebool**: `if (encryptedBool)` is a compile error — use `FHE.select`
- **No reading results in same tx**: computed values are available next block
- **No `view` functions that decrypt**: re-encryption requires a signed transaction to KMS
- **No uint256 encrypted**: `euint256` exists but is extremely gas-heavy — avoid
- **No overflow protection by default**: `FHE.add` wraps — validate input ranges in your contract logic

---

## Developer environment

```bash
# Install fhEVM Hardhat plugin
npm install --save-dev @fhevm/hardhat-plugin @fhevm/solidity

# hardhat.config.ts — minimum required
import "@fhevm/hardhat-plugin";
// fhevm plugin adds: hre.fhevm.createEncryptedInput(), hre.fhevm.initializeCLIApi()

# Compile
npx hardhat compile

# Test with mock coprocessor (no real FHE, deterministic — fast)
npx hardhat test

# Deploy to Sepolia (real FHE coprocessor)
npx hardhat run scripts/deploy.ts --network sepolia
```

The Hardhat plugin includes a **mock coprocessor** for local testing — all FHE operations are computed instantly in plaintext simulation. Tests are deterministic and fast. The mock is NOT a security check — it only validates logic flow.
