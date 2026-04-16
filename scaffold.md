---
name: fhevm-scaffold
description: Agentic command — scaffold a complete fhEVM confidential contract project from a plain-language spec. Generates contract, tests, deploy script, and frontend snippet.
type: command
trigger: /fhevm scaffold
---

# /fhevm scaffold — Confidential Contract Generator

You are an fhEVM code generator. When this command is invoked, execute the full workflow below without stopping to ask clarifying questions unless a required field is completely missing.

---

## Step 1 — Extract spec from user input

Parse the user's description and extract:

| Field | Source | Default if missing |
|-------|--------|-------------------|
| **Contract name** | User message | `ConfidentialProtocol` |
| **What to encrypt** | User message | collateral + debt amounts |
| **Roles needed** | User message | ADMIN_ROLE only |
| **Decryption pattern** | User message | Pattern 3 (makePubliclyDecryptable) |
| **Token type** | User message | ETH only |
| **Multi-asset?** | User message | false |
| **Credit score gate?** | User message | false |

If the user says "lending" → use ConfidentialLending architecture (collateral/debt/health factor)
If the user says "voting" → use encrypted tallies + FHE.select + Pattern 2 oracle reveal
If the user says "payroll" → use EMPLOYER_ROLE + AUDITOR_ROLE + salary euint64
If the user says "vault" → use single euint64 balance + deposit/withdraw + re-encrypt

### Zama Season 2 Builder Track context (apply when building for hackathon)

**What Season 1 winners had in common:**
- Creative use cases (not just "vault"): ZamaDAO (governance), ZamaBeliefSystem (prediction), Private-v4-Hooks (Uniswap AMM), privacy-pool-monorepo, Paychain (payroll)
- Real FHE usage: encrypted state that would be meaningless without FHE (not just hiding a number)
- Complete polish: working deployed frontend + working re-encrypt + working public decrypt
- DeFi primitives score higher than toy demos

**Avoid (already done / low scores):**
- Confidential payroll → Paychain won Season 1 with this
- Basic vault → too minimal
- ShieldLend (lending) → already our Season 1 submission

**High-scoring use cases still available:**
- Confidential AMM order book (hidden prices/quantities until fill)
- Sealed-bid auction (FHE.select for winner determination)
- Confidential DAO voting (hidden votes, tallied on-chain)
- Blind loan matching (lender/borrower matched without revealing terms)
- Private NFT trait reveal (encrypted metadata, reveal on demand)

If user hasn't specified a use case and is building for Season 2, suggest one of the high-scoring options above and confirm before scaffolding.

---

## Step 2 — Generate Solidity contract

Write the full contract to `contracts/<ContractName>.sol` following these rules:

**Imports (always):**
```solidity
pragma solidity ^0.8.24;
import {FHE, euint64, ebool, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "./ZamaConfig.sol";   // local copy
import {AccessControlEnumerable} from "@openzeppelin/contracts/access/extensions/AccessControlEnumerable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";
```

**If ERC20 tokens involved, add:**
```solidity
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
```

**Inheritance order:**
```solidity
contract <Name> is ZamaEthereumConfig, AccessControlEnumerable, ReentrancyGuard, Pausable {
    using SafeERC20 for IERC20; // if tokens involved
```

**Encrypted state — always private:**
```solidity
struct Position {
    euint64 <field1>;   // what it represents in ETH-wei or token units
    euint64 <field2>;
    ebool   <flag>;
    bool    active;
}
mapping(address => Position) private positions;
```

**Input functions — always include inputHandle + inputProof:**
```solidity
function deposit(bytes32 inputHandle, bytes calldata inputProof) external payable nonReentrant whenNotPaused {
    euint64 enc = FHE.fromExternal(externalEuint64.wrap(inputHandle), inputProof);
    FHE.allowThis(enc);
    FHE.allow(enc, msg.sender);
    // ... logic
}
```

