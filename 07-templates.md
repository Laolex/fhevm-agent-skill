---
name: fhevm-templates
description: fhEVM complete contract template (ConfidentialVault), ERC-7984 confidential token wrapper, architecture decision guide
type: reference
---

# fhEVM — Contract Templates & Architecture

## Complete contract template — ConfidentialVault
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

import {FHE, euint64, ebool, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";
import {SepoliaConfig} from "@fhevm/solidity/config/ZamaConfig.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract ConfidentialVault is SepoliaConfig, AccessControl, ReentrancyGuard {

    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");

    mapping(address => euint64) private balances;
    mapping(address => bool)    private hasBalance;

    // ⚠️ Never emit encrypted amounts in events — leaks handle
    event Deposited(address indexed user, uint256 timestamp);
    event Withdrawn(address indexed user, uint256 timestamp);

    error NoBalance(address user);

    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(ADMIN_ROLE, msg.sender);
    }

    function deposit(
        bytes32 inputHandle,
        bytes calldata inputProof
    ) external nonReentrant {
        euint64 amount = FHE.fromExternal(
            externalEuint64.wrap(inputHandle),
            inputProof
        );
        FHE.allowThis(amount);
        FHE.allow(amount, msg.sender);

        if (hasBalance[msg.sender]) {
            euint64 newBalance = FHE.add(balances[msg.sender], amount);
            FHE.allowThis(newBalance);
            FHE.allow(newBalance, msg.sender);
            balances[msg.sender] = newBalance;
        } else {
            balances[msg.sender] = amount;
            hasBalance[msg.sender] = true;
        }

        emit Deposited(msg.sender, block.timestamp);
    }

    function getBalance(address user) external view returns (euint64) {
        if (!hasBalance[user]) revert NoBalance(user);
        return balances[user];
    }

    function isActive(address user) external view returns (bool) {
        return hasBalance[user];
    }
}
```

---

## ERC-7984 Confidential Token Standard

ERC-7984 is the confidential token standard for FHEVM. Balances are stored as `euint64` handles — nobody can see token balances on-chain. Transfers are private. This is what cUSDT (the hackathon reward token) is built on.

### Install OpenZeppelin Confidential Contracts
```bash
npm install @openzeppelin/confidential-contracts
```

### ERC-7984 from scratch (custom confidential token)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, euint64, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";
import {SepoliaZamaFHEVMConfig} from "./ZamaConfig.sol";
import {ERC7984} from "@openzeppelin/confidential-contracts/token/ERC7984/ERC7984.sol";

contract MyConfidentialToken is SepoliaZamaFHEVMConfig, ERC7984 {
    constructor() ERC7984("My Confidential Token", "cMYT", "") {
        // Mint initial supply to deployer — encrypted
        euint64 supply = FHE.asEuint64(1_000_000 * 1e6); // 1M tokens (6 decimals)
        FHE.allowThis(supply);
        FHE.allow(supply, msg.sender);
        _unsafeMint(msg.sender, supply); // internal ERC7984 mint
    }
}
```

### ERC-7984 wrapping an existing ERC-20 (cUSDC pattern)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";
import {SepoliaZamaFHEVMConfig} from "./ZamaConfig.sol";
import {ERC7984} from "@openzeppelin/confidential-contracts/token/ERC7984/ERC7984.sol";
import {ERC7984ERC20Wrapper} from "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984ERC20Wrapper.sol";

contract ConfidentialUSDC is SepoliaZamaFHEVMConfig, ERC7984ERC20Wrapper {
    constructor(IERC20 underlying)
        ERC7984("Confidential USDC", "cUSDC", "")
        ERC7984ERC20Wrapper(underlying)
    {}

    function decimals() public view override(ERC7984, ERC7984ERC20Wrapper) returns (uint8) {
        return ERC7984ERC20Wrapper.decimals();
    }
}
// User calls wrap(amount) → deposits USDC, receives cUSDC with encrypted balance
// User calls unwrap(encryptedHandle, proof) → redeems cUSDC, receives USDC
```

### ERC-7984 key interface
```solidity
// Encrypted transfer — amount stays private
function confidentialTransfer(
    address to,
    bytes32 encryptedAmountHandle,
    bytes calldata inputProof
) external returns (bool);

// Read your own encrypted balance (returns euint64 handle)
function encryptedBalanceOf(address account) external view returns (euint64);

// Approve for encrypted allowances (private DeFi composability)
function confidentialApprove(
    address spender,
    bytes32 encryptedAmountHandle,
    bytes calldata inputProof
) external returns (bool);
```

### Frontend: wrapping ERC-20 → ERC-7984
```typescript
// 1. Approve the wrapper contract to spend USDC
const usdc = new Contract(USDC_ADDRESS, ERC20_ABI, signer);
await (await usdc.approve(CONFIDENTIAL_USDC_ADDRESS, rawAmount)).wait();

// 2. Wrap — deposits USDC, balance becomes encrypted on-chain
const wrapper = new Contract(CONFIDENTIAL_USDC_ADDRESS, cUSDC_ABI, signer);
await wrapper.wrap(rawAmount);

