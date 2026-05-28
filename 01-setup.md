---
name: fhevm-setup
description: fhEVM project setup — install, hardhat config, contract imports, inheritance patterns, Sepolia network addresses
type: reference
---

# fhEVM Setup & Imports

## Install dependencies
```bash
# Hardhat project (contracts + tests)
npm install @fhevm/solidity @fhevm/hardhat-plugin @fhevm/mock-utils
npm install --save-dev hardhat @nomicfoundation/hardhat-ethers @nomicfoundation/hardhat-chai-matchers
# Toolchain requires @zama-fhe/relayer-sdk at EXACTLY 0.4.1 — pin via overrides if needed
npm install @zama-fhe/relayer-sdk@0.4.1

# Frontend — legacy SDK (custom contracts, raw encrypted inputs)
npm install @zama-fhe/relayer-sdk@0.4.1

# Frontend — new SDK (ERC-7984 dApps, React)
npm install @zama-fhe/sdk@^3.0.0                    # vanilla TS / Node.js (requires Node >= 22)
npm install @zama-fhe/react-sdk@^3.0.0@tanstack/react-query@^5  # React
```

### Dependency Conflict: `relayer-sdk` 0.4.1 Pin

If you install both the Hardhat toolchain and `@zama-fhe/sdk@^3` in the **same** `package.json`,
npm can hoist `relayer-sdk` to `0.4.2+` and Hardhat aborts:

```
Error in plugin @fhevm/hardhat-plugin: Invalid @zama-fhe/relayer-sdk version.
Expecting 0.4.1. Got 0.4.2 instead.
```

**Fix (single-package):** add to `package.json`:
```json
{
  "overrides": { "@zama-fhe/relayer-sdk": "0.4.1" }
}
```
Then `rm -rf node_modules package-lock.json && npm install`.

**Fix (monorepo, recommended):** Keep the Hardhat package and the React/dApp frontend in
separate workspace packages. The contracts workspace pins `0.4.1`; the frontend workspace
installs whatever `@zama-fhe/sdk` needs. See `09-new-sdk.md` for full details.

