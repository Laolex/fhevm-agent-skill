---
name: fhevm-migrate
description: Agentic command — migrate an fhEVM contract from old TFHE.* API / GatewayCaller pattern to current FHE.* API (@fhevm/solidity v0.11.x). Rewrites imports, function calls, inheritance, decryption pattern (Pattern 3 only), and frontend SDK usage in one pass.
type: command
trigger: /fhevm migrate
---

# /fhevm migrate — TFHE → FHE API Migration

You are an fhEVM migration agent. When invoked, read the target contract(s), apply every transformation below in a single pass, and write the result back. Do not ask for confirmation between steps — read, transform, write, then show a diff summary.

**Target stack:** `@fhevm/solidity@^0.11.1`, `@fhevm/hardhat-plugin@^0.4.x`, `@zama-fhe/relayer-sdk@^0.4.1`.

---

## Step 1 — Identify files to migrate

If the user gave file paths: use those.
Otherwise glob: `contracts/**/*.sol`, `hardhat.config.ts`, `test/**/*.ts`, `frontend/src/**/*.ts`, `frontend/src/**/*.tsx`.

Read all files before transforming.

---

## Step 2 — Apply Solidity transformations

### 2a. Fix imports

| Old (remove) | New (add) |
|---|---|
| `import "fhevm/lib/TFHE.sol";` | `import {FHE, euint64, ebool, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";` |
| `import {TFHE} from "fhevm/lib/TFHE.sol";` | same as above |
| `import {SepoliaZamaFHEVMConfig} from "@fhevm/solidity/config/ZamaFHEVMConfig.sol";` | `import {SepoliaConfig} from "@fhevm/solidity/config/ZamaConfig.sol";` |
| `import {SepoliaZamaGatewayConfig} from "@fhevm/solidity/config/ZamaGatewayConfig.sol";` | (remove) |
| `import {GatewayCaller} from "@fhevm/solidity/gateway/GatewayCaller.sol";` | (remove) |
| `import {Gateway} from "@fhevm/solidity/gateway/lib/Gateway.sol";` | (remove) |
| Any local `import {...} from "./ZamaConfig.sol";` | `import {SepoliaConfig} from "@fhevm/solidity/config/ZamaConfig.sol";` (npm canonical — do NOT vendor a local copy) |

### 2b. Fix inheritance

| Old | New |
|---|---|
| `is SepoliaZamaFHEVMConfig, SepoliaZamaGatewayConfig, GatewayCaller` | `is SepoliaConfig` |
| `is SepoliaZamaFHEVMConfig` | `is SepoliaConfig` |

For Ethereum mainnet contracts substitute `ZamaEthereumConfig` for `SepoliaConfig`.

Keep all other inheritance (AccessControl, ReentrancyGuard, Pausable) unchanged.

### 2c. Replace all TFHE.* calls with FHE.*

Apply globally (replace_all):

| Old | New |
|---|---|
| `TFHE.asEuint8(` | `FHE.asEuint8(` |
| `TFHE.asEuint16(` | `FHE.asEuint16(` |
| `TFHE.asEuint32(` | `FHE.asEuint32(` |
| `TFHE.asEuint64(` | `FHE.asEuint64(` |
| `TFHE.asEbool(` | `FHE.asEbool(` |
| `TFHE.add(` | `FHE.add(` |
| `TFHE.sub(` | `FHE.sub(` |
| `TFHE.mul(` | `FHE.mul(` |
| `TFHE.div(` | `FHE.div(` |
| `TFHE.eq(` | `FHE.eq(` |
| `TFHE.ne(` | `FHE.ne(` |
| `TFHE.lt(` | `FHE.lt(` |
| `TFHE.lte(` | `FHE.lte(` |
| `TFHE.gt(` | `FHE.gt(` |
| `TFHE.gte(a, b)` | `FHE.not(FHE.lt(a, b))` ⚠️ no `FHE.gte` in v0.11 — see 2f |
| `TFHE.and(` | `FHE.and(` |
| `TFHE.or(` | `FHE.or(` |
| `TFHE.not(` | `FHE.not(` |
| `TFHE.select(` | `FHE.select(` |
| `TFHE.shr(` | `FHE.shr(` |
| `TFHE.shl(` | `FHE.shl(` |
| `TFHE.min(` | `FHE.min(` |
| `TFHE.max(` | `FHE.max(` |
| `TFHE.rand(` | `FHE.rand(` |
| `TFHE.allow(` | `FHE.allow(` |
| `TFHE.allowThis(` | `FHE.allowThis(` |
| `TFHE.allowPublic(` | `FHE.allowPublic(` |
| `TFHE.isSenderAllowed(` | (remove call — ACL is automatic in v0.9+) |

