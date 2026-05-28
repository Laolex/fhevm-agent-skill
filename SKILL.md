---
name: fhevm
description: >
  Production-ready agent skill for FHEVM confidential smart contracts (Zama Protocol).
  Full development workflow: architecture, setup, contracts, testing, deployment, frontend.
  Covers: encrypted types (euint8/16/32/64/128, ebool, eaddress), all FHE ops,
  ACL (allowThis/allow/allowTransient/allowPublic), input proofs, re-encryption,
  public decryption, oracle callbacks, ERC-7984 confidential tokens,
  OpenZeppelin Confidential Contracts, @fhevm/solidity v0.11+.
  Frontend SDKs: @zama-fhe/relayer-sdk v0.4 (legacy primitives) AND
  @zama-fhe/sdk v3 / @zama-fhe/react-sdk v3 (new — ZamaSDK, Token API, 59 hooks).
  HCU cost table for every operation and type. relayer-sdk 0.4.1 pin conflict resolution.
  Agentic commands: /fhevm scaffold (full Vercel-deployable project from spec),
  /fhevm audit (ACL/anti-pattern review + auto-fix), /fhevm migrate (TFHE→FHE rewrite).
  Validated against Claude Code, Cursor, and Windsurf.
---

# fhEVM Agent Skill

## Compatibility matrix (canonical stack)

| Layer | Pinned version | Notes |
|---|---|---|
| `@fhevm/solidity` | `^0.11.1` | `FHE.*` API. **No** `FHE.requestDecryption`, **no** `FHE.gte` — use Pattern 3 + `FHE.not(FHE.lt(...))` |
| `@fhevm/hardhat-plugin` | `^0.4.2` | mock coprocessor for tests, `fhevm.initializeCLIApi()` |
| `@fhevm/mock-utils` | `^0.4.2` | peer-pins `@zama-fhe/relayer-sdk` at **exactly** `0.4.1` |
| `@zama-fhe/relayer-sdk` | `0.4.1` exact | legacy primitives — import from `/web` (Vite/Next.js). Pin `0.4.1` via `overrides` if also using new SDK |
| `@zama-fhe/sdk` | `^3.0.0` | NEW — `ZamaSDK`, `Token`, sessions. Requires **Node >= 22** |
| `@zama-fhe/react-sdk` | `^3.0.0` | NEW — `ZamaProvider`, 59 TanStack Query hooks |
| Solidity | `^0.8.28` (new default) / `0.8.24` (min proven) | `evmVersion: "cancun"` |
| Network | Sepolia (chainId 11155111) | `SepoliaConfig` from npm — do NOT vendor a local `ZamaConfig.sol` |

**Which SDK to use:**
- Custom contracts, raw encrypted inputs, legacy apps → `@zama-fhe/relayer-sdk@0.4.x` (see `05-frontend.md`)
- Confidential ERC-7984 dApps, React frontends → `@zama-fhe/sdk@^3` / `@zama-fhe/react-sdk@^3` (see `09-new-sdk.md`)
- Hardhat tests → always `@fhevm/hardhat-plugin` + `@zama-fhe/relayer-sdk@0.4.1`

**Supported APIs (legacy relayer-sdk):**
- Encryption: `createEncryptedInput(contract, caller).addNN(value).encrypt()` builder
- Decryption: `userDecrypt` (Pattern 1, batch `HandleContractPair`), `makePubliclyDecryptable` + `checkSignatures(handlesList, ...)` with handle pinning (Pattern 3)
- ACL: `allowThis`, `allow`, `allowTransient`, `allowPublic`, `makePubliclyDecryptable`
- Frontend init: `initSDK()` then `createInstance({ ...SepoliaConfig, network, relayerUrl: <proxy> })`

**Banned APIs (will not compile / will not load):**
- `TFHE.*` (any), `einput`, `GatewayCaller`, `Gateway.requestDecryption`, `onlyGateway`
- `FHE.requestDecryption`, `FHE.gte` — not present in `@fhevm/solidity@0.11.x`
- `fhevmInstance.encryptUint(...)`, `fhevmInstance.encrypt32/64(...)` — removed in `relayer-sdk@0.4.1`
- `initFhevm` — removed; use `initSDK` (legacy) or `new ZamaSDK(...)` (new SDK)
- `@zama-fhe/relayer-sdk/web.js`, `@zama-fhe/relayer-sdk/bundle` (in Vite/Next.js ESM)
- `hardhat verify` auto-run on Etherscan (v2 API broken in `hardhat-verify ≤ 2.1.3`) — use manual `std_input.json` upload
- Local-copy `ZamaConfig.sol`

