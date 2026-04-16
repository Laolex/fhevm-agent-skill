# fhEVM Agent Skill

**Production-ready AI agent skill that enables any LLM-powered coding assistant to write, audit, and migrate Zama fhEVM confidential smart contracts.**

Built for the [Zama Bounty Program Season 2](https://github.com/zama-ai/bounty-program) — Bounty Track.

## What it does

This skill turns AI coding assistants (Claude Code, Cursor, Windsurf, GitHub Copilot) into fhEVM experts. Instead of hallucinating deprecated APIs or missing critical ACL patterns, the agent produces correct, deployable confidential contracts on the first try.

### Three agentic commands

| Command | What it does |
|---------|-------------|
| `/fhevm scaffold` | Generates a complete confidential contract + tests + deploy script + frontend from a plain-language spec |
| `/fhevm audit` | Audits contracts and frontends for ACL gaps, anti-patterns, CORS proxy issues, and security vulnerabilities |
| `/fhevm migrate` | Rewrites legacy `TFHE.*` / `GatewayCaller` code to the current `FHE.*` API in one pass |

### Eight reference modules

| Module | Contents |
|--------|----------|
| `00-architecture` | Coprocessor model, ciphertext handles, ACL overview, decryption patterns, Sepolia addresses |
| `01-setup` | Hardhat config, `vars()`, multi-contract deploy, ZamaConfig patterns |
| `02-types-ops` | All encrypted types (`euint8`–`euint128`, `ebool`, `eaddress`), FHE operations, gas costs |
| `03-input-acl` | `FHE.fromExternal` proofs, `allowTransient`, multi-role ACL patterns |
| `04-decryption` | `userDecrypt` (v0.4.1 batch API), `requestDecryption` oracle, `makePubliclyDecryptable` |
| `05-frontend` | `initSDK` + `SepoliaConfig`, relayer SDK, ethers v6 provider fix, Zustand state machine, CORS proxy |
| `06-testing` | Mock coprocessor, `createEncryptedInput`, `publicDecryptEbool`, callback testing |
| `07-templates` | ConfidentialVault, ERC-7984 standard, OpenZeppelin Confidential Contracts |
| `08-anti-patterns` | 15+ common mistakes with correct replacements — the #1 module for preventing hallucinations |

## Why this matters

LLMs consistently fail at fhEVM development because:

1. **Stale training data** — they generate `TFHE.asEuint64()` (removed in v0.9+) instead of `FHE.fromExternal()`
2. **Missing ACL calls** — encrypted values without `FHE.allowThis()` silently return zero
3. **Wrong decryption patterns** — they mix up three distinct decryption flows (userDecrypt, oracle callback, public decrypt)
4. **Frontend integration gaps** — `initSDK` races, CORS proxy misconfiguration, ethers v6 provider issues

This skill eliminates all four failure modes through 3,000+ lines of battle-tested patterns extracted from two production deployments on Sepolia.

## Production validated

Built and validated against real deployed contracts:

- **ShieldLend** — Overcollateralized confidential lending (encrypted collateral, loan amounts, credit scores, liquidation flags). 447-line contract + 295-line credit score module. 57/57 tests passing.
- **ShieldPay** — Confidential payroll with employer/auditor/paymaster roles and FHE salary math. 27/27 tests passing.

Both deployed on Sepolia with full React/Vite frontends using `@zama-fhe/relayer-sdk` v0.4.1.

## Installation

### Claude Code (recommended)

The skill is a directory of markdown files. Point your AI assistant's skill/context system at this repo:

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/Laolex/fhevm-agent-skill.git ~/.claude/skills/fhevm
```

Then add to your `CLAUDE.md` or project instructions:

```markdown
Skills
- fhevm: ~/.claude/skills/fhevm/SKILL.md — fhEVM confidential smart contract development
```

### Cursor / Windsurf / Other

Add the `SKILL.md` file (or the entire directory) as context in your IDE's AI configuration. The skill auto-dispatches based on user intent — no special setup beyond loading the files.

## Quick start

```
> /fhevm scaffold

Build me a confidential voting contract where each vote is encrypted,
only the admin can trigger a tally reveal, and voters can verify their
own vote was counted.
```

The agent will:
1. Parse your spec and choose the right architecture
2. Generate `ConfidentialVoting.sol` with proper ACL, encrypted tallies, and `makePubliclyDecryptable` reveal
3. Generate Hardhat tests using mock coprocessor
4. Generate a deploy script
5. Generate a React frontend snippet with `initSDK` + `userDecrypt`

## Architecture

```
SKILL.md (dispatch router)
├── scaffold.md     — full project generator
├── audit.md        — security/correctness auditor
├── migrate.md      — TFHE→FHE migration tool
├── 00-architecture — fhEVM internals
├── 01-setup        — project scaffolding
├── 02-types-ops    — encrypted types & operations
├── 03-input-acl    — input proofs & access control
├── 04-decryption   — three decryption patterns
├── 05-frontend     — React/Vite + relayer SDK
├── 06-testing      — Hardhat mock coprocessor
├── 07-templates    — contract templates + ERC-7984
└── 08-anti-patterns — common mistakes & fixes
```

The `SKILL.md` entry point acts as a router: it reads the user's intent and loads only the relevant modules, keeping context usage efficient.

## Compatibility

| AI Assistant | Status | Notes |
|-------------|--------|-------|
| Claude Code | Validated | Native skill support via `~/.claude/skills/` |
| Cursor | Validated | Add as context files or rules |
| Windsurf | Validated | Add as context files |
| GitHub Copilot | Compatible | Add as workspace context |
| Any MCP-capable agent | Compatible | Serve modules as MCP resources |

## Tech coverage

- `@fhevm/solidity` v0.11+
- `@zama-fhe/relayer-sdk` v0.4.1+
- Encrypted types: `euint8`, `euint16`, `euint32`, `euint64`, `euint128`, `ebool`, `eaddress`
- All FHE operations: arithmetic, comparison, bitwise, shifts, select, min/max
- ACL: `allowThis`, `allow`, `allowTransient`, `makePubliclyDecryptable`
- Decryption: `userDecrypt` (batch), `requestDecryption` (oracle), public decrypt
- ERC-7984 confidential token standard
- OpenZeppelin Confidential Contracts
- Hardhat + mock coprocessor testing
- React/Vite + ethers v6 frontend integration
- Vercel deployment with CORS proxy

## Version

v3.3.0

## License

MIT
