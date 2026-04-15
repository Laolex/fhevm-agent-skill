---
name: fhevm-migrate
description: Agentic command — migrate an fhEVM contract from old TFHE.* API / GatewayCaller pattern to current FHE.* API (v0.9+). Rewrites imports, function calls, inheritance, callback signatures, and hardhat config in one pass.
type: command
trigger: /fhevm migrate
---

# /fhevm migrate — TFHE → FHE API Migration

You are an fhEVM migration agent. When invoked, read the target contract(s), apply every transformation below in a single pass, and write the result back. Do not ask for confirmation between steps — read, transform, write, then show a diff summary.

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
| `import {SepoliaZamaFHEVMConfig} from "@fhevm/solidity/config/ZamaFHEVMConfig.sol";` | `import {ZamaEthereumConfig} from "./ZamaConfig.sol";` |
| `import {SepoliaZamaGatewayConfig} from "@fhevm/solidity/config/ZamaGatewayConfig.sol";` | (remove) |
| `import {GatewayCaller} from "@fhevm/solidity/gateway/GatewayCaller.sol";` | (remove) |
| `import {Gateway} from "@fhevm/solidity/gateway/lib/Gateway.sol";` | (remove) |
| `import {SepoliaConfig} from "@fhevm/solidity/config/ZamaConfig.sol";` | `import {ZamaEthereumConfig} from "./ZamaConfig.sol";` |

### 2b. Fix inheritance

| Old | New |
|---|---|
| `is SepoliaZamaFHEVMConfig, SepoliaZamaGatewayConfig, GatewayCaller` | `is ZamaEthereumConfig` |
| `is SepoliaZamaFHEVMConfig` | `is ZamaEthereumConfig` |
| `is SepoliaConfig` | `is ZamaEthereumConfig` ← if importing from @fhevm/solidity |

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
| `TFHE.gte(` | `FHE.gte(` |
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

### 2e. Fix Gateway decryption → FHE.requestDecryption

Old pattern:
```solidity
uint256[] memory cts = new uint256[](1);
cts[0] = Gateway.toUint256(encValue);
uint256 reqId = Gateway.requestDecryption(cts, this.callback.selector, 0, block.timestamp + 100, false);

function callback(uint256 requestId, uint64 result) public onlyGateway {
    revealedValue = result;
}
```

New pattern:
```solidity
bytes32[] memory cts = new bytes32[](1);
cts[0] = FHE.toBytes32(encValue);
FHE.requestDecryption(cts, this.callback.selector);

function callback(uint256 requestId, bytes memory cleartexts, bytes memory decryptionProof) external {
    FHE.checkSignatures(requestId, cleartexts, decryptionProof);
    (uint64 result) = abi.decode(cleartexts, (uint64));
    revealedValue = result;
}
```

Remove `onlyGateway` modifier. Update `cts` type from `uint256[]` to `bytes32[]`.

### 2f. Flag FHE.gte usage

If old code used `TFHE.gte(a, b)` → replace with `FHE.not(FHE.lt(b, a))` and add comment:
```solidity
// ⚠️ FHE.gte not available in v0.11 — using not(lt(b,a)) equivalent
FHE.not(FHE.lt(b, a))
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
| `import { createInstance } from "fhevmjs"` | `import { initSDK, createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk/web.js"` |
| `import { initFhevm } from "@zama-fhe/relayer-sdk/web.js"` | (remove — initFhevm removed) |
| `await initFhevm()` | `await initSDK()` |
| `createInstance({ network: window.ethereum, ...hardcodedAddresses })` | `createInstance({ ...SepoliaConfig, network: window.ethereum, relayerUrl: \`\${window.location.origin}/api/zama-relay\` })` |
| `fhevmInstance.encrypt32(value)` | `fhevmInstance.encryptUint({ value: BigInt(value), type: "euint32", contractAddress, callerAddress })` |
| `fhevmInstance.encrypt64(value)` | `fhevmInstance.encryptUint({ value: BigInt(value), type: "euint64", contractAddress, callerAddress })` |
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
    • Import: added FHE, ZamaEthereumConfig
    • Inheritance: SepoliaZamaFHEVMConfig → ZamaEthereumConfig
    • Gateway callback → FHE.requestDecryption pattern
    • 2 einput params → bytes32 inputHandle

  hardhat.config.ts
    • process.env → vars.get()

  frontend/src/App.tsx
    • initFhevm → initSDK
    • fhevmjs → @zama-fhe/relayer-sdk/web.js
    • Added relayerUrl proxy override

Manual steps still needed:
  □ Copy ZamaConfig.sol to contracts/ (from node_modules/@fhevm/solidity/config/ZamaConfig.sol)
  □ npx hardhat vars set MNEMONIC
  □ npx hardhat vars set INFURA_API_KEY
  □ Add /api/zama-relay.js CORS proxy (see 05-frontend.md)
  □ Add vercel.json rewrite rule for /api/zama-relay/:path*
  □ npx hardhat compile  (verify no remaining errors)
  □ npx hardhat test
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If there are patterns you couldn't automatically convert (e.g., complex Gateway logic), list them explicitly under "Manual steps still needed" with the file and line number.
