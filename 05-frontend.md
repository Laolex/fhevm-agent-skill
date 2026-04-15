---
name: fhevm-frontend
description: fhEVM frontend integration — createInstance config, encryptUint, userDecrypt (SDK v0.4.1), ethers v6 provider fix, UI phase state machine, computation overlay, encrypted shimmer, CORS proxy, vite.config.ts
type: reference
---

# fhEVM — Frontend Integration

## createInstance — use SepoliaConfig spread (production pattern)
```typescript
import { createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk/web.js";
// Note: /web.js for browser, /node.js for Node.js

// ✅ PRODUCTION: spread SepoliaConfig — all addresses stay in sync with SDK releases
// SepoliaConfig includes: aclContractAddress, kmsContractAddress, inputVerifierContractAddress,
// verifyingContractAddressDecryption, verifyingContractAddressInputVerification, chainId, gatewayChainId
const provider = new ethers.BrowserProvider(window.ethereum);
const eth = provider.provider; // raw EIP-1193 provider

const fhevmInstance = await createInstance({
    ...SepoliaConfig,                                             // ✅ baked-in correct addresses
    relayerUrl: "https://yourdomain.vercel.app/api/zama-relay",  // your CORS proxy
    network: eth,                                                 // EIP-1193 provider
});

// ✅ vs hardcoding addresses manually — only use hardcoded form if SDK doesn't export SepoliaConfig
// ❌ Do NOT call initFhevm() — removed, causes "invalid EIP-1193 provider" error
// ❌ Do NOT use fhevmjs package — replaced by @zama-fhe/relayer-sdk
```

## Encrypting a value for contract input
```typescript
const encrypted = await fhevmInstance.encryptUint({
    value: BigInt(1000),
    type: "euint64",            // match the contract param type exactly
    contractAddress: CONTRACT_ADDRESS,
    callerAddress: userAddress,
});

const handle = ethers.hexlify(encrypted.handles[0]);   // bytes32
const proof  = ethers.hexlify(encrypted.inputProof);   // bytes

await contract.deposit(handle, proof, { value: ethers.parseEther("0.1") });
```

## Multi-value encryption (one proof, multiple handles)
```typescript
// Frontend mirrors the contract's multi-input function
const encrypted = await fhevmInstance.encryptUint({
    values: [{ value: BigInt(500), type: "euint64" }, { value: 1n, type: "euint8" }],
    contractAddress: CONTRACT_ADDRESS,
    callerAddress: userAddress,
});

const weightHandle = ethers.hexlify(encrypted.handles[0]);
const voteHandle   = ethers.hexlify(encrypted.handles[1]);
const proof        = ethers.hexlify(encrypted.inputProof); // single proof covers both

await contract.vote(weightHandle, voteHandle, proof);
```

## User Decrypt — user decrypts their own data (SDK v0.4.1)

> `reencrypt()` is removed in SDK v0.4.1. Use `userDecrypt()` instead.
> See `04-decryption.md` Pattern 1 for full API reference.

```typescript
const encHandle = await contract.getEncryptedBalance(userAddress);
const { publicKey, privateKey } = fhevmInstance.generateKeypair();

// ✅ v0.4.1: contractAddresses array, startTimestamp, durationDays
const now = Math.floor(Date.now() / 1000);
const eip712 = fhevmInstance.createEIP712(publicKey, [CONTRACT_ADDRESS], now, 1);
const typeName = eip712.types.Reencrypt
    ? "Reencrypt"
    : Object.keys(eip712.types).find((k: string) => k !== "EIP712Domain")!;
const signature = await signer.signTypedData(
    eip712.domain, { [typeName]: eip712.types[typeName] }, eip712.message
);

// ✅ v0.4.1: userDecrypt with HandleContractPair[]
const results = await fhevmInstance.userDecrypt(
    [{ handle: encHandle, contractAddress: CONTRACT_ADDRESS }],
    privateKey, publicKey, signature,
    [CONTRACT_ADDRESS], userAddress, now, 1,
);
const plaintext = Object.values(results)[0] as bigint;
```

