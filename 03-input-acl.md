---
name: fhevm-input-acl
description: fhEVM input proofs (FHE.fromExternal, externalEuint64), ACL grants (FHE.allowThis/allow/allowPublic) — most commonly missed patterns
type: reference
---

# fhEVM — Input Proofs & ACL

## Input proofs — receiving encrypted values from users

```solidity
// ✅ Correct pattern
function deposit(
    bytes32 inputHandle,      // encrypted value handle from frontend encryptUint()
    bytes calldata inputProof // ZK proof that value is valid
) external payable {
    euint64 encAmount = FHE.fromExternal(
        externalEuint64.wrap(inputHandle),
        inputProof
    );
    FHE.allowThis(encAmount);           // grant contract permission immediately
    FHE.allow(encAmount, msg.sender);   // grant sender re-encrypt permission
    positions[msg.sender] = encAmount;
}

// ❌ Wrong — skipping proof validation
function deposit(bytes32 handle) external {
    euint64 amount = euint64.wrap(handle); // NEVER — no validation, security hole
}
```

## Available external wrapper types
```solidity
externalEuint8   // wrap(bytes32)
externalEuint16  // wrap(bytes32)
externalEuint32  // wrap(bytes32)
externalEuint64  // wrap(bytes32)
externalEbool    // wrap(bytes32)
externalEaddress // wrap(bytes32)
```

## Multi-value input proof (single proof covers all handles from one encrypt() call)
```solidity
// ✅ Two encrypted values, one proof
function vote(
    bytes32 weightHandle,   // euint64
    bytes32 voteHandle,     // euint8 — encrypted vote type
    bytes calldata inputProof
) external {
    euint64 weight   = FHE.fromExternal(externalEuint64.wrap(weightHandle), inputProof);
    euint8  voteType = FHE.fromExternal(externalEuint8.wrap(voteHandle), inputProof);

    FHE.allowThis(weight);
    FHE.allowThis(voteType);

    ebool isFor     = FHE.eq(voteType, FHE.asEuint8(1));
    ebool isAgainst = FHE.eq(voteType, FHE.asEuint8(0));
    euint64 zero    = FHE.asEuint64(0);

    forVotes     = FHE.add(forVotes,     FHE.select(isFor,     weight, zero));
    againstVotes = FHE.add(againstVotes, FHE.select(isAgainst, weight, zero));

    FHE.allowThis(forVotes);
    FHE.allowThis(againstVotes);
}

// Frontend: input.add64(weight); input.add8(voteType); — two values, one proof
```

---

## ACL grants — most commonly missed

fhEVM uses an ACL to control who can use an encrypted value. **Every encrypted value must be explicitly allowed for every contract/address that needs to use it.**

```solidity
FHE.allowThis(encryptedValue);       // grant this contract permission
FHE.allow(encryptedValue, addr);     // grant a specific address (user re-encrypt)
FHE.allowPublic(encryptedValue);     // anyone can compute on it — use carefully
```

### Typical pattern — allow contract + owner + employer
```solidity
function setSalary(address employee, bytes32 handle, bytes calldata proof) external {
    euint64 salary = FHE.fromExternal(externalEuint64.wrap(handle), proof);
    FHE.allowThis(salary);           // contract can use in future operations
    FHE.allow(salary, employee);     // employee can re-encrypt/decrypt
    FHE.allow(salary, msg.sender);   // employer can re-encrypt/decrypt
    salaries[employee] = salary;
}
```

### ACL for computed values
```solidity
// ✅ Computed values need fresh ACL grants — they are new handles
euint64 newDebt = FHE.add(existingDebt, newLoan);
FHE.allowThis(newDebt);          // ✅ required — new handle, no ACL by default
FHE.allow(newDebt, borrower);    // ✅ if borrower needs to decrypt it
existingDebt = newDebt;

// ❌ Storing without allowThis — contract can't operate on it in next call
euint64 val = FHE.fromExternal(externalEuint64.wrap(handle), proof);
storage[user] = val;  // ❌ stored but contract can't use it later
```

### FHE.allowTransient — single-use temporary permission

Use when a value needs to cross contract boundaries within ONE transaction but should not persist:

```solidity
// Caller grants transient permission before calling external contract
function liquidateViaHelper(address borrower, address helperContract) external {
    FHE.allowTransient(positions[borrower].collateral, helperContract);
    ILiquidationHelper(helperContract).execute(borrower);
    // After tx: permission is automatically revoked — helperContract cannot use it again
}

// ✅ Use for: cross-contract calls within same tx, composability patterns
// ❌ Do NOT use for: values the contract needs in future txs (use allowThis instead)
```

Difference summary:
- `FHE.allow(val, addr)` — **persistent** permission, survives across transactions
- `FHE.allowTransient(val, addr)` — **ephemeral** permission, cleared when tx ends
- `FHE.allowThis(val)` — persistent permission for the current contract

