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

| Field                  | Source       | Default if missing                  |
| ---------------------- | ------------ | ----------------------------------- |
| **Contract name**      | User message | `ConfidentialProtocol`              |
| **What to encrypt**    | User message | collateral + debt amounts           |
| **Roles needed**       | User message | ADMIN_ROLE only                     |
| **Decryption pattern** | User message | Pattern 3 (makePubliclyDecryptable) |
| **Token type**         | User message | ETH only                            |
| **Multi-asset?**       | User message | false                               |
| **Credit score gate?** | User message | false                               |

If the user says "lending" → use ConfidentialLending architecture (collateral/debt/health factor)
If the user says "voting" → use encrypted tallies + FHE.select + Pattern 3 public reveal (request + verify with handle pinning)
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

### 2a. MANDATORY pre-write checklist (read before writing a single line of contract code)

The scaffold templates below cover common mechanics but cannot anticipate every spec. Before writing the contract, cross-reference the spec against these five domain-correctness rules. Each one corresponds to a real bug observed in production fhEVM code — if you skip them, you will reproduce those bugs.

1. **Encrypted enum / branch domain validation** — does any function dispatch on an encrypted `euintN` (vote type, action type, asset kind)? If yes, you cannot `require` on an `ebool`. Designate one branch as the domain fallthrough and drive it via the negation of the other branches' union, so every input lands in exactly one bucket. See `03-input-acl.md` → "Multi-value input proof" and `08-anti-patterns.md` → "Silent-discard on out-of-range encrypted enum". **Invariant to state in a test:** `sum(tallies) == sum(accepted_weights)`.
2. **Pattern 3 verify-reveal handle pinning** — if you expose `verifyXxxReveal(bytes32[] handlesList, ...)`, the caller controls `handlesList`. `FHE.checkSignatures` does NOT bind handles to slots. Every callback MUST `require(handlesList[i] == FHE.toBytes32(expected_i))` for every `i` before calling `checkSignatures`. See `08-anti-patterns.md` → "Not pinning handles in checkSignatures".
3. **Don't role-gate VIEWS that return ciphertext handles** — handles are public IDs, not plaintext. Gating the view doesn't buy privacy (ACL does) but actively breaks callers that need the handle to build a verify proof. Role-gate the state-changing `requestXxxReveal` instead. See `08-anti-patterns.md` → "Role-gating a VIEW that returns ciphertext handles".
4. **Binary-search leak via callable encrypted predicate** — any function that returns `ebool` of `FHE.ge(secret, caller_supplied_threshold)` with `FHE.allow(result, msg.sender)` is a decryption oracle. Gate by subject-or-role, not open. See `08-anti-patterns.md` → "Binary-search leak".
5. **User-supplied encrypted "equivalent" clamping** — if a user encrypts a value meant to represent a plaintext-derivable quantity (token→ETH conversion, collateral credit), clamp via `FHE.select(lt(encCap, encIn), encCap, encIn)` against an oracle-derived plaintext cap. Otherwise users over-report freely. See `08-anti-patterns.md` → "Trusting user-supplied encrypted equivalent".

Also apply these gas/ACL hygiene rules from `02-types-ops.md`:
- Only `FHE.allowThis` values that will survive the current call (storage LHS, return value, or cross-contract arg). Skip it on scratch predicates/deltas (~30k saved per grant).
- Prefer `FHE.shr` over `FHE.div` for powers of two.
- Use the floor-at-zero form for `FHE.sub` (no `FHE.gte` in v0.11 — use `FHE.not(FHE.lt(a, b))`).

If the spec mentions multi-branch encrypted dispatch (voting, asset type, action kind), write and commit to rule 1 BEFORE writing the function body. If the spec mentions liquidation/close/reveal with Pattern 3, write and commit to rule 2 BEFORE writing `verifyXxxReveal`. Do not fold these in as afterthoughts.

### 2b. Write the contract

Write the full contract to `contracts/<ContractName>.sol` following these rules:

**Imports (always):**