## CORS proxy for Zama relayer (required in browser with COEP headers)
The Zama relayer doesn't send CORS/CORP headers. When your site uses `Cross-Origin-Embedder-Policy: require-corp` (needed for WASM), ALL cross-origin fetches are blocked unless proxied through same-origin.

**Critical bug to avoid**: hardcoding `content-type: application/json` on proxy requests. The Zama relayer SDK sends `ZAMA-SDK-VERSION` and `ZAMA-SDK-NAME` headers — if your proxy drops them by overriding headers, the relayer rejects the request silently and `createInstance` fails with FHE showing offline.

```javascript
// api/zama-relay.js — Vercel edge function
// ✅ Forwards original request headers (ZAMA-SDK-VERSION, ZAMA-SDK-NAME, content-type, etc.)
export const config = { runtime: 'edge' };
export default async function handler(req) {
    if (req.method === 'OPTIONS') {
        return new Response(null, {
            status: 204,
            headers: {
                'Access-Control-Allow-Origin': '*',
                'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
                'Access-Control-Allow-Headers': '*',
            },
        });
    }

    const url = new URL(req.url);
    // ✅ Anchor the replace and fallback to '/' — prevents double-replace bugs
    const path = url.pathname.replace(/^\/api\/zama-relay/, '') || '/';
    const target = `https://relayer.testnet.zama.org${path}${url.search}`;

    // ✅ Forward original headers — MUST include ZAMA-SDK-VERSION and ZAMA-SDK-NAME
    // ❌ DO NOT override with { 'content-type': 'application/json' } — drops SDK headers
    const forwardHeaders = {};
    for (const [key, value] of req.headers.entries()) {
        if (!['host', 'connection'].includes(key.toLowerCase())) {
            forwardHeaders[key] = value;
        }
    }

    const response = await fetch(target, {
        method: req.method,
        headers: forwardHeaders,
        body: req.method !== 'GET' && req.method !== 'HEAD' ? req.body : undefined,
    });

    // ✅ Forward actual content-type — binary responses MUST use arrayBuffer
    const contentType = response.headers.get('content-type') ?? 'application/octet-stream';
    const isBinary = contentType.includes('octet-stream') || contentType.includes('binary');
    const data = isBinary ? await response.arrayBuffer() : await response.text();

    return new Response(data, {
        status: response.status,
        headers: {
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
            'Access-Control-Allow-Headers': '*',
            'content-type': contentType,
        },
    });
}
```

```json
// vercel.json — ✅ REQUIRED rewrite rule for sub-path forwarding
// Without this, SDK calls to /api/zama-relay/some/path return 404
{
  "rewrites": [
    { "source": "/api/zama-relay/:path*", "destination": "/api/zama-relay" }
  ]
}
```

Then use `relayerUrl: "https://yourdomain.vercel.app/api/zama-relay"` in `createInstance`.

## vite.config.ts — required WASM worker headers
```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
    plugins: [react()],
    server: {
        headers: {
            "Cross-Origin-Opener-Policy": "same-origin",
            "Cross-Origin-Embedder-Policy": "require-corp",
        },
    },
    optimizeDeps: {
        exclude: ["@zama-fhe/relayer-sdk"],
    },
});
```

## Init pattern with proxy + fallback
```typescript
await initSDK(); // ✅ load WASM first — always before createInstance
let inst = null;
try {
    // Primary: use CORS proxy (required when site has COEP require-corp)
    inst = await Promise.race([
        createInstance({ ...SepoliaConfig, network: eth, relayerUrl: `${window.location.origin}/api/zama-relay` }),
        new Promise((_, rej) => setTimeout(() => rej(new Error("timeout")), 60000)) // ✅ 60s — relayer is slow
    ]);
} catch (e) {
    console.error("fhevmjs init failed:", e);
    // Fallback: SepoliaConfig already has its own relayerUrl baked in
    try {
        inst = await Promise.race([
            createInstance({ ...SepoliaConfig, network: eth }),
            new Promise((_, rej) => setTimeout(() => rej(new Error("fallback timeout")), 30000))
        ]);
    } catch (e2) {
        console.error("fhevmjs fallback failed:", e2);
    }
}
// App degrades gracefully when inst is null — show FHE offline indicator, block FHE actions
```

## Sepolia network enforcement before FHE init
```typescript
// ✅ Always enforce correct network before connecting — prevents "wrong network" errors
async function connect() {
    const provider = new ethers.BrowserProvider(window.ethereum);
    const network  = await provider.getNetwork();
    if (Number(network.chainId) !== CHAIN_ID) {
        await provider.send("wallet_switchEthereumChain", [
            { chainId: `0x${CHAIN_ID.toString(16)}` }
        ]);
    }
    // Now safe to init FHE
    inst = await Promise.race([...]);
}
```

## ETH-wei equivalent helper (multi-asset normalization)
```typescript
// Normalize any token amount to ETH-wei equivalent for FHE input
// Used for cross-collateral: USDC, ZAMA, etc.
function toEthWei(amount: string, token: typeof TOKENS[number]): bigint {
    const raw = parseUnits(amount, token.decimals);        // token raw units
    return (raw * token.ethWeiPerToken) / (10n ** BigInt(token.decimals));
}