### 2d. Fix input proof pattern

Old pattern (replace):
```solidity
// Old — einput type
function deposit(einput encAmount, bytes calldata inputProof) external {
    euint64 amount = TFHE.asEuint64(encAmount, inputProof);
```

New pattern:
```solidity
function deposit(bytes32 inputHandle, bytes calldata inputProof) external {
    euint64 amount = FHE.fromExternal(externalEuint64.wrap(inputHandle), inputProof);
    FHE.allowThis(amount);
    FHE.allow(amount, msg.sender);
```

Also replace `einput` type with `bytes32` wherever it appears as a parameter type.

### 2e. Decryption: convert Gateway oracle callbacks → Pattern 3 (makePubliclyDecryptable)

> ⚠️ **`FHE.requestDecryption` does NOT exist in `@fhevm/solidity@0.11.x`.** The only on-chain public-decrypt path is Pattern 3: `FHE.makePubliclyDecryptable(handle)` on-chain → off-chain relayer `publicDecrypt` → on-chain `verifyReveal(handles, cleartexts, proof)` with `FHE.checkSignatures(handlesList, cleartexts, proof)` plus **handle pinning**.

Old Gateway pattern:
```solidity
uint256[] memory cts = new uint256[](1);
cts[0] = Gateway.toUint256(encValue);
uint256 reqId = Gateway.requestDecryption(cts, this.callback.selector, 0, block.timestamp + 100, false);

function callback(uint256 requestId, uint64 result) public onlyGateway {
    revealedValue = result;
}
```

New Pattern 3 rewrite:
```solidity
// Step 1 — flag the ciphertext as publicly decryptable
function requestReveal() external {
    FHE.makePubliclyDecryptable(encValue);
    pendingReveal = true;
}

// Step 2 — caller submits handlesList + cleartexts + KMS proof
function verifyReveal(
    bytes32[] calldata handlesList,
    bytes calldata cleartexts,
    bytes calldata decryptionProof
) external {
    require(pendingReveal, "no pending reveal");

    // 🔒 CRITICAL: pin every handle BEFORE checkSignatures.
    // checkSignatures only proves "KMS signed these handles decrypt to these cleartexts" —
    // it does NOT bind a handle to any contract slot. Without this require, an attacker
    // can substitute any KMS-signed handle (e.g. a publicly-decryptable euint64(0))
    // to forge the result.
    require(handlesList.length == 1, "bad handles length");
    require(handlesList[0] == FHE.toBytes32(encValue), "handle mismatch");

    FHE.checkSignatures(handlesList, cleartexts, decryptionProof);
    (uint64 result) = abi.decode(cleartexts, (uint64));
    revealedValue = result;
    pendingReveal = false;
}
```

Remove `onlyGateway` modifier. Replace `cts` array (old `uint256[]`) and `Gateway.toUint256` plumbing with `FHE.makePubliclyDecryptable`.

### 2f. Replace FHE.gte usage

`FHE.gte` is **not** present in `@fhevm/solidity@0.11.x`. Rewrite as:

```solidity
// gte(a, b)  →  not(lt(a, b))
ebool isGte = FHE.not(FHE.lt(a, b));

// Floor-at-zero subtraction (canonical form)
euint64 result = FHE.select(FHE.not(FHE.lt(a, b)), FHE.sub(a, b), FHE.asEuint64(0));
```