// 3. Confidential transfer — amount hidden
const encrypted = await fhevmInst.encryptUint({
    value: BigInt(transferAmount),
    type: "euint64",
    contractAddress: CONFIDENTIAL_USDC_ADDRESS,
    callerAddress: userAddress,
});
await wrapper.confidentialTransfer(
    recipientAddress,
    ethers.hexlify(encrypted.handles[0]),
    ethers.hexlify(encrypted.inputProof)
);

// 4. Decrypt your own balance (SDK v0.4.1)
const balanceHandle = await wrapper.encryptedBalanceOf(userAddress);
const { publicKey, privateKey } = fhevmInst.generateKeypair();
const now = Math.floor(Date.now() / 1000);
const eip712 = fhevmInst.createEIP712(publicKey, [CONFIDENTIAL_USDC_ADDRESS], now, 1);
const typeName = eip712.types.Reencrypt
    ? "Reencrypt"
    : Object.keys(eip712.types).find((k: string) => k !== "EIP712Domain")!;
const sig = await signer.signTypedData(eip712.domain, { [typeName]: eip712.types[typeName] }, eip712.message);
const results = await fhevmInst.userDecrypt(
    [{ handle: balanceHandle, contractAddress: CONFIDENTIAL_USDC_ADDRESS }],
    privateKey, publicKey, sig, [CONFIDENTIAL_USDC_ADDRESS], userAddress, now, 1,
);
const balance = Object.values(results)[0] as bigint;
```

### OZ Confidential Contracts extensions available
```
@openzeppelin/confidential-contracts/token/ERC7984/ERC7984.sol          — base token
@openzeppelin/confidential-contracts/token/ERC7984/extensions/
  ERC7984ERC20Wrapper.sol    — wrap plaintext ERC-20 into confidential version
  ERC7984Burnable.sol        — encrypted burn
  ERC7984Mintable.sol        — role-gated encrypted mint
  ERC7984Pausable.sol        — emergency pause
  ERC7984Votes.sol           — confidential governance votes
```

Note: Hackathon rewards (cUSDT) are paid in ERC-7984. Building ERC-7984-compatible contracts that accept/produce cUSDT directly demonstrates protocol fluency.

---

## Architecture decision guide

| Need | Pattern |
|------|---------|
| User sees only their own data | User decrypt (`fhevmInstance.userDecrypt`) |
| Public result after deadline | FHE oracle (`FHE.requestDecryption` + callback) |
| Conditional logic without leaking branch | `FHE.select(condition, a, b)` |
| Token with private balances | ERC-7984 (`ERC7984ERC20Wrapper`) |
| Custom encrypted state | `euint64` mappings + ACL grants |
| Encrypted voting | `euint64` tallies + `FHE.select` + oracle reveal |
| Private lending | `euint64` collateral/debt + `FHE.lt` health factor + Pattern 3 liquidation |
| Multi-asset cross-collateral | ETH-wei normalization + `_applyCollateral` accumulator pattern |
| Sealed-bid auction | `euint64` bids + `FHE.max` to find winner + oracle reveal |
| Encrypted NFT attributes | `euint64` mappings keyed by `tokenId` |
| Private credit score | `euint64` score + tiered `FHE.select` to gate loan terms without revealing |
| Permissioned liquidation | 3-step: requestReveal → verifyReveal → liquidate (confirmedLiquidatable gate) |
| User-initiated position close | 2-step: requestClose → verifyAndClose (debt == 0 check via decryption) |

---

---

## Reference contract: ConfidentialLending (ShieldLend — production, Sepolia)
Full overcollateralized lending protocol with multi-asset cross-collateral, credit score integration, and 3-step liquidation.

**Key patterns:**
- `ZamaEthereumConfig` (local copy) + `AccessControlEnumerable` + `ReentrancyGuard` + `Pausable`
- `SafeERC20` for all ERC20 transfers
- `_applyCollateral()` — init-or-add encrypted accumulator (see 03-input-acl.md)
- `_computeHealthFactor()` — score-gated tiered `FHE.select` ratio
- `_allowLiquidators()` — iterates `getRoleMemberCount` to grant ACL to all liquidators
- `_returnTokens()` — multi-asset cleanup on liquidation/close
- Events only emit plaintext metadata (timestamps, addresses) — never encrypted handles

**Deployed:** Sepolia `0xFC0f1744d3cF752Bdd67c0BA4b0CaD4048f7376A` (ConfidentialLending)
**Companion:** `ConfidentialCreditScore` — stores `euint64` score, exposes `hasScore()` + `meetsThreshold()`
**Source:** `/home/laolex/Projects/shieldlend/contracts/`

---

## Deployment checklist
```bash
# 1. Compile
npx hardhat compile

# 2. Test (local mock fhEVM)
npx hardhat test

# 3. Deploy to Sepolia
npx hardhat deploy --network sepolia

# 4. Generate std_input.json for Etherscan verification
node -e "
const fs = require('fs');
const bi = JSON.parse(fs.readFileSync('artifacts/build-info/<hash>.json'));
fs.writeFileSync('std_input.json', JSON.stringify(bi.input, null, 2));
"
# Upload to https://sepolia.etherscan.io/verifyContract
# Settings: Solidity (Standard-Json-Input), compiler 0.8.27, 200 runs, EVM cancun
```
