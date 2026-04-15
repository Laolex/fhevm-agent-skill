---
name: fhevm-decryption
description: fhEVM decryption patterns — userDecrypt (SDK v0.4.1, batch HandleContractPair), FHE.requestDecryption oracle callback (public reveal), FHE.makePubliclyDecryptable relayer pattern
type: reference
---

# fhEVM — Decryption Patterns

Three patterns. Choose based on who needs the value and when.

| | Pattern 1 | Pattern 2 | Pattern 3 |
|---|---|---|---|
| **API** | `fhevmInstance.userDecrypt` | `FHE.requestDecryption` | `FHE.makePubliclyDecryptable` |
| **Who sees it** | Only the authorized user | Publicly — via callback | Publicly — via relayer |
| **Contract receives plaintext** | No | Yes — in callback | No |
| **Use when** | User decrypts their own position | Contract logic needs the result | Only need to publish the value |

---

## Pattern 1 — User Decrypt (user decrypts their own data)

> **SDK v0.4.1 breaking change**: `reencrypt()` is removed. Use `userDecrypt()` instead.
> `createEIP712` signature changed from `(pubKey, contractAddr)` to `(pubKey, contractAddresses[], startTimestamp, durationDays)`.

**Contract:**
```solidity
function getEncryptedSalary(address employee) external view returns (euint64) {
    return salaries[employee];
    // ACL must already allow caller — fhEVM checks automatically
}
```

**Frontend — single handle:**
```typescript
import { createInstance } from "@zama-fhe/relayer-sdk/web.js";

const encHandle = await contract.getEncryptedSalary(userAddress);
const { publicKey, privateKey } = fhevmInstance.generateKeypair();

// ✅ SDK v0.4.1: createEIP712(pubKey, contractAddresses[], startTimestamp, durationDays)
const now = Math.floor(Date.now() / 1000);
const eip712 = fhevmInstance.createEIP712(publicKey, [CONTRACT_ADDRESS], now, 1);

// ✅ Sign EIP-712 — type name may vary between SDK versions
const typeName = eip712.types.Reencrypt
    ? "Reencrypt"
    : Object.keys(eip712.types).find((k: string) => k !== "EIP712Domain")!;
const signature = await signer.signTypedData(
    eip712.domain,
    { [typeName]: eip712.types[typeName] },
    eip712.message
);

// ✅ SDK v0.4.1: userDecrypt(HandleContractPair[], privKey, pubKey, sig, contractAddresses[], userAddr, startTs, days)
const handles = [{ handle: encHandle, contractAddress: CONTRACT_ADDRESS }];
const results = await fhevmInstance.userDecrypt(
    handles, privateKey, publicKey, signature,
    [CONTRACT_ADDRESS], userAddress, now, 1,
);
// results is Record<0xHandleHex, bigint | boolean | string>
const plaintext = Object.values(results)[0] as bigint;
```

**Frontend — batch decrypt (multiple handles, one signature):**
```typescript
// Fetch all encrypted handles from contract
const [collHandle, debtHandle, rateHandle, scoreHandle] = await Promise.all([
    contract.getEncCollateral(userAddress),
    contract.getEncDebt(userAddress),
    contract.getEncRate(userAddress),
    contract.getEncScore(userAddress),
]);

const { publicKey, privateKey } = fhevmInstance.generateKeypair();
const now = Math.floor(Date.now() / 1000);
const eip712 = fhevmInstance.createEIP712(publicKey, [CONTRACT_ADDRESS], now, 1);
const typeName = eip712.types.Reencrypt
    ? "Reencrypt"
    : Object.keys(eip712.types).find((k: string) => k !== "EIP712Domain")!;
const sig = await signer.signTypedData(
    eip712.domain, { [typeName]: eip712.types[typeName] }, eip712.message
);

// ✅ All handles in one call — single network round-trip
const handles = [collHandle, debtHandle, rateHandle, scoreHandle].map(
    (h: string) => ({ handle: h, contractAddress: CONTRACT_ADDRESS })
);
const results = await fhevmInstance.userDecrypt(
    handles, privateKey, publicKey, sig,
    [CONTRACT_ADDRESS], userAddress, now, 1,
);
// Results keyed by handle hex — iterate in order
const vals = Object.values(results) as bigint[];
const [collateral, debt, rate, score] = vals;
```

