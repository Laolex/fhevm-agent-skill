---
name: fhevm
description: >
  Production-ready agent skill for FHEVM confidential smart contracts (Zama Protocol).
  Full development workflow: architecture, setup, contracts, testing, deployment, frontend.
  Covers: encrypted types (euint8/16/32/64/128, ebool, eaddress), all FHE ops,
  ACL (allowThis/allow/allowTransient/allowPublic), input proofs, re-encryption,
  public decryption, oracle callbacks, ERC-7984 confidential tokens,
  OpenZeppelin Confidential Contracts, @fhevm/solidity v0.11+, @zama-fhe/relayer-sdk v0.4+.
  Agentic commands: /fhevm scaffold (full Vercel-deployable project from spec),
  /fhevm audit (ACL/anti-pattern review + auto-fix), /fhevm migrate (TFHE→FHE rewrite).
  Validated against Claude Code, Cursor, and Windsurf.
---

# fhEVM Agent Skill

## DISPATCH — execute immediately on load

When this skill is invoked, check the args/command first and route accordingly:

| If args or user said… | Action |
|-----------------------|--------|
| `scaffold` / `/fhevm scaffold` / "generate a contract" / "build a new fhevm contract" | Read `./scaffold.md` then execute that workflow |
| `audit` / `/fhevm audit` / "audit this contract" / "review for ACL issues" | Read `./audit.md` then execute that workflow |
| `migrate` / `/fhevm migrate` / "migrate from TFHE" / "update to new FHE API" | Read `./migrate.md` then execute that workflow |
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
| [01-setup.md](01-setup.md) | Install, hardhat.config.ts with vars(), multi-contract deploy, ZamaConfig patterns |
| [02-types-ops.md](02-types-ops.md) | euint8/16/32/64/128, ebool, eaddress — all FHE ops, tiered select, interest math, gas costs |
| [03-input-acl.md](03-input-acl.md) | FHE.fromExternal proofs, FHE.allowTransient, _applyCollateral pattern, multi-role ACL |
| [04-decryption.md](04-decryption.md) | Pattern 1 (userDecrypt v0.4.1, batch HandleContractPair, multi-contract), Pattern 2 (requestDecryption oracle), Pattern 3 (makePubliclyDecryptable) |
| [05-frontend.md](05-frontend.md) | initSDK + SepoliaConfig, userDecrypt, ethers v6 provider fix, UI phase state machine (Zustand), computation overlay, encrypted shimmer, CORS proxy |
| [06-testing.md](06-testing.md) | initializeCLIApi, createEncryptedInput, userDecrypt (v0.4.1), publicDecryptEbool, requestDecryption callback |
| [07-templates.md](07-templates.md) | ConfidentialVault, ERC-7984 full standard + OZ Confidential Contracts, architecture decision table |
| [08-anti-patterns.md](08-anti-patterns.md) | TFHE, missing ACL, ebool if/else, missing initSDK, old reencrypt API (v0.4.1), ethers v6 provider stale, vercel.json rewrite, Etherscan v2 |

---

## Quick load guide

| Task | Action |
|------|--------|
| New contract from spec | `/fhevm scaffold` |
| Audit existing code | `/fhevm audit` |
| Migrate from old TFHE API | `/fhevm migrate` |
| Manual contract work | Read 01 → 02 → 03 → 04 → 08 |
| Frontend FHE init / relayer issues | Read 05 |
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
v3.3.0 — Proxy header forwarding: hardcoding `content-type: application/json` drops `ZAMA-SDK-VERSION`/`ZAMA-SDK-NAME` → FHE offline; fix: forward original headers. Duplicate env var declarations without `.trim()` cause ENS `{0A}` error; fix: declare once in `config.ts` with `.trim()`, import everywhere. Path extraction anchored to `^\/api\/zama-relay` with `|| '/'` fallback.