**AI tool compatibility:** Claude Code (native), Cursor, Windsurf, GitHub Copilot, any MCP-capable agent.

---

## DISPATCH — execute immediately on load

When this skill is invoked, check the args/command first and route accordingly:

| If args or user said… | Action |
|-----------------------|--------|
| `scaffold` / `/fhevm scaffold` / `/fhevm:scaffold` / "generate a contract" / "build a new fhevm contract" | Read `scaffold.md` from this skill's directory, then execute that workflow |
| `audit` / `/fhevm audit` / `/fhevm:audit` / "audit this contract" / "review for ACL issues" | Read `audit.md` from this skill's directory, then execute that workflow |
| `migrate` / `/fhevm migrate` / `/fhevm:migrate` / "migrate from TFHE" / "update to new FHE API" | Read `migrate.md` from this skill's directory, then execute that workflow |
| No args / help / "what can you do" | Show the command menu below and wait for user |
| Any other fhEVM task | Read relevant reference module(s) below, then proceed |

**Do not wait for the user to repeat themselves. Read the file and start executing immediately.**

---

## Commands

| Command | What it does |
|---------|-------------|
| `/fhevm scaffold` | Generate full contract + tests + deploy script + frontend from plain-language spec |
| `/fhevm audit` | Audit for ACL gaps, anti-patterns, CORS issues — numbered report + optional auto-fix |
| `/fhevm migrate` | Rewrite TFHE.*/GatewayCaller code to current FHE.* API in one pass |

---

## Reference Modules

| File | Contents |
|------|----------|
| [00-architecture.md](00-architecture.md) | FHEVM architecture, coprocessor model, ciphertext handles, ACL overview, decryption patterns, Sepolia addresses |
| [01-setup.md](01-setup.md) | Install, hardhat.config.ts with vars(), multi-contract deploy, ZamaConfig patterns, relayer-sdk conflict fix |
| [02-types-ops.md](02-types-ops.md) | euint8/16/32/64/128, ebool, eaddress — all FHE ops, tiered select, interest math, gas costs |
| [03-input-acl.md](03-input-acl.md) | FHE.fromExternal proofs, FHE.allowTransient, _applyCollateral pattern, multi-role ACL |
| [04-decryption.md](04-decryption.md) | Pattern 1 (userDecrypt v0.4.1, batch HandleContractPair, multi-contract), Pattern 3 (makePubliclyDecryptable + signature verification) |
| [05-frontend.md](05-frontend.md) | Legacy SDK: initSDK + SepoliaConfig, userDecrypt, ethers v6 provider fix, UI phase state machine, CORS proxy |
| [06-testing.md](06-testing.md) | initializeCLIApi, createEncryptedInput, userDecrypt (v0.4.1), publicDecryptEbool, verifyReveal flow |
| [07-templates.md](07-templates.md) | ConfidentialVault, ERC-7984 full standard + OZ Confidential Contracts, architecture decision table |
| [08-anti-patterns.md](08-anti-patterns.md) | TFHE, missing ACL, ebool if/else, missing initSDK, old reencrypt API (v0.4.1), ethers v6 provider stale, vercel.json rewrite, Etherscan v2, new SDK anti-patterns |
| [09-new-sdk.md](09-new-sdk.md) | **NEW** @zama-fhe/sdk v3 + @zama-fhe/react-sdk v3: ZamaSDK, Token API (shield/unshield/transfer/balance), ZamaProvider, 59 hooks, session/keypair TTL, error taxonomy, relayer-sdk 0.4.1 conflict resolution |
| [10-hcu-costs.md](10-hcu-costs.md) | **NEW** HCU cost table — every operation for ebool/euint8/16/32/64/128/256, scalar vs non-scalar, budget patterns, optimization tips, diagnosing HCU reverts |

---

## Quick load guide

