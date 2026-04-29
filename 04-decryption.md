---
name: fhevm-decryption
description: fhEVM decryption patterns — userDecrypt (SDK v0.4.1, batch HandleContractPair), FHE.makePubliclyDecryptable relayer verification pattern
type: reference
---

# fhEVM — Decryption Patterns

Two production patterns. Choose based on who needs the value and when.

|                                 | Pattern 1                        | Pattern 3                                     |
| ------------------------------- | -------------------------------- | --------------------------------------------- |
| **API**                         | `fhevmInstance.userDecrypt`      | `FHE.makePubliclyDecryptable`                 |
| **Who sees it**                 | Only the authorized user         | Publicly — via relayer                        |
| **Contract receives plaintext** | No                               | Yes — when verify function decodes cleartexts |
| **Use when**                    | User decrypts their own position | Public reveal / settlement flows              |

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
import { createInstance } from "@zama-fhe/relayer-sdk/web";

const encHandle = await contract.getEncryptedSalary(userAddress);
const { publicKey, privateKey } = fhevmInstance.generateKeypair();

// ✅ SDK v0.4.x: createEIP712(pubKey, contractAddresses[], startTimestamp, durationDays)
const now = Math.floor(Date.now() / 1000);
const eip712 = fhevmInstance.createEIP712(
  publicKey,
  [CONTRACT_ADDRESS],
  now,
  1,
);

// ✅ v0.4.x exposes eip712.primaryType ("UserDecryptRequestVerification").
//    Do not guess with `types.Reencrypt ? ... : Object.keys(...)` — that silently signs
//    a different struct on SDK upgrades and the KMS rejects with a cryptic error.
const { primaryType } = eip712;
const signature = await signer.signTypedData(
  eip712.domain,
  { [primaryType]: eip712.types[primaryType] },
  eip712.message,
);

// ✅ SDK v0.4.1: userDecrypt(HandleContractPair[], privKey, pubKey, sig, contractAddresses[], userAddr, startTs, days)
const handles = [{ handle: encHandle, contractAddress: CONTRACT_ADDRESS }];
const results = await fhevmInstance.userDecrypt(
  handles,
  privateKey,
  publicKey,
  signature,
  [CONTRACT_ADDRESS],
  userAddress,
  now,
  1,
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
const eip712 = fhevmInstance.createEIP712(
  publicKey,
  [CONTRACT_ADDRESS],
  now,
  1,
);
const { primaryType } = eip712;
const sig = await signer.signTypedData(
  eip712.domain,
  { [primaryType]: eip712.types[primaryType] },
  eip712.message,
);

// ✅ All handles in one call — single network round-trip
const handles = [collHandle, debtHandle, rateHandle, scoreHandle].map(
  (h: string) => ({ handle: h, contractAddress: CONTRACT_ADDRESS }),
);
const results = await fhevmInstance.userDecrypt(
  handles,
  privateKey,
  publicKey,
  sig,
  [CONTRACT_ADDRESS],
  userAddress,
  now,
  1,
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
  publicKey,
  [LENDING_ADDRESS, SCORE_ADDRESS],
  now,
  1,
);
// ... sign and call userDecrypt with [LENDING_ADDRESS, SCORE_ADDRESS]
```

---

## Pattern 2 — Deprecated in this stack

`FHE.requestDecryption(...)` is not available in the currently pinned stack (`@fhevm/solidity ^0.11.1` + `@fhevm/hardhat-plugin ^0.4.x`).
Use Pattern 3 with `FHE.makePubliclyDecryptable` + `FHE.checkSignatures(handlesList, ...)` and handle pinning.

```solidity
function requestTallyReveal(uint256 proposalId) external {
    Proposal storage p = proposals[proposalId];
    FHE.makePubliclyDecryptable(p.encForVotes);
    FHE.makePubliclyDecryptable(p.encAgainstVotes);
    FHE.makePubliclyDecryptable(p.encAbstainVotes);
    p.revealRequested = true;
}

function verifyTallyReveal(
    uint256 proposalId,
    bytes32[] calldata handlesList,
    bytes calldata abiEncodedCleartexts,
    bytes calldata decryptionProof
) external {
    Proposal storage p = proposals[proposalId];
    require(handlesList.length == 3, "Expected 3 handles");
    require(handlesList[0] == FHE.toBytes32(p.encForVotes), "For handle mismatch");
    require(handlesList[1] == FHE.toBytes32(p.encAgainstVotes), "Against handle mismatch");
    require(handlesList[2] == FHE.toBytes32(p.encAbstainVotes), "Abstain handle mismatch");

    FHE.checkSignatures(handlesList, abiEncodedCleartexts, decryptionProof);
    (uint64 forVotes, uint64 againstVotes, uint64 abstainVotes) = abi.decode(
        abiEncodedCleartexts,
        (uint64, uint64, uint64)
    );
    // ... settle
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

    // 🔒 CRITICAL: pin handlesList to the expected ciphertext BEFORE checkSignatures.
    //    checkSignatures only attests "KMS signed these handles decrypt to these
    //    cleartexts" — it does NOT bind the handle to any caller-supplied slot.
    //    Without this require, any caller can substitute a publicly-decryptable
    //    euint64(0) (or any forged pair) from elsewhere and the verify would pass.
    require(handlesList.length == 1, "Expected 1 handle");
    require(
        handlesList[0] == FHE.toBytes32(positions[subject].isLiquidatable),
        "Handle mismatch"
    );

    FHE.checkSignatures(handlesList, abiEncodedCleartexts, decryptionProof); // ✅ array form
    bool isLiquidatable = abi.decode(abiEncodedCleartexts, (bool));
    delete pendingReveal[subject];
    emit RevealResult(subject, isLiquidatable, block.timestamp);
}

// ✅ Pattern 3 checkSignatures signature (array form — different from Pattern 2):
// FHE.checkSignatures(bytes32[] handlesList, bytes abiEncodedCleartexts, bytes decryptionProof)
//
// 🔒 Pattern 3 is caller-controlled: always precede checkSignatures with a
//    `require(handlesList[i] == FHE.toBytes32(expected_i))` for every handle.
//    Pattern 2 (requestDecryption + requestId) is coprocessor-bound — no pin needed.

// No requestId-based checkSignatures overload in this stack.
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
    require(handlesList.length == 1, "Expected 1 handle");
    require(
        handlesList[0] == FHE.toBytes32(positions[borrower].isLiquidatable),
        "Handle mismatch"   // 🔒 block handle-substitution attack
    );
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
    require(handlesList.length == 1, "Expected 1 handle");
    require(
        handlesList[0] == FHE.toBytes32(positions[msg.sender].totalDebt),
        "Handle mismatch"   // 🔒 prevent caller substituting any euint64(0) handle
    );
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
