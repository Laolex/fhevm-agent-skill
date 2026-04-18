---
name: fhevm-anti-patterns
description: fhEVM common anti-patterns and mistakes to avoid — TFHE, missing ACL, if/else on ebool, initFhevm, encrypted divisor, old Gateway API, old reencrypt API (v0.4.1), ethers v6 provider stale, public storage
type: reference
---

# fhEVM — Anti-Patterns (Must Avoid)

## ❌ Using TFHE instead of FHE
```solidity
TFHE.asEuint64(100)    // ❌ outdated — removed in v0.9
FHE.asEuint64(100)     // ✅
```

## ❌ Emitting encrypted values in events
```solidity
emit Transfer(from, to, encryptedAmount);  // ❌ handle is public — privacy leak
emit Transfer(from, to, block.timestamp);  // ✅ emit only metadata/timestamps
```

## ❌ Missing ACL grants after computation
```solidity
euint64 newTotal = FHE.add(a, b);
storage[user] = newTotal;             // ❌ contract can't use newTotal in next call

// ✅ Fix:
FHE.allowThis(newTotal);
FHE.allow(newTotal, user);
storage[user] = newTotal;
```

## ❌ Using if/else on ebool
```solidity
ebool condition = FHE.gt(a, b);
if (condition) { ... }    // ❌ compile error — ebool is not bool

// ✅ Fix: use FHE.select()
euint64 result = FHE.select(condition, valueIfTrue, valueIfFalse);
```

## ❌ Calling initFhevm() in frontend
```typescript
import { initFhevm } from "@zama-fhe/relayer-sdk/web.js";
await initFhevm();   // ❌ removed — causes "invalid EIP-1193 provider" error

// ✅ Just call createInstance() directly
```

## ❌ FHE.div with encrypted divisor
```solidity
euint64 bad  = FHE.div(a, b);   // ❌ b cannot be euint64 — compile error
euint64 good = FHE.div(a, 2);   // ✅ divisor must be plaintext uint64
// Better: FHE.shr(a, 1) for ÷2, FHE.shr(a, 2) for ÷4 — cheaper gas
```

## ❌ Using fhevmjs package (deprecated)
```typescript
import { createInstance } from "fhevmjs";                   // ❌ deprecated
import { createInstance } from "@zama-fhe/relayer-sdk/web.js"; // ✅
```

## ❌ Old reencrypt() API (removed in SDK v0.4.1)
```typescript
// ❌ reencrypt() removed — causes "is not a function" or "invalid signature"
const eip712 = fhevmInstance.createEIP712(publicKey, CONTRACT_ADDRESS);        // ❌ old 2-arg form
const sig = await signer.signTypedData(
    eip712.domain, { Reencrypt: eip712.types.Reencrypt }, eip712.message
);
const plaintext = await fhevmInstance.reencrypt(                               // ❌ removed
    encHandle, privateKey, publicKey, sig, CONTRACT_ADDRESS, userAddress
);

// ✅ SDK v0.4.1 — use userDecrypt with HandleContractPair[]
const now = Math.floor(Date.now() / 1000);
const eip712 = fhevmInstance.createEIP712(publicKey, [CONTRACT_ADDRESS], now, 1);  // ✅ array + timestamp + days
const typeName = eip712.types.Reencrypt
    ? "Reencrypt"
    : Object.keys(eip712.types).find((k: string) => k !== "EIP712Domain")!;
const sig = await signer.signTypedData(eip712.domain, { [typeName]: eip712.types[typeName] }, eip712.message);
const results = await fhevmInstance.userDecrypt(                                    // ✅ new API
    [{ handle: encHandle, contractAddress: CONTRACT_ADDRESS }],
    privateKey, publicKey, sig, [CONTRACT_ADDRESS], userAddress, now, 1,
);
const plaintext = Object.values(results)[0] as bigint;
```

## ❌ ethers v6 BrowserProvider stale after chain switch
```typescript
// ❌ BrowserProvider caches chainId — becomes invalid after wallet_switchEthereumChain
const provider = new ethers.BrowserProvider(window.ethereum);
await provider.send("wallet_switchEthereumChain", [{ chainId: "0xaa36a7" }]);
const signer = await provider.getSigner();  // ❌ NETWORK_ERROR: cached chainId mismatches

// ✅ Re-create BrowserProvider after chain switch
const eth = window.ethereum;
await eth.request({ method: "wallet_switchEthereumChain", params: [{ chainId: "0xaa36a7" }] });
const provider = new ethers.BrowserProvider(eth);  // ✅ fresh instance picks up new chain
const signer = await provider.getSigner();
```

