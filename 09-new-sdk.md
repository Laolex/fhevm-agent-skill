---
name: fhevm-new-sdk
description: @zama-fhe/sdk v3 and @zama-fhe/react-sdk v3 — ZamaSDK constructor, Token API (shield/unshield/confidentialTransfer/balanceOf), 59 React hooks, ZamaProvider, session/keypair TTL, new error taxonomy, dependency conflict resolution, Node >= 22 requirement
type: reference
---

# New SDK: `@zama-fhe/sdk@^3` and `@zama-fhe/react-sdk@^3`

Released 2026-04-22. A high-level TypeScript surface built on top of the legacy
`@zama-fhe/relayer-sdk` primitives. The two families coexist — **do not choose one
over the other blindly**; the right choice depends on what you're building.

## Which SDK to Use

| Situation | Use |
|---|---|
| Building a confidential ERC-7984 dApp (shield / unshield / transfer / balance) | `@zama-fhe/sdk` or `@zama-fhe/react-sdk` |
| React dApp | `@zama-fhe/react-sdk` — `ZamaProvider` + 59 hooks |
| Node.js backend / script / Hardhat test runtime | `@zama-fhe/sdk` with `RelayerNode` |
| Custom confidential contract (NOT ERC-7984), raw encrypted inputs | Either SDK via `sdk.userDecrypt` + `sdk.relayer.createEncryptedInput`, or drop to `@zama-fhe/relayer-sdk` primitives (see `05-frontend.md`) |
| Hardhat `npx hardhat test` | Still `@fhevm/hardhat-plugin` + `@zama-fhe/relayer-sdk@0.4.1` — new SDK is for runtime apps |

## Packages

```bash
# React app (browser, hooks)
npm install @zama-fhe/react-sdk@^3.0.0 @tanstack/react-query@^5

# Vanilla TypeScript / Node.js
npm install @zama-fhe/sdk@^3.0.0
```

**Peer requirements:**
- `@zama-fhe/sdk@3.0.0`: `viem >= 2`, `ethers >= 6`, `@tanstack/query-core >= 5`
- `@zama-fhe/react-sdk@3.0.0`: `viem ^2.47.0`, `react >= 18`, `wagmi >= 2`, `@tanstack/react-query >= 5`
- **`engines.node >= 22`** — Node 20 is NOT supported by the new SDK runtime. The Hardhat toolchain (`@fhevm/hardhat-plugin`) still works on Node 20.

## Dependency Conflict: `relayer-sdk` 0.4.1 Pin

`@fhevm/hardhat-plugin@0.4.2` and `@fhevm/mock-utils@0.4.2` peer-pin
`@zama-fhe/relayer-sdk` at **exactly `0.4.1`**. The new `@zama-fhe/sdk@3.0.0`
depends on `~0.4.2`. When both are in the same `package.json`, npm can hoist
`0.4.2` or `0.4.3`, causing Hardhat to abort at startup:

```
Error in plugin @fhevm/hardhat-plugin: Invalid @zama-fhe/relayer-sdk version.
Expecting 0.4.1. Got 0.4.2 instead.
```

**Fix A — `overrides` (single-package projects):**
```json
{
  "overrides": {
    "@zama-fhe/relayer-sdk": "0.4.1"
  }
}
```
Then `rm -rf node_modules package-lock.json && npm install`.

**Fix B — workspace split (monorepos, recommended):**
Keep Hardhat contracts in one workspace package; keep the React/dApp frontend
in a separate workspace. Each workspace gets its own `node_modules`. The
contracts workspace pins `0.4.1`; the frontend workspace installs whatever
`@zama-fhe/sdk` needs.

`0.4.1`, `0.4.2`, and `0.4.3` ship identical transitive deps — forcing `0.4.1`
does not break the new SDK at runtime. Latest published `@zama-fhe/relayer-sdk`
is `0.4.3` (2026-05-06).

## Subpath Exports

```ts
/* Core — browser, vanilla, shared types */
import { ZamaSDK, RelayerWeb, indexedDBStorage } from "@zama-fhe/sdk";

/* Node.js relayer + per-request storage */
import { RelayerNode, asyncLocalStorage } from "@zama-fhe/sdk/node";

/* viem signer adapter */
import { ViemSigner } from "@zama-fhe/sdk/viem";

/* ethers signer adapter */
import { EthersSigner } from "@zama-fhe/sdk/ethers";
```

## ZamaSDK Constructor