**Multi-contract decrypt (handles from different contracts):**
```typescript
// When handles come from multiple contracts, list all contract addresses
const handles = [
    { handle: lendingHandle, contractAddress: LENDING_ADDRESS },
    { handle: scoreHandle, contractAddress: SCORE_ADDRESS },
];
const eip712 = fhevmInstance.createEIP712(
    publicKey, [LENDING_ADDRESS, SCORE_ADDRESS], now, 1
);
// ... sign and call userDecrypt with [LENDING_ADDRESS, SCORE_ADDRESS]
```

---

## Pattern 2 — FHE oracle / public decryption (ERC-7995)
Used when contract logic needs the plaintext after decryption (vote tallies, market resolution).
**Current API — do NOT use old Gateway.* approach.**

```solidity
// ✅ No GatewayCaller, no SepoliaZamaGatewayConfig, no onlyGateway
contract VotingContract is SepoliaConfig {
    euint64 private encryptedTally;
    uint64  public revealedTally;
    bool    public isRevealed;
    mapping(uint256 => bool) public callbackCalled;

    function requestReveal() external {
        require(block.timestamp > endTime, "Not expired");
        require(!isRevealed, "Already revealed");

        bytes32[] memory cts = new bytes32[](1);
        cts[0] = FHE.toBytes32(encryptedTally);
        uint256 requestId = FHE.requestDecryption(cts, this.onRevealCallback.selector);
    }

    // ✅ Callback signature: (requestId, bytes cleartexts, bytes decryptionProof)
    function onRevealCallback(
        uint256 requestId,
        bytes memory cleartexts,
        bytes memory decryptionProof
    ) external {
        FHE.checkSignatures(requestId, cleartexts, decryptionProof); // ✅ REQUIRED
        (uint64 result) = abi.decode(cleartexts, (uint64));
        revealedTally = result;
        isRevealed = true;
        callbackCalled[requestId] = true;
    }
}

// ✅ Multiple values — request and decode in same order
function requestMultiReveal() external {
    bytes32[] memory cts = new bytes32[](3);
    cts[0] = FHE.toBytes32(forVotes);
    cts[1] = FHE.toBytes32(againstVotes);
    cts[2] = FHE.toBytes32(abstainVotes);
    uint256 requestId = FHE.requestDecryption(cts, this.onMultiCallback.selector);
    requestIdToProposal[requestId] = proposalId;
}

function onMultiCallback(uint256 requestId, bytes memory cleartexts, bytes memory decryptionProof) external {
    FHE.checkSignatures(requestId, cleartexts, decryptionProof);
    (uint64 revFor, uint64 revAgainst, uint64 revAbstain) =
        abi.decode(cleartexts, (uint64, uint64, uint64));
}
```

### ❌ Old Gateway API — do NOT use
```solidity
// ❌ ALL of these are wrong/outdated
import {GatewayCaller} from "@fhevm/solidity/gateway/GatewayCaller.sol";
import {Gateway} from "@fhevm/solidity/gateway/lib/Gateway.sol";
uint256[] memory cts = new uint256[](1);         // ❌ wrong type (must be bytes32[])
cts[0] = Gateway.toUint256(encryptedTally);       // ❌ wrong method
Gateway.requestDecryption(cts, selector, 0, deadline, false); // ❌ wrong function
function cb(uint256 requestId, uint64 result) public onlyGateway {} // ❌ wrong signature
```

---

## Pattern 3 — Relayer-assisted public decryption (v0.11.1)
Used when only the value needs publishing — contract doesn't need the plaintext itself.
Available in `@fhevm/solidity ^0.11.1`.