## ❌ Storing encrypted values as public
```solidity
mapping(address => euint64) public balances;  // ❌ handle exposed to world
mapping(address => euint64) private balances; // ✅
```

## ❌ Missing config inheritance
```solidity
contract MyContract {                          // ❌ FHE operations will fail at runtime
contract MyContract is SepoliaConfig {         // ✅
```

## ❌ Comparing encrypted values with ==
```solidity
if (encA == encB) { ... }              // ❌ compiles but checks handle equality, not values
ebool equal = FHE.eq(encA, encB);      // ✅ actual encrypted comparison
```

## ❌ Old Gateway.requestDecryption API
```solidity
// ❌ ALL of this is wrong — will not compile with current @fhevm/solidity
import {GatewayCaller} from "@fhevm/solidity/gateway/GatewayCaller.sol";
import {Gateway} from "@fhevm/solidity/gateway/lib/Gateway.sol";

uint256[] memory cts = new uint256[](1);          // ❌ wrong type
cts[0] = Gateway.toUint256(encryptedTally);        // ❌ wrong method
Gateway.requestDecryption(cts, selector, 0, deadline, false); // ❌ wrong function
function cb(uint256 id, uint64 result) public onlyGateway {}  // ❌ wrong signature

// ✅ CORRECT
bytes32[] memory cts = new bytes32[](1);
cts[0] = FHE.toBytes32(encryptedTally);
FHE.requestDecryption(cts, this.cb.selector);
function cb(uint256 id, bytes memory cleartexts, bytes memory proof) external {
    FHE.checkSignatures(id, cleartexts, proof);
    (uint64 result) = abi.decode(cleartexts, (uint64));
}
```

## ❌ Missing FHE.checkSignatures in decryption callbacks
```solidity
function onRevealCallback(uint256 requestId, bytes memory cleartexts, bytes memory proof) external {
    // ❌ Skipping checkSignatures — anyone can forge a decryption result
    (uint64 result) = abi.decode(cleartexts, (uint64));

    // ✅ Always verify first
    FHE.checkSignatures(requestId, cleartexts, proof);
    (uint64 result) = abi.decode(cleartexts, (uint64));
}
```

## ❌ CRITICAL: Not pinning handles in `FHE.checkSignatures(handlesList, ...)` callbacks
```solidity
// Pattern 3 (makePubliclyDecryptable + off-chain publicDecrypt) passes a handlesList
// that the CALLER controls. `checkSignatures` only attests "KMS signed these handles
// decrypt to these cleartexts" — it does NOT pin handles to any expected slot.
//
// ❌ VULNERABLE — attacker substitutes a publicly-decryptable euint64(0) handle to
//    forge debt==0, or substitutes a true-handle to flip isLiquidatable on a solvent user.
function verifyAndClose(bytes32[] calldata handlesList, bytes calldata cleartexts, bytes calldata proof) external {
    require(pendingClose[msg.sender], "No pending close");
    FHE.checkSignatures(handlesList, cleartexts, proof);   // ❌ handlesList[0] unchecked
    (uint64 debt) = abi.decode(cleartexts, (uint64));
    require(debt == 0, "Debt not zero");
    _close(msg.sender);
}

// ✅ CORRECT — pin every handle to the exact ciphertext the contract produced
function verifyAndClose(bytes32[] calldata handlesList, bytes calldata cleartexts, bytes calldata proof) external {
    require(pendingClose[msg.sender], "No pending close");
    require(handlesList.length == 1, "Expected 1 handle");
    require(
        handlesList[0] == FHE.toBytes32(positions[msg.sender].totalDebt),  // ✅ pinned
        "Handle mismatch"
    );
    FHE.checkSignatures(handlesList, cleartexts, proof);
    (uint64 debt) = abi.decode(cleartexts, (uint64));
    require(debt == 0, "Debt not zero");
    _close(msg.sender);
}
```
**Rule:** every Pattern-3 `FHE.checkSignatures(bytes32[], bytes, bytes)` callback MUST `require(handlesList[i] == FHE.toBytes32(expected_i))` for every `i` before calling `checkSignatures`. Pattern 2 (`FHE.requestDecryption` with requestId) is safe — the coprocessor binds the request.