**ACL rules — NEVER skip:**
- `FHE.allowThis(x)` immediately after every `FHE.fromExternal`
- `FHE.allowThis(x)` after every computed value (FHE.add, FHE.sub, FHE.select, etc.)
- `FHE.allow(x, user)` for every user who needs to re-encrypt
- If LIQUIDATOR_ROLE exists: `_allowLiquidators(flag)` using AccessControlEnumerable pattern

**Floor-at-zero subtraction (always use this form):**
```solidity
euint64 result = FHE.select(FHE.not(FHE.lt(a, b)), FHE.sub(a, b), FHE.asEuint64(0));
```

**Events — only plaintext metadata:**
```solidity
event Deposited(address indexed user, uint256 timestamp);   // ✅
// ❌ Never emit euint64 handles in events
```

**Decryption — use Pattern 3 for liquidation/close (two-step):**
```solidity
// Step 1
function requestReveal(address subject) external {
    FHE.makePubliclyDecryptable(positions[subject].flag);
    pendingReveal[subject] = true;
    emit RevealRequested(subject, FHE.toBytes32(positions[subject].flag), block.timestamp);
}
// Step 2
function verifyReveal(address subject, bytes32[] calldata handles, bytes calldata cleartexts, bytes calldata proof) external {
    require(pendingReveal[subject]);
    FHE.checkSignatures(handles, cleartexts, proof);
    bool result = abi.decode(cleartexts, (bool));
    delete pendingReveal[subject];
}
```

---

## Step 3 — Generate test file

Write `test/<ContractName>.test.ts`:

```typescript
import { ethers, fhevm } from "hardhat";
import { expect } from "chai";

describe("<ContractName>", function () {
    let contract: any, owner: any, user: any;

    before(async function () {
        await fhevm.initializeCLIApi(); // ✅ REQUIRED — must be first
    });

    beforeEach(async function () {
        [owner, user] = await ethers.getSigners();
        const Factory = await ethers.getContractFactory("<ContractName>");
        contract = await Factory.deploy();
        await contract.waitForDeployment();
    });

    // Generate tests for every public function:
    // - Happy path (valid inputs)
    // - ACL: verify re-encrypt returns correct value
    // - Edge cases: zero amounts, duplicate deposits
    // - Role restrictions: non-admin cannot call admin functions

    it("should encrypt and store value", async function () {
        const enc = await fhevm
            .createEncryptedInput(await contract.getAddress(), user.address)
            .add64(BigInt(1_000_000))
            .encrypt();
        await contract.connect(user).deposit(enc.handles[0], enc.inputProof, { value: ethers.parseEther("0.1") });
        expect(await contract.isActive(user.address)).to.be.true;
    });

    it("should decrypt user value correctly", async function () {
        // ... deposit first ...
        const handle = await contract.getEncrypted<Field>(user.address);
        const { publicKey, privateKey } = fhevm.generateKeypair();
        const contractAddr = await contract.getAddress();
        const now = Math.floor(Date.now() / 1000);
        const eip712 = fhevm.createEIP712(publicKey, [contractAddr], now, 1);
        const typeName = eip712.types.Reencrypt
            ? "Reencrypt"
            : Object.keys(eip712.types).find((k: string) => k !== "EIP712Domain")!;
        const sig = await user.signTypedData(eip712.domain, { [typeName]: eip712.types[typeName] }, eip712.message);
        const results = await fhevm.userDecrypt(
            [{ handle, contractAddress: contractAddr }],
            privateKey, publicKey, sig, [contractAddr], user.address, now, 1,
        );
        const decrypted = Object.values(results)[0] as bigint;
        expect(decrypted).to.equal(BigInt(1_000_000));
    });
});
```

Cover at minimum: deposit, borrow/action, userDecrypt, reveal flow, role checks.

---

## Step 4 — Generate deploy script

Write `scripts/deploy.ts`:

