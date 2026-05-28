---
name: fhevm-audit
description: Agentic command — audit an fhEVM contract or frontend for ACL gaps, anti-patterns, missing signatures, CORS proxy issues, and security vulnerabilities. Returns a numbered issue list with file:line references and fix snippets.
type: command
trigger: /fhevm audit
---

# /fhevm audit — fhEVM Security & Correctness Auditor

You are an fhEVM code auditor. When invoked, read every target file the user specifies (or the entire project if no specific file is given) and run the full checklist below. Do NOT skip checks because a file looks clean — run every check on every relevant file.

---

## Step 1 — Collect target files

If the user gave file paths: read those files.
If the user said "audit the project" or similar: glob for `contracts/**/*.sol`, `test/**/*.ts`, `frontend/src/**/*.ts`, `frontend/src/**/*.tsx`, `api/**/*.js`, `vercel.json`.

Read every file before starting the checklist.

---

## Step 2 — Run the full audit checklist

For each issue found: record **[SEVERITY] File:line — description + fix snippet**.

Severity levels: `[CRITICAL]` (security hole), `[HIGH]` (broken functionality), `[MEDIUM]` (correctness risk), `[LOW]` (best practice)

### A. Contract ACL checks (08-anti-patterns + 03-input-acl)

- [ ] **A1** Every `FHE.fromExternal(...)` is immediately followed by `FHE.allowThis(result)` — if not: `[HIGH]`
- [ ] **A2** Every computed value (`FHE.add`, `FHE.sub`, `FHE.mul`, `FHE.select`, `FHE.div`, `FHE.shr`) has `FHE.allowThis(result)` before being stored — if not: `[HIGH]`
- [ ] **A3** Every user-readable encrypted field has `FHE.allow(value, user)` granted somewhere — if not: `[MEDIUM]`
- [ ] **A4** `FHE.checkSignatures(...)` is called before `abi.decode` in every decryption callback — if not: `[CRITICAL]`
- [ ] **A5** No encrypted value is emitted in events — if yes: `[HIGH]` privacy leak
- [ ] **A6** No `mapping(address => euint64) public` — encrypted handles must be private — if yes: `[HIGH]`
- [ ] **A7** Every Pattern-3 callback that receives `bytes32[] calldata handlesList` asserts `require(handlesList[i] == FHE.toBytes32(expected_i))` for every expected handle BEFORE `FHE.checkSignatures` — if not: `[CRITICAL]` handle-substitution attack lets any caller pass a forged pair (e.g. debt==0, isLiquidatable==true) that a valid KMS signature covers
- [ ] **A8** Any function that accepts `externalEuint*` meant to represent a plaintext-observable quantity (token→ETH equivalence, USD value, collateral credit) clamps against a plaintext upper bound via `FHE.select(FHE.lt(encCap, encIn), encCap, encIn)` — if unclamped: `[CRITICAL]` user over-reports freely, collateral model is fictional
- [ ] **A9** Any function returning an encrypted predicate (`meetsThreshold`, `isEligible`, `hasBalance`) is role-gated (`msg.sender == subject || hasRole(READER_ROLE, msg.sender)`) — if callable by anyone against anyone: `[HIGH]` binary-search leak decodes the encrypted value in O(log n) calls

### B. Contract API checks (08-anti-patterns + 02-types-ops)

- [ ] **B1** No `TFHE.*` usage anywhere (TFHE.asEuint64, TFHE.add, etc.) — if yes: `[HIGH]` won't compile
- [ ] **B2** No `if (eboolVar)` or `require(eboolVar)` — ebool is not bool — if yes: `[HIGH]` compile error
- [ ] **B3** No `FHE.div(a, euintVar)` — second arg must be plaintext — if yes: `[HIGH]` compile error
- [ ] **B4** No `FHE.sub(a, b)` without underflow guard — must use `FHE.select(FHE.not(FHE.lt(a,b)), FHE.sub(a,b), zero)` — if yes: `[MEDIUM]` wraps to huge number
- [ ] **B5** Contract inherits `SepoliaConfig` or `ZamaEthereumConfig` — if not: `[CRITICAL]` FHE ops fail at runtime
- [ ] **B6** No old Gateway API (`GatewayCaller`, `Gateway.requestDecryption`, `onlyGateway`) — if yes: `[HIGH]` won't compile
- [ ] **B7** No `euint64 x = euint64.wrap(handle)` without going through `FHE.fromExternal` — if yes: `[CRITICAL]` no proof validation
- [ ] **B8** If multi-liquidator pattern: contract uses `AccessControlEnumerable` not `AccessControl` — if plain AccessControl: `[MEDIUM]` can't iterate role members

### C. Decryption pattern checks (04-decryption)

- [ ] **C1** No `FHE.requestDecryption` usage anywhere — function is **not present** in `@fhevm/solidity@0.11.x`. Use Pattern 3 (`FHE.makePubliclyDecryptable` + verifyReveal). If found: `[CRITICAL]` won't compile
- [ ] **C2** Pattern 3 `verifyReveal` uses array form `FHE.checkSignatures(bytes32[], bytes, bytes)` — only form available in this stack — if missing: `[HIGH]`
- [ ] **C3** `pendingReveal` mapping is deleted after `verifyReveal` — if not: `[MEDIUM]` replay attack
- [ ] **C4** `abi.decode` type matches the encrypted type (bool for ebool, uint64 for euint64) — if mismatch: `[HIGH]` silent wrong values