```solidity
mapping(address => bool) public pendingReveal;

function requestReveal(address subject) external {
    require(positions[subject].active, "No active position");
    ebool flag = positions[subject].isLiquidatable;

    FHE.makePubliclyDecryptable(flag);  // mark for relayer
    pendingReveal[subject] = true;

    emit RevealRequested(subject, FHE.toBytes32(flag), block.timestamp);
}

// Relayer calls this after off-chain decryption
function verifyReveal(
    address subject,
    bytes32[] calldata handlesList,
    bytes calldata abiEncodedCleartexts,
    bytes calldata decryptionProof
) external {
    require(pendingReveal[subject], "No pending reveal");
    FHE.checkSignatures(handlesList, abiEncodedCleartexts, decryptionProof); // ✅ array form
    bool isLiquidatable = abi.decode(abiEncodedCleartexts, (bool));
    delete pendingReveal[subject];
    emit RevealResult(subject, isLiquidatable, block.timestamp);
}

// ✅ Pattern 3 checkSignatures signature (array form — different from Pattern 2):
// FHE.checkSignatures(bytes32[] handlesList, bytes abiEncodedCleartexts, bytes decryptionProof)

// ✅ Pattern 2 checkSignatures signature (requestId form):
// FHE.checkSignatures(uint256 requestId, bytes cleartexts, bytes proof)
```

---

## Production Pattern 3: 3-step liquidation flow (ShieldLend)
Used when a third party (anyone) initiates reveal, then a permissioned role executes.

```solidity
mapping(address => bool) private pendingLiquidationReveal;
mapping(address => bool) private confirmedLiquidatable;   // gate for step 3

// Step 1 — anyone can trigger a reveal (permissionless oracle query)
function requestLiquidationReveal(address borrower) external {
    require(positions[borrower].active, "No active position");
    ebool liqFlag = positions[borrower].isLiquidatable;
    FHE.makePubliclyDecryptable(liqFlag);
    pendingLiquidationReveal[borrower] = true;
    emit LiquidationRevealRequested(borrower, FHE.toBytes32(liqFlag), block.timestamp);
}

// Step 2 — relayer submits decryption result (anyone can submit)
function verifyLiquidationReveal(
    address borrower,
    bytes32[] calldata handlesList,
    bytes calldata abiEncodedCleartexts,
    bytes calldata decryptionProof
) external {
    require(pendingLiquidationReveal[borrower], "No pending reveal");
    FHE.checkSignatures(handlesList, abiEncodedCleartexts, decryptionProof); // ✅ verify
    bool isLiq = abi.decode(abiEncodedCleartexts, (bool));
    delete pendingLiquidationReveal[borrower];
    if (isLiq) confirmedLiquidatable[borrower] = true;
    emit LiquidationAlertPublic(borrower, isLiq, block.timestamp);
}

// Step 3 — liquidator executes (requires confirmedLiquidatable gate)
function liquidate(address borrower) external onlyRole(LIQUIDATOR_ROLE) nonReentrant {
    require(confirmedLiquidatable[borrower], "Not confirmed liquidatable");
    // ... clear position, transfer collateral to liquidator
}
```

## Production Pattern 3: 2-step position close (borrower self-close)
User requests decryption of their own debt to prove it's zero before returning collateral:

```solidity
mapping(address => bool) private pendingClose;

// Step 1 — borrower requests close (marks debt for public decryption)
function requestClosePosition() external {
    Position storage pos = positions[msg.sender];
    require(pos.active, "No active position");
    require(!pendingClose[msg.sender], "Close already pending");
    FHE.makePubliclyDecryptable(pos.totalDebt);
    pendingClose[msg.sender] = true;
    emit CloseRequested(msg.sender, FHE.toBytes32(pos.totalDebt), block.timestamp);
}

// Step 2 — relayer submits decryption; if debt == 0, collateral returned
function verifyAndClose(
    bytes32[] calldata handlesList,
    bytes calldata abiEncodedCleartexts,
    bytes calldata decryptionProof
) external nonReentrant {
    require(pendingClose[msg.sender], "No pending close");
    FHE.checkSignatures(handlesList, abiEncodedCleartexts, decryptionProof);
    uint64 debtPlaintext = abi.decode(abiEncodedCleartexts, (uint64));
    require(debtPlaintext == 0, "Outstanding debt");

    uint256 ethToReturn = ethDeposited[msg.sender];
    positions[msg.sender].active = false;
    delete ethDeposited[msg.sender];
    delete pendingClose[msg.sender];
    // return token collateral
    _returnTokens(msg.sender, msg.sender);
    if (ethToReturn > 0) {
        (bool ok,) = msg.sender.call{value: ethToReturn}("");
        require(ok, "ETH transfer failed");
    }
}
// Key: verifyAndClose decodes as uint64 (debt type) not bool — match ABI exactly
```