```typescript
import { ethers } from "hardhat";

async function main() {
    const [deployer] = await ethers.getSigners();
    console.log("Deployer:", deployer.address);

    const Factory = await ethers.getContractFactory("<ContractName>");
    const contract = await Factory.deploy();
    await contract.waitForDeployment();
    const addr = await contract.getAddress();
    console.log("<ContractName>:", addr);
    console.log("Etherscan:", `https://sepolia.etherscan.io/address/${addr}`);

    // Wire any companion contracts here
    // Grant roles if needed: await contract.grantRole(...)

    console.log("\nUpdate frontend/src/config.ts:");
    console.log(`  CONTRACT_ADDRESS = "${addr}"`);
}

main().catch((e) => { console.error(e); process.exit(1); });
```

---

## Step 5 — Generate full Vercel-deployable frontend

Write ALL of the following files (not just a snippet — a complete working app):

### `frontend/package.json`
```json
{
  "name": "<contract-name-kebab>-frontend",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "ethers": "^6.13.5",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@types/react": "^18.3.12",
    "@types/react-dom": "^18.3.1",
    "@vitejs/plugin-react": "^4.3.3",
    "typescript": "^5.6.3",
    "vite": "^5.4.10"
  }
}
```
Note: do NOT put @zama-fhe/relayer-sdk in package.json — it ships as a vendor bundle.

### `frontend/vite.config.ts`
```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    headers: {
      "Cross-Origin-Opener-Policy": "same-origin",
      "Cross-Origin-Embedder-Policy": "require-corp",
    },
  },
  optimizeDeps: { exclude: ["@zama-fhe/relayer-sdk"] },
});
```

### `frontend/vercel.json`
```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "Cross-Origin-Opener-Policy", "value": "same-origin" },
        { "key": "Cross-Origin-Embedder-Policy", "value": "require-corp" }
      ]
    }
  ],
  "rewrites": [
    { "source": "/api/zama-relay/:path*", "destination": "/api/zama-relay" }
  ]
}
```

### `frontend/api/zama-relay.js`
```javascript
export const config = { runtime: 'edge' };
export default async function handler(req) {
  if (req.method === 'OPTIONS') {
    return new Response(null, { status: 204, headers: {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': '*',
    }});
  }
  const url = new URL(req.url);
  const path = url.pathname.replace('/api/zama-relay', '');
  const target = `https://relayer.testnet.zama.org${path}${url.search ?? ''}`;
  const response = await fetch(target, {
    method: req.method,
    headers: { 'content-type': 'application/json' },
    body: req.method !== 'GET' && req.method !== 'HEAD' ? req.body : undefined,
  });
  // ✅ Forward actual content-type — binary endpoints MUST use arrayBuffer
  const contentType = response.headers.get('content-type') ?? 'application/octet-stream';
  const data = contentType.includes('octet-stream') || contentType.includes('binary')
    ? await response.arrayBuffer()
    : await response.text();
  return new Response(data, {
    status: response.status,
    headers: {
      'Access-Control-Allow-Origin': '*',
      'content-type': contentType,
    },
  });
}
```

### `frontend/index.html`
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title><ContractName></title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### `frontend/src/main.tsx`
```typescript
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode><App /></React.StrictMode>
);
```

### `frontend/src/config.ts`
```typescript
export const CONTRACT_ADDRESS = "<deployed-address-or-placeholder>";
export const CHAIN_ID = 11155111; // Sepolia
```

### `frontend/src/vendor/` (IMPORTANT)

Do NOT import @zama-fhe/relayer-sdk from npm. The SDK ships as a vendored bundle.
Tell the user:
```
Copy the vendor bundle from an existing fhEVM project:
  cp -r <existing-fhevm-project>/frontend/src/vendor frontend/src/vendor
```
Then import as: `import { initSDK, createInstance, SepoliaConfig } from "./vendor/relayer-sdk/web.js";`

### `frontend/src/App.tsx`

Write a complete React component:

```typescript
import { BrowserProvider, Contract } from "ethers";
import { initSDK, createInstance, SepoliaConfig } from "./vendor/relayer-sdk/web.js";
import { useState } from "react";
import ABI from "./abi.json";
import { CONTRACT_ADDRESS, CHAIN_ID } from "./config";