### Common ACL checklist
- [ ] `FHE.allowThis` immediately after every `FHE.fromExternal`
- [ ] `FHE.allow(val, user)` for every user who needs to re-encrypt
- [ ] `FHE.allowThis` after every computed value (`FHE.add`, `FHE.sub`, `FHE.select`, etc.)
- [ ] Use `FHE.allowTransient` for cross-contract calls within same tx
- [ ] Never `allowPublic` unless you specifically want world-readable computation

---

## Init-or-add pattern (_applyCollateral)
When a user may deposit multiple times (ETH, USDC, ZAMA) and you want one encrypted accumulator:

```solidity
/// @dev First deposit initialises the position; subsequent deposits add to collateral.
function _applyCollateral(address user, euint64 encAmount) internal {
    Position storage pos = positions[user];
    if (!pos.active) {
        // First deposit — initialise all fields
        pos.collateral     = encAmount;
        pos.loanAmount     = FHE.asEuint64(0);
        pos.totalDebt      = FHE.asEuint64(0);
        pos.creditScore    = FHE.asEuint64(500);  // default score
        pos.interestRate   = FHE.asEuint64(BASE_RATE_BPS);
        pos.isLiquidatable = FHE.asEbool(false);
        pos.lastAccrual    = block.timestamp;
        pos.active         = true;
        borrowerIndex[user] = borrowerList.length;
        borrowerList.push(user);
        // ✅ allowThis for all fields the contract will compute on
        FHE.allowThis(pos.loanAmount);
        FHE.allowThis(pos.totalDebt);
        FHE.allowThis(pos.creditScore);
        FHE.allowThis(pos.interestRate);
        FHE.allowThis(pos.isLiquidatable);
    } else {
        // Subsequent deposit — add to existing collateral
        euint64 newCollateral = FHE.add(pos.collateral, encAmount);
        FHE.allowThis(newCollateral);
        FHE.allow(newCollateral, user);
        pos.collateral = newCollateral;
    }
    // Grant user access to collateral handle (same path for both branches)
    FHE.allow(pos.collateral, user);
    FHE.allowThis(pos.collateral);
}
```

## Multi-role ACL grant (AccessControlEnumerable pattern)
When multiple liquidators need access to an encrypted flag:

```solidity
// ✅ Requires AccessControlEnumerable (not plain AccessControl)
function _allowLiquidators(ebool ct) internal {
    uint256 n = getRoleMemberCount(LIQUIDATOR_ROLE);
    for (uint256 i = 0; i < n; i++) {
        FHE.allow(ct, getRoleMember(LIQUIDATOR_ROLE, i));
    }
}
// Call after borrow/repay whenever isLiquidatable is recomputed
```

## ERC20 cross-collateral input (approve → depositToken)
Frontend must approve before calling depositToken. The inputHandle carries the ETH-wei equivalent:

```solidity
// Contract — depositToken
function depositToken(
    address token,
    uint256 tokenAmount,        // plaintext raw token units (for safeTransferFrom)
    bytes32 inputHandle,        // encrypted ETH-wei equivalent
    bytes calldata inputProof
) external nonReentrant whenNotPaused {
    TokenConfig memory cfg = tokenConfigs[token];
    require(cfg.supported, "Token not supported");
    IERC20(token).safeTransferFrom(msg.sender, address(this), tokenAmount);
    tokenDeposited[msg.sender][token] += tokenAmount;
    euint64 encEquiv = FHE.fromExternal(externalEuint64.wrap(inputHandle), inputProof);
    _applyCollateral(msg.sender, encEquiv);
}
```

```typescript
// Frontend — ERC20 approve + depositToken flow
const token = TOKENS.find(t => t.symbol === selectedToken);
const rawAmount = parseUnits(amount, token.decimals);      // token raw units
// ETH-wei equivalent: rawAmount * ethWeiPerToken / 10^decimals
const ethWeiEquiv = (rawAmount * token.ethWeiPerToken) / (10n ** BigInt(token.decimals));

// 1. Approve
const erc20 = new Contract(token.address, ERC20_ABI, signer);
await (await erc20.approve(CONTRACT_ADDRESS, rawAmount)).wait();

// 2. Encrypt ETH-wei equivalent
const encrypted = await fhevmInstance.encryptUint({
    value: ethWeiEquiv,
    type: "euint64",
    contractAddress: CONTRACT_ADDRESS,
    callerAddress: userAddress,
});
const handle = ethers.hexlify(encrypted.handles[0]);
const proof  = ethers.hexlify(encrypted.inputProof);

// 3. Deposit
await contract.depositToken(token.address, rawAmount, handle, proof);
```

**Privacy note:** the ETH-wei equivalent is derivable from `rawAmount × knownPrice`, so it is pseudo-private. Production would use an encrypted price oracle.