```ts
import { ZamaSDK, RelayerWeb, ViemSigner, indexedDBStorage } from "@zama-fhe/sdk";

const relayer = new RelayerWeb({
  getChainId: () => signer.getChainId(),
  transports: {
    11155111: {
      relayerUrl: "/api/relayer/11155111",
      network: "https://sepolia.infura.io/v3/YOUR_KEY",
    },
  },
});

const sdk = new ZamaSDK({
  relayer,
  signer,         // ViemSigner | EthersSigner | WagmiSigner | GenericSigner
  storage,        // indexedDBStorage (browser) | asyncLocalStorage (Node)
  keypairTTL?: number,   // seconds; default 2_592_000 (30 days)
  sessionTTL?: number | "infinite",  // seconds or 0 for sign-every-op
});
```

## Token API

### Creating a Token Instance

```ts
/* Single-contract deployment: token IS the wrapper */
const token = sdk.createToken("0xEncryptedERC20Address");

/* Two-contract deployment: separate wrapper */
const token = sdk.createToken("0xTokenAddress", "0xWrapperAddress");

/* Resolve wrapper from on-chain registry */
const result = await sdk.registry.getConfidentialToken("0xPlainERC20");
if (!result) throw new Error("No wrapper registered");
const token = sdk.createToken("0xPlainERC20", result.confidentialTokenAddress);
```

### Shield (public ERC-20 → confidential)

```ts
/* Default: exact approval — two wallet prompts (approve + shield) */
const { txHash } = await token.shield(1000n);

/* Max approval — first call prompts twice; subsequent shields skip approval */
await token.shield(1000n, { approvalStrategy: "max" });

/* Skip approval — wrapper already approved */
await token.shield(1000n, { approvalStrategy: "skip" });

/* With progress callbacks */
await token.shield(1000n, {
  onApprovalSubmitted: (txHash) => setStatus("Approving..."),
  onShieldSubmitted: (txHash) => setStatus("Shielding..."),
  to: "0xRecipient",  // optional — defaults to connected wallet
});
```

Throws `InsufficientERC20BalanceError` (with `.requested` / `.available`) if
the public balance is insufficient — no transaction is sent.

### Unshield (confidential → public, two-phase)

```ts
/* Simple */
const { txHash } = await token.unshield(500n);

/* With callbacks */
await token.unshield(500n, {
  onUnwrapSubmitted: (hash) => setStatus("Phase 1 submitted..."),
  onFinalizing: () => setStatus("Waiting for decryption proof..."),
  onFinalizeSubmitted: (hash) => setStatus("Done!"),
});

/* Unshield entire balance */
await token.unshieldAll();
```

Throws `InsufficientConfidentialBalanceError` (with `.requested` / `.available`)
if the confidential balance is insufficient.

### Confidential Transfer

```ts
await token.confidentialTransfer("0xRecipient", 100n);
```

Throws `InsufficientConfidentialBalanceError` if balance is insufficient.

### Read Balances

```ts
/* Confidential balance — requires wallet signature + decryption */
const balance: bigint = await token.balanceOf();

/* Public (shielded wrapper) ERC-20 balance — no signing required */
const publicBalance: bigint = await token.underlyingBalanceOf();

/* ReadonlyToken — read-only, batch decrypt for dashboards */
const readonly = sdk.createReadonlyToken("0xToken");
const balance = await readonly.decryptBalanceAs("0xUser", ephemeralPublicKey, signature);
```

## React SDK — `ZamaProvider` Setup

```tsx
"use client"; // Required in Next.js

import { WagmiProvider, createConfig, http } from "wagmi";
import { sepolia } from "wagmi/chains";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ZamaProvider, RelayerWeb, indexedDBStorage } from "@zama-fhe/react-sdk";
import { WagmiSigner } from "@zama-fhe/react-sdk/wagmi";

const wagmiConfig = createConfig({
  chains: [sepolia],
  transports: { [sepolia.id]: http("https://sepolia.infura.io/v3/YOUR_KEY") },
});

const signer = new WagmiSigner({ config: wagmiConfig });
const relayer = new RelayerWeb({
  getChainId: () => signer.getChainId(),
  transports: {
    [sepolia.id]: {
      relayerUrl: "/api/relayer/11155111",
      network: "https://sepolia.infura.io/v3/YOUR_KEY",
    },
  },
});

const queryClient = new QueryClient();

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <WagmiProvider config={wagmiConfig}>
      <QueryClientProvider client={queryClient}>
        <ZamaProvider relayer={relayer} signer={signer} storage={indexedDBStorage}>
          {children}
        </ZamaProvider>
      </QueryClientProvider>
    </WagmiProvider>
  );
}
```

`ZamaProvider` props: `relayer` (required), `signer` (required), `storage`
(required — use `indexedDBStorage` for browsers, `memoryStorage` for tests),
`sessionStorage?` (optional — `chromeSessionStorage` for MV3 extensions),
`keypairTTL?` (seconds, default 30 days), `sessionTTL?` (seconds or `"infinite"`).