```solidity
pragma solidity ^0.8.24;
import {FHE, euint64, ebool, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";   // npm — do NOT copy locally
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
    await contract
      .connect(user)
      .deposit(enc.handles[0], enc.inputProof, {
        value: ethers.parseEther("0.1"),
      });
    expect(await contract.isActive(user.address)).to.be.true;
  });

  it("should decrypt user value correctly", async function () {
    // ... deposit first ...
    const handle = await contract.getEncrypted<Field>(user.address);
    const { publicKey, privateKey } = fhevm.generateKeypair();
    const contractAddr = await contract.getAddress();
    const now = Math.floor(Date.now() / 1000);
    const eip712 = fhevm.createEIP712(publicKey, [contractAddr], now, 1);
    // SDK v0.4.x exposes the active EIP-712 primaryType directly; don't guess type keys.
    const { primaryType } = eip712;
    const sig = await user.signTypedData(
      eip712.domain,
      { [primaryType]: eip712.types[primaryType] },
      eip712.message,
    );
    const results = await fhevm.userDecrypt(
      [{ handle, contractAddress: contractAddr }],
      privateKey,
      publicKey,
      sig,
      [contractAddr],
      user.address,
      now,
      1,
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
import { ethers, network, run } from "hardhat";

async function main() {
  const [deployer] = await ethers.getSigners();
  console.log("Deployer:", deployer.address);

  const Factory = await ethers.getContractFactory("<ContractName>");
  // If the constructor takes an admin, pass deployer.address (or whatever the spec requires).
  const contract = await Factory.deploy(/* <constructor args, if any> */);
  await contract.waitForDeployment();
  const addr = await contract.getAddress();
  console.log("<ContractName>:", addr);
  console.log("Etherscan:", `https://sepolia.etherscan.io/address/${addr}`);

  // Wire any companion contracts here
  // Grant roles if needed: await contract.grantRole(...)

  console.log("\nUpdate frontend/src/config.ts:");
  console.log(`  CONTRACT_ADDRESS = "${addr}"`);

  // ─── Auto-verify on Etherscan (skip on local networks) ────────────────────
  if (network.name !== "hardhat" && network.name !== "localhost") {
    console.log("\nWaiting for Etherscan indexing (30s)...");
    await new Promise((r) => setTimeout(r, 30_000));

    try {
      await run("verify:verify", {
        address: addr,
        constructorArguments: [/* same args as deploy */],
      });
      console.log("Etherscan verification submitted");
    } catch (err: any) {
      const msg = String(err?.message ?? err);
      if (/already verified/i.test(msg)) {
        console.log("Already verified on Etherscan");
      } else {
        // Non-fatal — verification can be retried manually
        console.warn("Etherscan verify failed (non-fatal):", msg);
      }
    }
  }
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

Requires `@nomicfoundation/hardhat-verify` and an `ETHERSCAN_API_KEY` in `hardhat.config.ts`. Keep `constructorArguments` in sync with the deploy `.deploy(...)` call — mismatches are the #1 cause of verify failure.

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
    "@zama-fhe/relayer-sdk": "^0.4.1",
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

Note: install `@zama-fhe/relayer-sdk` from npm and import from `@zama-fhe/relayer-sdk/bundle` (self-contained WASM bundle that works out of the box on Vite/Vercel). Do NOT vendor-copy the SDK — earlier skill revisions recommended that, but v0.4.x exports `./bundle` and `./web` subpaths cleanly. Keep `optimizeDeps.exclude` below so Vite leaves the WASM bundle alone.

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
export const config = { runtime: "edge" };

const RELAYER_ORIGIN = "https://relayer.testnet.zama.org";

// Allow-list: only the request headers the relayer actually needs. Anything set
// on our own origin (cookie, authorization, host, origin, referer, user-agent)
// is dropped so the proxy never leaks credentials to the third-party relayer.
const FORWARD_REQ_HEADERS = new Set([
  "content-type",
  "accept",
  "accept-encoding",
  "x-zama-client",
  "zama-sdk-version",
  "zama-sdk-name",
]);

function filterRequestHeaders(src) {
  const out = new Headers();
  for (const [k, v] of src.entries()) {
    if (FORWARD_REQ_HEADERS.has(k.toLowerCase())) out.set(k, v);
  }
  return out;
}