### D. Frontend checks (05-frontend + 08-anti-patterns)

- [ ] **D1** `initSDK()` is called before `createInstance()` — if not: `[HIGH]` WASM never loads, FHE always fails
- [ ] **D2** `createInstance` overrides `relayerUrl` with proxy URL — if using SepoliaConfig's direct URL in browser: `[HIGH]` CORS blocks all FHE ops
- [ ] **D3** `vercel.json` (or equivalent) has rewrite rule for `/api/zama-relay/:path*` — if not: `[HIGH]` SDK sub-path calls 404
- [ ] **D4** CORS proxy edge function handles `OPTIONS` preflight — if not: `[HIGH]` browser CORS preflight fails
- [ ] **D5** CORS proxy forwards query string (`url.search`) — if not: `[MEDIUM]` some SDK calls may fail
- [ ] **D6** No `initFhevm()` call (removed from SDK) — if present: `[HIGH]` "invalid EIP-1193 provider" error
- [ ] **D7** Network enforced before FHE init (`wallet_switchEthereumChain`) — if not: `[MEDIUM]` wrong-chain FHE failures
- [ ] **D8** `createInstance` timeout is >= 30s — if < 15s: `[MEDIUM]` Zama relayer can be slow

### E. Environment & deploy checks (01-setup + 08-anti-patterns)

- [ ] **E1** `VITE_*` env vars use `.trim()` when consumed — if not: `[MEDIUM]` trailing newline causes ENS errors
- [ ] **E2** No `process.env.PRIVATE_KEY` in hardhat.config — use `vars.get()` — if yes: `[LOW]` key in env
- [ ] **E3** Etherscan verification: if using `hardhat verify` plugin, note that Etherscan v2 API is broken — recommend manual std_input.json — `[LOW]` informational
- [ ] **E4** If `@zama-fhe/sdk@^3` and Hardhat toolchain are in the same `package.json`: confirm `"overrides": { "@zama-fhe/relayer-sdk": "0.4.1" }` is present — if not: `[HIGH]` Hardhat will abort at startup with version mismatch error

### F. New SDK checks (09-new-sdk — only run if project uses `@zama-fhe/sdk@^3` or `@zama-fhe/react-sdk@^3`)

- [ ] **F1** `ZamaProvider` is wrapped by `QueryClientProvider` — if not: `[CRITICAL]` runtime crash, all hooks throw
- [ ] **F2** All files importing from `@zama-fhe/react-sdk` have `"use client"` (Next.js App Router) — if not: `[HIGH]` SSR crash; IndexedDB/WebWorker unavailable during SSR
- [ ] **F3** `RelayerWeb` is used in browser context, `RelayerNode` in Node.js context — if mixed: `[HIGH]` silent hang or crash
- [ ] **F4** `indexedDBStorage` used in production (not `memoryStorage`) — if memoryStorage in prod: `[MEDIUM]` keypair lost on page refresh, users must re-approve every session
- [ ] **F5** Node.js runtime version is >= 22 if using new SDK — if Node 20: `[HIGH]` new SDK silently fails on import
- [ ] **F6** SDK objects (`relayer`, `signer`) are NOT created at module level in Server Components — if yes: `[HIGH]` SSR crash in Next.js
- [ ] **F7** Error handling uses `matchZamaError` — if catching raw Error with `.message` only: `[LOW]` loses structured error data (`.requested`, `.available`, etc.)

### G. HCU budget checks (10-hcu-costs — run for complex contracts)

- [ ] **G1** Any function with > 5 FHE operations on `euint64`: estimate total HCU (see 10-hcu-costs.md) — if > 5M: `[HIGH]` may hit depth limit
- [ ] **G2** Any use of `FHE.mul(euint64, euint64)` (596K HCU each) in a loop or combined with many other ops: flag total — if > 20M: `[CRITICAL]` tx reverts
- [ ] **G3** Any use of `FHE.rem(euint64, ...)` (~1.2M HCU): flag and recommend bitmasking alternative for power-of-2 moduli — `[MEDIUM]`
- [ ] **G4** Any use of `FHE.div(euint64, ...)` (~700K HCU): confirm plaintext divisor; recommend `FHE.shr` for power-of-2 divisors — `[LOW]` optimization

---

## Step 3 — Report format

Print results grouped by severity:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔴 CRITICAL (must fix before deploy)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[C1] contracts/Vault.sol:47 — Missing FHE.checkSignatures before abi.decode
     Fix: Add `FHE.checkSignatures(handlesList, cleartexts, proof);` before line 47

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🟠 HIGH (broken functionality)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[A1] contracts/Vault.sol:23 — FHE.fromExternal result not allowThis'd
     Fix: Add `FHE.allowThis(encAmount);` after line 23

[D1] frontend/src/App.tsx:89 — initSDK() not called before createInstance
     Fix: Add `await initSDK();` before `createInstance({...})`

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🟡 MEDIUM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔵 LOW / INFORMATIONAL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ PASSED CHECKS: A2, A3, B1, B2, B3, B5, C1, D3, D4 ...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total: X critical, Y high, Z medium, W low
```

After the report, ask: "Apply all fixes automatically? (yes/no)"
If yes: apply every fix in-place using Edit tool, then confirm each change.

Note: Sections F and G are conditional — only run F if the project uses `@zama-fhe/sdk@^3` or `@zama-fhe/react-sdk@^3`, and G for any contract with complex FHE logic. Skip inapplicable sections and mark them as N/A in the passed checks list.