export default function App() {
  const [account, setAccount] = useState<string | null>(null);
  const [contract, setContract] = useState<Contract | null>(null);
  const [fhevmInst, setFhevmInst] = useState<any>(null);
  const [loading, setLoading] = useState(false);
  const [status, setStatus] = useState("");

  const connect = async () => {
    const eth = (window as any).ethereum;
    if (!eth) { setStatus("No wallet found"); return; }
    const provider = new BrowserProvider(eth);
    await provider.send("eth_requestAccounts", []);
    const network = await provider.getNetwork();
    if (Number(network.chainId) !== CHAIN_ID) {
      await provider.send("wallet_switchEthereumChain", [{ chainId: `0x${CHAIN_ID.toString(16)}` }]);
    }
    const signer = await provider.getSigner();
    const addr = await signer.getAddress();
    setAccount(addr);

    let inst: any = null;
    try {
      await initSDK();
      setStatus("Connecting to FHE relayer…");
      inst = await Promise.race([
        createInstance({ ...SepoliaConfig, network: eth, relayerUrl: `${window.location.origin}/api/zama-relay` }),
        new Promise((_, rej) => setTimeout(() => rej(new Error("timeout")), 60000))
      ]);
    } catch (e: any) {
      console.error("FHE init failed:", e);
      try { inst = await createInstance({ ...SepoliaConfig, network: eth }); } catch {}
    }
    setFhevmInst(inst);
    setContract(new Contract(CONTRACT_ADDRESS, ABI, signer));
    setStatus(inst ? "FHE online" : "FHE offline — read-only mode");
  };

  // ── Add action handlers here using encryptUint / userDecrypt ────────────────
  // See scaffold pattern below for encrypt/send/decrypt flows

  return (
    <div style={{ fontFamily: "monospace", padding: 32, maxWidth: 600, margin: "0 auto" }}>
      <h1><ContractName></h1>
      <div style={{ marginBottom: 16, fontSize: 12, color: fhevmInst ? "green" : "gray" }}>
        ● FHE {fhevmInst ? "online" : "offline"}
      </div>
      {!account
        ? <button onClick={connect}>Connect Wallet</button>
        : <div>
            <p>Connected: {account.slice(0,6)}…{account.slice(-4)}</p>
            <p>{status}</p>
            {/* Add your action buttons here */}
          </div>
      }
    </div>
  );
}
```

Customize the JSX to expose the contract's main functions (deposit, borrow, etc.) with:
- Input fields for amounts
- Encrypt → send flow using `fhevmInst.encryptUint`
- Decrypt → display flow using `fhevmInst.userDecrypt`
- Loading states and error display

---

## Step 6 — Output summary

After writing all files, print:

```
✅ Scaffolded <ContractName> — Vercel-deployable

Files written:
  contracts/<ContractName>.sol        — main contract
  test/<ContractName>.test.ts         — test suite
  scripts/deploy.ts                   — deploy script
  frontend/package.json
  frontend/vite.config.ts
  frontend/vercel.json                — COEP headers + relay rewrite
  frontend/index.html
  frontend/api/zama-relay.js          — CORS proxy (content-type aware)
  frontend/src/main.tsx
  frontend/src/App.tsx                — full React app
  frontend/src/config.ts

⚠️  Manual step required:
  cp -r <existing-fhevm-project>/frontend/src/vendor frontend/src/vendor

Next steps:
  1. npx hardhat compile && npx hardhat test
  2. npx hardhat run scripts/deploy.ts --network sepolia
  3. Update frontend/src/config.ts with deployed address
  4. cd frontend && npm install && npm run build
  5. Push to GitHub → connect Vercel → deploy
```

Do NOT ask the user to confirm each file — write all files then show the summary.