## hardhat.config.ts
```typescript
import "@fhevm/hardhat-plugin";
import "@nomicfoundation/hardhat-ethers";
import "@nomicfoundation/hardhat-verify";
import type { HardhatUserConfig } from "hardhat/config";

// ✅ Use hardhat vars() instead of process.env — secrets stay out of .env files
// Setup: npx hardhat vars set MNEMONIC, INFURA_API_KEY, ETHERSCAN_API_KEY
const { vars } = require("hardhat/config");

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.28",   // ✅ 0.8.28 is the new default; 0.8.24 is the minimum proven on Sepolia
    settings: {
      optimizer: { enabled: true, runs: 200 },
      evmVersion: "cancun",
    },
  },
  networks: {
    sepolia: {
      url: `https://sepolia.infura.io/v3/${vars.get("INFURA_API_KEY", "")}`,
      accounts: { mnemonic: vars.get("MNEMONIC", "") },
      chainId: 11155111,
    },
  },
  etherscan: {
    apiKey: vars.get("ETHERSCAN_API_KEY", ""),
  },
};
export default config;
```

## Hardhat vars setup (once per machine)
```bash
npx hardhat vars set MNEMONIC
npx hardhat vars set INFURA_API_KEY
npx hardhat vars set ETHERSCAN_API_KEY
# View all set vars
npx hardhat vars list
```

## Contract imports — exact paths (v0.9+)
```solidity
// ✅ CORRECT — v0.9+ API
import {FHE, euint8, euint16, euint32, euint64, euint128, euint256} from "@fhevm/solidity/lib/FHE.sol";
import {ebool} from "@fhevm/solidity/lib/FHE.sol";
import {eaddress} from "@fhevm/solidity/lib/FHE.sol";
import {externalEuint8, externalEuint16, externalEuint32, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";
import {SepoliaConfig, ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

// ❌ WRONG — old API, do not use
import "fhevm/lib/TFHE.sol";
import {TFHE} from "fhevm/lib/TFHE.sol";
import {SepoliaZamaFHEVMConfig} from "@fhevm/solidity/config/ZamaFHEVMConfig.sol";  // ❌ outdated
import {SepoliaZamaGatewayConfig} from "@fhevm/solidity/config/ZamaGatewayConfig.sol";  // ❌ outdated
import {GatewayCaller} from "@fhevm/solidity/gateway/GatewayCaller.sol";               // ❌ outdated
```

## Contract inheritance
```solidity
// ✅ Always inherit config FIRST
contract MyConfidentialContract is SepoliaConfig, AccessControl {
    // sets up FHE coprocessor addresses for Sepolia
}

// ✅ For Ethereum mainnet
contract MyContract is ZamaEthereumConfig, AccessControl {}

// ✅ No extra inheritance needed for public decryption — Pattern 3
//    (FHE.makePubliclyDecryptable + FHE.checkSignatures) is built into FHE library
contract MyVotingContract is SepoliaConfig {}

// ✅ Production lending pattern — full OpenZeppelin stack
contract ConfidentialLending is ZamaEthereumConfig, AccessControlEnumerable, ReentrancyGuard, Pausable {
    using SafeERC20 for IERC20;
    // AccessControlEnumerable (not plain AccessControl) required for _allowLiquidators pattern
    // (needs getRoleMemberCount + getRoleMember to iterate role holders)
}

// ❌ OLD PATTERN — do not use GatewayCaller or SepoliaZamaGatewayConfig
// contract MyContract is SepoliaZamaFHEVMConfig, SepoliaZamaGatewayConfig, GatewayCaller {}
```

## Multi-contract deploy script pattern (production)
```typescript
// scripts/deploy-all.ts
import { ethers } from "hardhat";

async function main() {
    const [deployer] = await ethers.getSigners();

    // 1. Deploy supporting contracts first
    const ZamaFactory  = await ethers.getContractFactory("MockZAMA");
    const zamaToken    = await ZamaFactory.deploy();
    await zamaToken.waitForDeployment();

    const ScoreFactory = await ethers.getContractFactory("ConfidentialCreditScore");
    const score        = await ScoreFactory.deploy();
    await score.waitForDeployment();

    // 2. Deploy main contract
    const LendingFactory = await ethers.getContractFactory("ConfidentialLending");
    const lending        = await LendingFactory.deploy();
    await lending.waitForDeployment();

    // 3. Wire contracts together
    await (await lending.setScoreContract(await score.getAddress())).wait();

    // 4. Register collateral tokens (price = ETH-wei value of 1 whole token)
    const ETH_WEI = 10n ** 18n;
    await (await lending.addToken(USDC_ADDR, 6,  ETH_WEI / 3000n)).wait();  // 3000 USDC/ETH
    await (await lending.addToken(await zamaToken.getAddress(), 18, ETH_WEI / 100n)).wait(); // 100 ZAMA/ETH

    console.log("ConfidentialLending:", await lending.getAddress());
    console.log("ConfidentialCreditScore:", await score.getAddress());
    console.log("MockZAMA:", await zamaToken.getAddress());
}

main().catch((e) => { console.error(e); process.exit(1); });
```

## Sepolia Zama testnet addresses
```
ACL:                        0xf0Ffdc93b7E186bC2f8CB3dAA75D86d1930A433D
KMS Verifier:               0xbE0E383937d564D7FF0BC3b46c51f0bF8d5C311A
Input Verifier:             0xBBC1fFCdc7C316aAAd72E807D9b0272BE8F84DA0
Gateway Decryption:         0x5D8BD78e2ea6bbE41f26dFe9fdaEAa349e077478
Gateway Input Verification: 0x483b9dE06E4E4C7D35CCf5837A1668487406D955
Relayer URL:                https://relayer.testnet.zama.org
Chain ID:                   11155111
Gateway Chain ID:           10901
```

## Required environment variables
```bash
PRIVATE_KEY=           # deployer wallet
INFURA_API_KEY=        # or ALCHEMY_API_KEY
ETHERSCAN_API_KEY=     # for verification
```