---

## Step 3 — Fix hardhat.config.ts

| Old | New |
|---|---|
| `import "@fhevm/hardhat-plugin"` | keep (correct) |
| `import "fhevm-hardhat-plugin"` | `import "@fhevm/hardhat-plugin"` |
| `accounts: [process.env.PRIVATE_KEY!]` | `accounts: { mnemonic: vars.get("MNEMONIC", "") }` |
| `url: process.env.RPC_URL` | `url: \`https://sepolia.infura.io/v3/\${vars.get("INFURA_API_KEY", "")}\`` |

Add at top if missing:
```typescript
const { vars } = require("hardhat/config");
```

---

## Step 4 — Fix frontend files

| Old | New |
|---|---|
| `import { createInstance } from "fhevmjs"` | `import { initSDK, createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk/web";` |
| `import { initFhevm } from "@zama-fhe/relayer-sdk/web";` | (remove — `initFhevm` removed in v0.4.x) |
| `import { initSDK, createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk/web.js";` | drop the trailing `.js` — use the `/web` subpath: `from "@zama-fhe/relayer-sdk/web"` |
| `import { ... } from "@zama-fhe/relayer-sdk/bundle";` | use `/web` for Vite/Next.js. The `/bundle` UMD entry is for plain `<script>` tags only and breaks ESM module loaders |
| `await initFhevm()` | `await initSDK()` |
| `createInstance({ network: window.ethereum, ...hardcodedAddresses })` | `createInstance({ ...SepoliaConfig, network: window.ethereum, relayerUrl: \`\${window.location.origin}/api/zama-relay\` })` |
| `fhevmInstance.encryptUint({ value, type, contractAddress, callerAddress })` | `await fhevmInstance.createEncryptedInput(contractAddress, callerAddress).add64(BigInt(value)).encrypt()` (use `.add8/.add16/.add32/.add64/.addBool` per type — `encryptUint` was removed in `@zama-fhe/relayer-sdk@0.4.1`) |
| `fhevmInstance.encrypt32(value)` | same `createEncryptedInput().add32(BigInt(value)).encrypt()` builder |
| `fhevmInstance.encrypt64(value)` | same `createEncryptedInput().add64(BigInt(value)).encrypt()` builder |
| `const { handle, proof } = ...` | `const handle = ethers.hexlify(encrypted.handles[0]); const proof = ethers.hexlify(encrypted.inputProof);` |

---

## Step 5 — Output summary

After all files are written, print a migration report:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Migration complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Files modified:
  contracts/MyContract.sol
    • 12 TFHE.* → FHE.* replacements
    • Import: removed GatewayCaller, SepoliaZamaFHEVMConfig
    • Import: added FHE, SepoliaConfig (npm)
    • Inheritance: SepoliaZamaFHEVMConfig → SepoliaConfig
    • Gateway callback → Pattern 3 (makePubliclyDecryptable + verifyReveal with handle pinning)
    • 2 einput params → bytes32 inputHandle
    • FHE.gte → FHE.not(FHE.lt(...))

  hardhat.config.ts
    • process.env → vars.get()

  frontend/src/App.tsx
    • initFhevm → initSDK
    • fhevmjs → @zama-fhe/relayer-sdk/web
    • encryptUint → createEncryptedInput().addNN().encrypt()
    • Added relayerUrl proxy override

Manual steps still needed:
  □ npx hardhat vars set MNEMONIC
  □ npx hardhat vars set INFURA_API_KEY
  □ Add /api/zama-relay.js CORS proxy (see 05-frontend.md)
  □ Add vercel.json rewrite rule for /api/zama-relay/:path*
  □ npx hardhat compile  (verify no remaining errors)
  □ npx hardhat test
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If there are patterns you couldn't automatically convert (e.g., complex Gateway logic), list them explicitly under "Manual steps still needed" with the file and line number.
