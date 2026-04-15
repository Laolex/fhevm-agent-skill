---
name: fhevm-testing
description: fhEVM testing with Hardhat mock environment — fhevm.initializeCLIApi, createEncryptedInput, userDecrypt (v0.4.1), publicDecrypt, publicDecryptEbool, FHE.requestDecryption callback patterns
type: reference
---

# fhEVM — Testing

## Setup with mock environment
```typescript
import { ethers, fhevm } from "hardhat"; // fhevm injected by @fhevm/hardhat-plugin

describe("MyContract", function () {
    let contract: any;
    let owner: any, user: any;

    before(async function () {
        await fhevm.initializeCLIApi(); // ✅ REQUIRED — must call once before tests
    });

    beforeEach(async function () {
        [owner, user] = await ethers.getSigners();
        const Factory = await ethers.getContractFactory("MyContract");
        contract = await Factory.deploy();
        await contract.waitForDeployment();
    });
});
```

## Encrypt + send (basic pattern)
```typescript
it("should store encrypted value", async function () {
    const encrypted = await fhevm
        .createEncryptedInput(await contract.getAddress(), user.address)
        .add64(BigInt(1000))
        .encrypt();

    await contract.connect(user).deposit(
        encrypted.handles[0],
        encrypted.inputProof
    );
});
```

## Multi-value encrypted input
```typescript
// .add64().add8() — two values, one proof (mirrors contract's multi-input function)
const enc = await fhevm
    .createEncryptedInput(await contract.getAddress(), user.address)
    .add64(BigInt(500))  // weight — handles[0]
    .add8(1)              // vote type — handles[1]
    .encrypt();

await contract.connect(user).castVote(
    enc.handles[0], enc.handles[1], enc.inputProof
);
```

## User Decrypt (user decrypts their own data — SDK v0.4.1)
```typescript
const handle = await contract.getEncryptedBalance(user.address);
const { publicKey, privateKey } = fhevm.generateKeypair();
const contractAddr = await contract.getAddress();
const now = Math.floor(Date.now() / 1000);
const eip712 = fhevm.createEIP712(publicKey, [contractAddr], now, 1);
const typeName = eip712.types.Reencrypt
    ? "Reencrypt"
    : Object.keys(eip712.types).find((k: string) => k !== "EIP712Domain")!;
const sig = await user.signTypedData(
    eip712.domain,
    { [typeName]: eip712.types[typeName] },
    eip712.message
);
const results = await fhevm.userDecrypt(
    [{ handle, contractAddress: contractAddr }],
    privateKey, publicKey, sig,
    [contractAddr], user.address, now, 1,
);
const decrypted = Object.values(results)[0] as bigint;
expect(decrypted).to.equal(BigInt(1000));
```

## Testing Pattern 3: publicDecryptEbool + publicDecrypt (makePubliclyDecryptable)
```typescript
it("should publicly decrypt an ebool", async function () {
    // ... setup position ...
    const tx = await contract.requestLiquidationReveal(borrower.address);
    await tx.wait();

    // Shortcut: read ebool directly in mock env
    const isLiqHandle = await contract.getIsLiquidatable(borrower.address);
    const handleHex = ethers.hexlify(FHE.toBytes32(isLiqHandle));
    const isLiquidatable = await fhevm.publicDecryptEbool(handleHex); // returns boolean
    expect(isLiquidatable).to.be.false;
});

it("should verify reveal via verifyReveal", async function () {
    // ... setup + requestLiquidationReveal ...
    const isLiqHandle = await contract.getIsLiquidatable(borrower.address);
    const handleHex = ethers.hexlify(FHE.toBytes32(isLiqHandle));

    // Mock-decrypt: returns what verifyReveal expects
    const { clearValues, abiEncodedClearValues, decryptionProof } =
        await fhevm.publicDecrypt([handleHex]);

    const verifyTx = await contract.verifyLiquidationReveal(
        borrower.address,
        [handleHex],
        abiEncodedClearValues,
        decryptionProof
    );
    await verifyTx.wait();
    expect(await contract.isPendingReveal(borrower.address)).to.be.false;
});
```

## Testing Pattern 2: FHE.requestDecryption callback
```typescript
it("should decrypt tally via requestDecryption callback", async function () {
    if (!fhevm.isMock) this.skip();

    // Cast vote
    const enc = await fhevm
        .createEncryptedInput(await contract.getAddress(), voter.address)
        .add64(BigInt(100))
        .encrypt();
    await contract.connect(voter).vote(enc.handles[0], 1, enc.inputProof);

    // Advance time past voting end
    await ethers.provider.send("evm_increaseTime", [86400]);
    await ethers.provider.send("evm_mine", []);

    // In mock env, callback fires synchronously in same tx
    const tx = await contract.requestTallyReveal(proposalId);
    await tx.wait();

    expect(await contract.isCallbackCalled(proposalId)).to.be.true;
    const revealed = await contract.getRevealedTally(proposalId);
    expect(revealed).to.equal(BigInt(100));
});
```

## Encrypted input type helpers
```typescript
.add8(n)    // euint8 / externalEuint8
.add16(n)   // euint16
.add32(n)   // euint32
.add64(n)   // euint64
.addBool(b) // ebool / externalEbool
// All return the builder for chaining; .encrypt() finalizes
```