// Config (config.ts):
export const TOKENS = [
  { symbol:"ETH",  address:"native", decimals:18, ethWeiPerToken: 1n*(10n**18n) },
  { symbol:"USDC", address:"0x...",  decimals:6,  ethWeiPerToken: 333333333333333n },  // 3000 USDC/ETH
  { symbol:"ZAMA", address:"0x...",  decimals:18, ethWeiPerToken: 10000000000000000n }, // 100 ZAMA/ETH
] as const;
// Price formula: ethWeiPerToken = 1e18 / tokenPerEth
// e.g. 3000 USDC/ETH → 1e18/3000 = 333333333333333
```

## ethers v6 — re-create BrowserProvider after chain switch
```typescript
// ❌ ethers v6 BrowserProvider caches chainId — becomes invalid after wallet_switchEthereumChain
const provider = new ethers.BrowserProvider(window.ethereum);
await provider.send("wallet_switchEthereumChain", [{ chainId: "0xaa36a7" }]);
// provider is now stale — all subsequent calls throw NETWORK_ERROR

// ✅ Re-create provider after chain switch
const eth = window.ethereum;
let provider = new ethers.BrowserProvider(eth);
const network = await provider.getNetwork();
if (Number(network.chainId) !== CHAIN_ID) {
    await eth.request({ method: "wallet_switchEthereumChain", params: [{ chainId: `0x${CHAIN_ID.toString(16)}` }] });
    provider = new ethers.BrowserProvider(eth); // ✅ fresh provider with correct chainId
}
const signer = await provider.getSigner();
```

## UI phase state machine (Zustand)
Lightweight global state for FHE computation UI feedback:
```typescript
// ui/useUiPhase.ts
import { create } from "zustand";

export type UiPhase =
    | "disconnected" | "connecting" | "connected"
    | "encrypting" | "computing" | "decrypted";

type State = { phase: UiPhase; setPhase: (p: UiPhase) => void };
export const useUiPhase = create<State>((set) => ({
    phase: "disconnected",
    setPhase: (p) => set({ phase: p }),
}));

// Usage in App.tsx:
const { setPhase } = useUiPhase.getState();
setPhase("encrypting");  // before FHE encrypt call
setPhase("computing");   // after tx submitted, awaiting confirmation
setPhase("connected");   // after tx confirmed or on error recovery
```

## Computation overlay (encrypting/computing feedback)
Full-screen overlay during FHE operations — prevents user interaction and shows progress:
```tsx
// ui/ComputationOverlay.tsx
import { motion, AnimatePresence } from "framer-motion";
import { useUiPhase } from "./useUiPhase";