### Next.js SSR Note

The SDK runs FHE in a Web Worker and uses IndexedDB. Neither exists during SSR.
Any file importing from `@zama-fhe/react-sdk` **must** be a Client Component
(`"use client"`). Keep the `Providers` wrapper in a client file; `layout.tsx`
can stay a Server Component by only importing that wrapper.

## Key React Hooks (selected from 59 total)

```tsx
import {
  useToken,
  useShield, useUnshield,
  useConfidentialBalance, usePublicBalance,
  useEncrypt,
  useUserDecrypt,
  useAllow,
} from "@zama-fhe/react-sdk";

/* Create a memoized token instance */
const token = useToken({ tokenAddress: "0x...", wrapperAddress: "0x..." });

/* Shield tokens */
const { mutateAsync: shield, isPending } = useShield({ token });
await shield({ amount: 1000n });

/* Unshield */
const { mutateAsync: unshield } = useUnshield({ token });
await unshield({ amount: 500n });

/* Read confidential balance (auto-refreshes after mutations) */
const { data: balance, isLoading } = useConfidentialBalance({ token });

/* Read public ERC-20 balance */
const { data: publicBal } = usePublicBalance({ tokenAddress: "0x..." });

/* Encrypt arbitrary input for custom contracts */
const { mutateAsync: encrypt } = useEncrypt();
const { handles, inputProof } = await encrypt({
  values: [{ value: 1000n, type: "euint64" }],
  contractAddress: "0x...",
  userAddress: "0x...",
});
await contract.deposit(toHex(handles[0]), toHex(inputProof));

/* User decrypt (re-encryption — reads user's own private data) */
const { mutateAsync: userDecrypt } = useUserDecrypt();
const result = await userDecrypt({ handle: "0x...", contractAddress: "0x..." });
```

## Error Handling

Every SDK error extends `ZamaError` and carries a `.code` string.

```ts
import { matchZamaError } from "@zama-fhe/sdk"; // or @zama-fhe/react-sdk

const message = matchZamaError(error, {
  SIGNING_REJECTED:                () => "Approve in your wallet",
  ENCRYPTION_FAILED:               () => "Encryption failed — retry",
  TRANSACTION_REVERTED:            (e) => `Transaction failed: ${e.message}`,
  NO_CIPHERTEXT:                   () => "No confidential balance — shield tokens first",
  INSUFFICIENT_CONFIDENTIAL_BALANCE: (e) => `Need ${e.requested}, have ${e.available}`,
  INSUFFICIENT_ERC20_BALANCE:      (e) => `Not enough tokens: ${e.available} available`,
  BALANCE_CHECK_UNAVAILABLE:       () => "Sign to verify balance first",
  KEYPAIR_EXPIRED:                 () => "Session expired — reconnect wallet",
  _: (e) => `Unexpected error: ${e.message}`,
});
```

Full error class list: `SigningRejectedError`, `SigningFailedError`,
`EncryptionFailedError`, `DecryptionFailedError`, `ApprovalFailedError`,
`TransactionRevertedError`, `InvalidKeypairError`, `KeypairExpiredError`,
`NoCiphertextError`, `RelayerRequestFailedError`, `ConfigurationError`,
`InsufficientConfidentialBalanceError`, `InsufficientERC20BalanceError`,
`BalanceCheckUnavailableError`, `ERC20ReadFailedError`,
`DelegationNotPropagatedError`.

## Legacy vs New SDK — at a Glance

| Feature | Legacy (`@zama-fhe/relayer-sdk@0.4.x`) | New (`@zama-fhe/sdk@^3`) |
|---|---|---|
| Init pattern | `initSDK()` + `createInstance({...SepoliaConfig})` | `new ZamaSDK({ relayer, signer, storage })` |
| Encrypt input | `instance.createEncryptedInput(contract, user).add64(n).encrypt()` | `useEncrypt()` hook or `sdk.relayer.createEncryptedInput(...)` |
| User decrypt | `instance.userDecrypt(...)` | `sdk.userDecrypt(...)` or `useUserDecrypt()` |
| Token ops | Manual contract calls | `token.shield()`, `token.unshield()`, `token.confidentialTransfer()` |
| React | Manual state + useEffect | 59 TanStack Query hooks |
| Session/keypair | Per-instance, no TTL management | `keypairTTL`, `sessionTTL`, `indexedDBStorage` |
| Error handling | Catch raw ethers/viem errors | `ZamaError` + `matchZamaError` |
| Node requirement | Node 20+ | **Node 22+** |