## ❌ CRITICAL: Trusting user-supplied encrypted "equivalent" without a plaintext clamp
```solidity
// User deposits 1 USDC (plaintext) but encrypts 1e18 wei of "ETH-equivalent" (ciphertext)
// as collateral credit. Contract has no way to verify the encrypted value matches the
// real deposit — collateral model breaks.
//
// ❌ VULNERABLE
function depositToken(address token, uint256 amt, externalEuint64 enc, bytes calldata proof) external {
    IERC20(token).safeTransferFrom(msg.sender, address(this), amt);
    euint64 encEquiv = FHE.fromExternal(enc, proof);        // ❌ user-controlled value
    FHE.allowThis(encEquiv);
    _addCollateral(msg.sender, encEquiv);                   // ❌ trusted blindly
}

// ✅ CORRECT — clamp via FHE.select against a plaintext oracle-derived cap
function depositToken(address token, uint256 amt, externalEuint64 enc, bytes calldata inputProof) external {
    TokenConfig memory cfg = tokenConfigs[token];
    require(cfg.supported, "Unsupported token");
    IERC20(token).safeTransferFrom(msg.sender, address(this), amt);

    uint256 price  = oracle.getEthWeiPerToken(token);       // plaintext, oracle-derived
    uint256 capWei = (amt * price) / (10 ** cfg.decimals);
    if (capWei > type(uint64).max) capWei = type(uint64).max;

    euint64 encEquiv = FHE.fromExternal(enc, inputProof);
    FHE.allowThis(encEquiv);
    euint64 encCap   = FHE.asEuint64(uint64(capWei));
    FHE.allowThis(encCap);
    euint64 clamped  = FHE.select(FHE.lt(encCap, encEquiv), encCap, encEquiv); // ✅ min(cap, encIn)
    FHE.allowThis(clamped);
    _addCollateral(msg.sender, clamped);
}
```
**Rule:** whenever a user submits an encrypted value that is supposed to represent a plaintext-observable quantity (token→ETH conversion, USD value, collateral credit), always compute a plaintext upper bound and clamp via `FHE.select(lt(cap, enc), cap, enc)`. Otherwise users over-report freely.

## ❌ HIGH: Binary-search leak via callable predicate on encrypted values
```solidity
// ❌ VULNERABLE — anyone can sweep thresholds and decrypt each ebool result,
//    binary-searching the encrypted score / balance / vote count.
function meetsThreshold(address subject, uint64 threshold) external returns (ebool) {
    require(scores[subject].active, "No score");
    euint64 encScore = scores[subject].score;
    ebool   result   = FHE.ge(encScore, FHE.asEuint64(threshold));
    FHE.allow(result, msg.sender);    // ❌ grants ACL to anyone who calls
    return result;
}

// ✅ CORRECT — role-gate the predicate. Only the subject or a whitelisted reader
//    (e.g. the lending contract that integrates ShieldScore) may probe.
bytes32 public constant READER_ROLE = keccak256("READER_ROLE");

function grantReaderRole(address who) external onlyRole(DEFAULT_ADMIN_ROLE) {
    _grantRole(READER_ROLE, who);
}

function meetsThreshold(address subject, uint64 threshold) external returns (ebool) {
    require(scores[subject].active, "No score");
    require(
        msg.sender == subject || hasRole(READER_ROLE, msg.sender),   // ✅ gated
        "Not authorised reader"
    );
    euint64 encScore = scores[subject].score;
    ebool   result   = FHE.ge(encScore, FHE.asEuint64(threshold));
    FHE.allow(result, msg.sender);
    return result;
}
```
**Rule:** any function that returns an encrypted predicate over private data is a decryption oracle for anyone who holds ACL. Gate it by role (`READER_ROLE`, consumer contract) or require `msg.sender == subject`. Binary search on 10-bit values is 10 calls.

## ❌ FHE.sub underflow without guard
```solidity
euint64 result = FHE.sub(a, b);  // ❌ wraps to huge number if a < b

// ✅ Floor at zero (v0.11 has no FHE.gte — use not+lt)
euint64 result = FHE.select(FHE.not(FHE.lt(a, b)), FHE.sub(a, b), FHE.asEuint64(0));
```

## ❌ Trailing newline in Vercel env var (ENS invalid name)
```bash
# ❌ Causes "invalid ENS name (disallowed character: {0A})" — ethers tries to resolve as ENS
vercel env add VITE_SCORE_CONTRACT_ADDRESS <<< "0xSomeAddress"  # ❌ here-string adds \n

# ✅ Use printf to avoid trailing newline
printf '0xSomeAddress' | vercel env add VITE_SCORE_CONTRACT_ADDRESS production

# ✅ Also add .trim() in config.ts as a safety net
export const SCORE_CONTRACT_ADDRESS = (import.meta.env.VITE_SCORE_CONTRACT_ADDRESS ?? "").trim();
```