| Task | Action |
|------|--------|
| New contract from spec | `/fhevm scaffold` |
| Audit existing code | `/fhevm audit` |
| Migrate from old TFHE API | `/fhevm migrate` |
| Manual contract work | Read 01 → 02 → 03 → 04 → 08 |
| Frontend FHE init / relayer issues (legacy SDK) | Read 05 |
| New SDK (`@zama-fhe/sdk@^3` / React hooks) | Read 09 |
| HCU budget exceeded / cost estimate | Read 10 |
| relayer-sdk version conflict | Read 09 (conflict section) |
| Tests only | Read 06 |
| Debugging compile errors | Read 08 → 01 |

---

## FHE Frontend Init — Critical Pattern (inline for fast debugging)

The most common "FHE unavailable" cause: proxy forwards wrong content-type, or double-init race.

**Correct proxy (`api/zama-relay.js`):**
```javascript
const contentType = response.headers.get('content-type') ?? 'application/octet-stream';
const data = contentType.includes('application/octet-stream') || contentType.includes('binary')
  ? await response.arrayBuffer()
  : await response.text();
return new Response(data, { status: response.status, headers: { 'content-type': contentType, ... } });
```

**Correct createInstance with fallback:**
```typescript
try {
  await initSDK();
  inst = await Promise.race([
    createInstance({ ...SepoliaConfig, network: eth, relayerUrl: `${window.location.origin}/api/zama-relay` }),
    new Promise((_, rej) => setTimeout(() => rej(new Error("timeout")), 60000))
  ]);
} catch {
  // Fallback: SepoliaConfig has its own relayerUrl
  inst = await createInstance({ ...SepoliaConfig, network: eth });
}
```

---

## Production reference

**ShieldLend** — overcollateralized lending, Sepolia
- Contracts: `ConfidentialLending.sol` + `ConfidentialCreditScore.sol` (447 + 295 lines)
- Patterns: multi-asset cross-collateral, score-gated tiered ratios, 3-step liquidation, 2-step close
- Source: [github.com/Laolex/shieldlend](https://github.com/Laolex/shieldlend)

**ShieldPay v2** — FHE payroll, Sepolia
- Contract: `ConfidentialPayroll.sol` — EMPLOYER/AUDITOR/PAYMASTER roles, FHE payroll math
- Source: [github.com/Laolex/shieldpay](https://github.com/Laolex/shieldpay)

---

## Version
v3.5.0 — Added `09-new-sdk.md` (@zama-fhe/sdk v3 / @zama-fhe/react-sdk v3: ZamaSDK, Token API, 59 React hooks, ZamaProvider, session TTL, new error taxonomy, relayer-sdk 0.4.1 pin conflict resolution) and `10-hcu-costs.md` (full HCU cost table for all types and operations, budget planning, optimization tips). Updated compatibility matrix: @fhevm/hardhat-plugin and @fhevm/mock-utils pinned to 0.4.2, relayer-sdk at 0.4.1 exact, new SDK packages documented. Pragma updated — ^0.8.28 is now the default; 0.8.24 remains the minimum. Quick load guide updated with new SDK and HCU routing.

v3.4.0 — **Security findings from ShieldLend pre-submission audit (2026-04-17):**
(1) **Handle-pinning** — every Pattern-3 callback receiving `bytes32[] handlesList` MUST `require(handlesList[i] == FHE.toBytes32(expected))` before `FHE.checkSignatures`; without it, attackers substitute a valid-signed pair from elsewhere to forge outcomes. New audit check A7 + anti-pattern.
(2) **Input-equivalent clamping** — any encrypted value the user submits that represents a plaintext quantity (token→ETH, USD, credit) must be clamped via `FHE.select(lt(cap, enc), cap, enc)` against an oracle-derived plaintext cap; unclamped, the collateral model is fictional. New audit check A8 + anti-pattern.
(3) **Predicate binary-search leak** — `meetsThreshold`/`isEligible`-style functions must be role-gated (`msg.sender == subject || hasRole(READER_ROLE, msg.sender)`); otherwise any address binary-searches the encrypted value in O(log n) calls. New audit check A9 + anti-pattern.

v3.3.0 — Proxy header forwarding: hardcoding `content-type: application/json` drops `ZAMA-SDK-VERSION`/`ZAMA-SDK-NAME` → FHE offline; fix: forward original headers. Duplicate env var declarations without `.trim()` cause ENS `{0A}` error; fix: declare once in `config.ts` with `.trim()`, import everywhere. Path extraction anchored to `^\/api\/zama-relay` with `|| '/'` fallback.