export default async function handler(req) {
  if (req.method === "OPTIONS") {
    return new Response(null, {
      status: 204,
      headers: {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
        "Access-Control-Allow-Headers": "*",
      },
    });
  }

  const url = new URL(req.url);
  const path = url.pathname.replace(/^\/api\/zama-relay/, "") || "/";
  const target = `${RELAYER_ORIGIN}${path}${url.search ?? ""}`;

  const response = await fetch(target, {
    method: req.method,
    headers: filterRequestHeaders(req.headers),  // ✅ preserves ZAMA-SDK-* but strips cookies/auth
    body: req.method !== "GET" && req.method !== "HEAD" ? req.body : undefined,
  });

  // ✅ Forward actual content-type — binary endpoints MUST use arrayBuffer
  const contentType =
    response.headers.get("content-type") ?? "application/octet-stream";
  const data =
    contentType.includes("application/octet-stream") ||
    contentType.includes("binary")
      ? await response.arrayBuffer()
      : await response.text();

  return new Response(data, {
    status: response.status,
    headers: {
      "Access-Control-Allow-Origin": "*",
      "content-type": contentType,
      "ZAMA-SDK-VERSION": response.headers.get("ZAMA-SDK-VERSION") ?? "",
      "ZAMA-SDK-NAME": response.headers.get("ZAMA-SDK-NAME") ?? "",
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

### SDK install (IMPORTANT — use npm, NOT vendor-copy)

Install the relayer SDK directly from npm — the package's `./bundle` and `./web` subpath exports are supported in v0.4.x, so no vendor-copy is needed:

```bash
cd frontend && npm install @zama-fhe/relayer-sdk@^0.4.1
```

Import the `/bundle` entrypoint everywhere (self-contained WASM, works under Vite/Vercel without extra config):

```typescript
import { initSDK, createInstance, SepoliaConfig, type FhevmInstance } from "@zama-fhe/relayer-sdk/bundle";
```

If you need the tree-shakeable variant (advanced — requires manual WASM asset hosting), use `@zama-fhe/relayer-sdk/web` instead. Do NOT copy `src/vendor/` from another project; older skill revisions recommended that, but the npm package now publishes a drop-in bundle.

### `frontend/src/App.tsx`

Write a complete React component:

```typescript
import { BrowserProvider, Contract } from "ethers";
import { initSDK, createInstance, SepoliaConfig, type FhevmInstance } from "@zama-fhe/relayer-sdk/bundle";
import { useState } from "react";
import ABI from "./abi.json";
import { CONTRACT_ADDRESS, CHAIN_ID } from "./config";

// Params used by wallet_addEthereumChain when the user's wallet doesn't know about Sepolia.
// Without this, calling wallet_switchEthereumChain throws error 4902 and leaves the user stuck.
const SEPOLIA_ADD_PARAMS = {
  chainId: "0xaa36a7",
  chainName: "Sepolia",
  nativeCurrency: { name: "Sepolia Ether", symbol: "SEP", decimals: 18 },
  rpcUrls: ["https://rpc.sepolia.org"],
  blockExplorerUrls: ["https://sepolia.etherscan.io"],
} as const;

// Switch the wallet to Sepolia; if the wallet doesn't know it (error 4902), add it first.
async function ensureSepolia(eth: any): Promise<void> {
  try {
    await eth.request({
      method: "wallet_switchEthereumChain",
      params: [{ chainId: SEPOLIA_ADD_PARAMS.chainId }],
    });
  } catch (err: any) {
    if (err?.code === 4902) {
      await eth.request({ method: "wallet_addEthereumChain", params: [SEPOLIA_ADD_PARAMS] });
    } else {
      throw err;
    }
  }
}

export default function App() {
  const [account, setAccount] = useState<string | null>(null);
  const [contract, setContract] = useState<Contract | null>(null);
  const [fhevmInst, setFhevmInst] = useState<FhevmInstance | null>(null);
  const [loading, setLoading] = useState(false);
  const [status, setStatus] = useState("");

  const connect = async () => {
    const eth = (window as any).ethereum;
    if (!eth) { setStatus("No wallet found"); return; }

    // Resolve chain BEFORE constructing BrowserProvider — ethers v6 caches chainId
    // and will throw NETWORK_ERROR if the chain changes after provider construction.
    let provider = new BrowserProvider(eth);
    await provider.send("eth_requestAccounts", []);
    const network = await provider.getNetwork();
    if (Number(network.chainId) !== CHAIN_ID) {
      await ensureSepolia(eth);
      provider = new BrowserProvider(eth);   // re-create after chain switch
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

Next steps:
  1. npx hardhat compile && npx hardhat test
  2. npx hardhat run scripts/deploy.ts --network sepolia
  3. Update frontend/src/config.ts with deployed address
  4. cd frontend && npm install && npm run build   # installs @zama-fhe/relayer-sdk
  5. Push to GitHub → connect Vercel → deploy
```

Do NOT ask the user to confirm each file — write all files then show the summary.