## ❌ Missing vercel.json rewrite rule for CORS proxy sub-paths
```json
// ❌ Without this, SDK calls like /api/zama-relay/keys/...public return 404
// The fhEVM SDK calls multiple sub-paths of the relayer — all must route to the edge function
{
  "rewrites": []  // ❌ empty — FHE init will timeout silently
}

// ✅ Required — catches all sub-path calls
{
  "rewrites": [
    { "source": "/api/zama-relay/:path*", "destination": "/api/zama-relay" }
  ]
}
```

## ❌ Missing OPTIONS handler in Vercel edge function
```javascript
// ❌ Without OPTIONS, browser CORS preflight fails and FHE init hangs
export default async function handler(req) {
    const response = await fetch(target, ...);  // ❌ no preflight handling
}

// ✅ Handle OPTIONS first
if (req.method === 'OPTIONS') {
    return new Response(null, { status: 204, headers: { 'Access-Control-Allow-Origin': '*', ... } });
}
```

## ❌ Using AccessControl instead of AccessControlEnumerable for multi-liquidator ACL
```solidity
contract Foo is AccessControl { ... }  // ❌ can't iterate role members

// If you need to grant ACL to all role holders:
function _allowLiquidators(ebool ct) internal {
    getRoleMemberCount(ROLE);   // ❌ only on AccessControlEnumerable
    getRoleMember(ROLE, i);     // ❌ only on AccessControlEnumerable
}

// ✅ Use AccessControlEnumerable
import {AccessControlEnumerable} from "@openzeppelin/contracts/access/extensions/AccessControlEnumerable.sol";
contract Foo is AccessControlEnumerable { ... }
```

## ❌ Proxy hardcoding request headers — drops ZAMA-SDK headers, FHE goes offline
```javascript
// ❌ Overriding headers drops ZAMA-SDK-VERSION and ZAMA-SDK-NAME
// Zama relayer REQUIRES these headers — without them it silently rejects and FHE init fails
const response = await fetch(target, {
    method: req.method,
    headers: { 'content-type': 'application/json' },  // ❌ drops all original headers
    body: ...,
});

// ✅ Forward original request headers, only stripping hop-by-hop headers
const forwardHeaders = {};
for (const [key, value] of req.headers.entries()) {
    if (!['host', 'connection'].includes(key.toLowerCase())) {
        forwardHeaders[key] = value;  // ✅ preserves ZAMA-SDK-VERSION, ZAMA-SDK-NAME, etc.
    }
}
const response = await fetch(target, { method: req.method, headers: forwardHeaders, body: ... });
```

## ❌ Duplicate env var declarations without .trim() — ENS invalid name {0A}
```typescript
// ❌ Each component declaring its own SCORE_CONTRACT_ADDRESS without .trim()
// If the env var was set with a trailing newline (from echo pipe), this causes:
// "invalid ENS name (disallowed character: {0A})" — ethers tries to resolve 0x...\n as ENS
export const SCORE_CONTRACT_ADDRESS = import.meta.env.VITE_SCORE_CONTRACT_ADDRESS ?? ""; // ❌ no .trim()

// ✅ Declare once in config.ts with .trim(), import everywhere else
// config.ts:
export const SCORE_CONTRACT_ADDRESS = (import.meta.env.VITE_SCORE_CONTRACT_ADDRESS ?? "").trim();

// ShieldScore.tsx, BorrowerCard.tsx, etc.:
import { SCORE_CONTRACT_ADDRESS } from "./config"; // ✅ single source of truth + .trim()

// Also: set env vars with printf to avoid trailing newlines:
// ❌ echo "0x..." | vercel env add ...   (adds \n)
// ✅ printf '0x...' | vercel env add ...
```

## ❌ Etherscan v2 API with hardhat-verify (Etherscan migrated, plugin broken)
```
// ❌ hardhat-verify ≤ 2.1.3 sends to old v1 API — returns "Invalid API Key" or "deprecated V1"
npx hardhat verify --network sepolia 0xAddress

// ✅ Use manual Standard-JSON-Input verification instead:
// 1. Find artifacts/build-info/<hash>.json
// 2. Extract .input field: python3 -c "import json,sys; print(json.dumps(json.load(open('artifacts/build-info/<hash>.json'))['input'], indent=2))" > std_input.json
// 3. Upload at https://sepolia.etherscan.io/verifyContract
//    Settings: Solidity (Standard-Json-Input), compiler 0.8.24, 200 runs, EVM cancun
```