export default function ComputationOverlay() {
    const { phase } = useUiPhase();
    const show = phase === "encrypting" || phase === "computing";
    const label = phase === "encrypting"
        ? "Encrypting via FHE relayer..."
        : "Running encrypted computation via fhEVM...";

    return (
        <AnimatePresence>
            {show && (
                <motion.div
                    initial={{ opacity: 0 }} animate={{ opacity: 1 }} exit={{ opacity: 0 }}
                    style={{
                        position: "fixed", inset: 0, zIndex: 200,
                        background: "rgba(0,0,0,0.65)", backdropFilter: "blur(6px)",
                        display: "flex", flexDirection: "column",
                        alignItems: "center", justifyContent: "center", gap: 16,
                    }}
                >
                    <motion.div
                        animate={{ opacity: [0.3, 1, 0.3] }}
                        transition={{ repeat: Infinity, duration: 1.2 }}
                        style={{ fontSize: 13, fontFamily: "'Space Mono', monospace",
                            color: "#a78bfa", letterSpacing: "0.12em" }}
                    >
                        {label}
                    </motion.div>
                    {/* Animated progress bar */}
                    <div style={{ width: 200, height: 3, borderRadius: 99,
                        background: "rgba(139,92,246,0.15)", overflow: "hidden" }}>
                        <motion.div
                            style={{ height: "100%", borderRadius: 99,
                                background: "linear-gradient(90deg, #8b5cf6, #34d399)" }}
                            animate={{ width: ["0%", "80%", "40%", "100%"] }}
                            transition={{ repeat: Infinity, duration: 2, ease: "easeInOut" }}
                        />
                    </div>
                </motion.div>
            )}
        </AnimatePresence>
    );
}
```

## Encrypted field shimmer (pre-decrypt placeholder)
CSS shimmer animation for fields showing encrypted data before user decrypts:
```css
@keyframes enc-shimmer {
    0%   { background-position: -200% 0; }
    100% { background-position: 200% 0; }
}
.enc-shimmer {
    background: linear-gradient(90deg,
        rgba(139,92,246,0.08) 25%, rgba(139,92,246,0.18) 50%, rgba(139,92,246,0.08) 75%);
    background-size: 200% 100%;
    animation: enc-shimmer 1.5s ease-in-out infinite;
    border-radius: 6px;
    color: transparent;
    user-select: none;
}
```

```tsx
// EncryptedField component — shimmer → value with staggered reveal
function EncryptedField({ label, value, phase, delay = 0 }: {
    label: string; value: string;
    phase: "encrypted" | "computing" | "decrypted"; delay?: number;
}) {
    const [revealed, setRevealed] = useState(false);
    useEffect(() => {
        if (phase === "decrypted") {
            const t = setTimeout(() => setRevealed(true), delay);
            return () => clearTimeout(t);
        }
        setRevealed(false);
    }, [phase, delay]);

    return (
        <div>
            <div className="field-label">{label}</div>
            {revealed ? (
                <motion.div initial={{ opacity: 0, filter: "blur(8px)" }}
                    animate={{ opacity: 1, filter: "blur(0px)" }}
                    transition={{ duration: 0.4 }}>
                    {value}
                </motion.div>
            ) : (
                <div className="enc-shimmer" style={{ width: 80, height: 20 }}>████</div>
            )}
        </div>
    );
}

// Usage — stagger 150ms per field:
<EncryptedField label="Collateral" value={formatEther(collateral)} phase={phase} delay={0} />
<EncryptedField label="Debt"       value={formatEther(debt)}       phase={phase} delay={150} />
<EncryptedField label="Rate"       value={`${rate}%`}              phase={phase} delay={300} />
```

## Wallet disconnect pattern
```typescript
// Disconnect clears all state without requiring MetaMask logout
function disconnect() {
    setAddress("");
    setSigner(null);
    setContract(null);
    setFhevmInst(null);
    setPosition(null);
    useUiPhase.getState().setPhase("disconnected"); // ✅ reset UI phase
}
```
