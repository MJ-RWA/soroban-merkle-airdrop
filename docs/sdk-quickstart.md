# SDK Quick Start

This guide shows how to use the TypeScript SDK to prepare an airdrop, check a recipient's proof, read contract state, and submit a claim. It covers both a Node.js signer and a browser wallet (Freighter).

## 1. Installation

The SDK requires Node.js 20 or later.

```bash
npm install @soroban-merkle-airdrop/sdk
```

For the browser-wallet example in section 6, also install Freighter's client library in your frontend project:

```bash
npm install @stellar/freighter-api
```

Use real Stellar StrKey addresses and the deployed contract ID in place of the illustrative values below. Amounts are `bigint` values in the token's smallest units (not display-formatted decimal amounts).

## 2. Build a Merkle tree

Build the tree using the **deployed airdrop contract ID** and the complete distribution list. The contract ID is part of each leaf hash, so proofs generated for another deployment will not verify.

```ts
import { buildMerkleTree } from "@soroban-merkle-airdrop/sdk";

const contractId = "C..."; // deployed airdrop contract
const entries = [
  { address: "G...", amount: 1_000n },
  { address: "G...", amount: 500n },
];

const { root, proofs } = buildMerkleTree(contractId, entries);

console.log("Merkle root:", root); // 64-character hexadecimal string
const aliceProof = proofs.get(entries[0].address);

if (!aliceProof) {
  throw new Error("Recipient is missing from the generated tree");
}

console.log(aliceProof.proof); // sibling hashes, leaf to root
```

Store the resulting root in the contract during initialization and distribute each recipient's proof securely. Addresses must be unique in the entries list; the builder throws if the list is empty or contains duplicates.

## 3. Verify a proof

Use `verifyProof(root, contractId, address, amount, proof)` to validate a proof locally before attempting a claim.

```ts
import { buildMerkleTree, verifyProof } from "@soroban-merkle-airdrop/sdk";

const contractId = "C...";
const address = "G...";
const amount = 1_000n;

const { root, proofs } = buildMerkleTree(contractId, [
  { address, amount },
  { address: "G...", amount: 500n },
]);

const claimProof = proofs.get(address);
if (!claimProof) throw new Error("No allocation found for this address");

const valid = verifyProof(
  root,
  contractId,
  claimProof.address,
  claimProof.amount,
  claimProof.proof,
);

if (!valid) {
  throw new Error("Invalid Merkle proof");
}
```

The root, contract ID, address, amount, and proof must all correspond to the same distribution and deployment. A changed amount or a proof from a different contract produces a failed verification.

## 4. Query contract state

Create a client with a network preset and the deployed contract ID. The client exposes read methods including `isActive()`, `totalDeposited()`, and `expiration()`.

```ts
import {
  createAirdropClient,
  NETWORKS,
  AirdropContractError,
  RpcError,
} from "@soroban-merkle-airdrop/sdk";

const contractId = "C...";
const client = createAirdropClient({
  ...NETWORKS.testnet,
  contractId,
});

try {
  const active = await client.isActive();
  const deposited = await client.totalDeposited();
  const expiration = await client.expiration();

  console.log("Accepting claims:", active);
  console.log("Total deposited (base units):", deposited.toString());
  console.log(
    "Reclaim available after:",
    expiration === null
      ? "not initialized"
      : new Date(Number(expiration) * 1_000).toISOString(),
  );
} catch (error) {
  if (error instanceof AirdropContractError) {
    console.error("Contract query failed:", error.code, error.message);
  } else if (error instanceof RpcError) {
    console.error("Soroban RPC request failed:", error.message);
  } else {
    throw error;
  }
}
```

`expiration()` returns a Unix timestamp in seconds as a `bigint`, or `null` if the contract has not been initialized. `totalDeposited()` is also returned as a `bigint`; keep amounts in base units when doing calculations.

Other useful read methods include `isClaimed(address)`, `merkleRoot()`, `remainingBalance()`, `totalClaimed()`, `claimedCount()`, and `tokenDecimals()`.

## 5. Submit a claim from Node.js

For a Node.js script or trusted backend, `client.claim(proof, secretKey)` signs and submits the transaction. It requires the claimant's secret key. **Never put a secret key in browser code, a frontend bundle, or source control.**

```ts
import {
  createAirdropClient,
  NETWORKS,
  AirdropContractError,
  RpcError,
  type ClaimProof,
} from "@soroban-merkle-airdrop/sdk";

const client = createAirdropClient({
  ...NETWORKS.testnet,
  contractId: "C...",
});

async function submitClaim(proof: ClaimProof, secretKey: string) {
  try {
    const result = await client.claim(proof, secretKey);
    console.log("Claim confirmed:", result.txHash);
    console.log("Claimed amount:", result.amount.toString());
    return result;
  } catch (error) {
    if (error instanceof AirdropContractError) {
      // Examples include an already-claimed address, an invalid proof,
      // or a contract that is not active.
      console.error("The contract rejected the claim:", error.code, error.message);
    } else if (error instanceof RpcError) {
      // Simulation, submission, RPC, or transaction-confirmation failure.
      console.error("Could not complete the claim:", error.message);
    } else {
      console.error("Unexpected claim error:", error);
    }
    throw error;
  }
}
```

The claim result includes the transaction hash, claimant address, and claimed amount. Handle errors in the UI or calling application; do not treat a submitted transaction as successful until the SDK reports confirmation.

## 6. Browser wallet flow (Freighter)

In a browser, build the transaction with the SDK, ask Freighter to sign it, then pass the signed XDR back to the SDK for submission and confirmation. The user approves the transaction in the wallet; your application never handles their secret key.

Install `@stellar/freighter-api` as shown in section 1.

```ts
import { getAddress, signTransaction } from "@stellar/freighter-api";
import {
  createAirdropClient,
  NETWORKS,
  AirdropContractError,
  RpcError,
  type ClaimProof,
} from "@soroban-merkle-airdrop/sdk";

const network = NETWORKS.testnet;
const contractId = "C...";
const client = createAirdropClient({ ...network, contractId });

async function claimWithFreighter(proof: ClaimProof) {
  try {
    // Request the connected wallet address and ensure it matches the proof.
    const wallet = await getAddress();
    if (wallet.error) throw new Error(wallet.error.message);
    if (wallet.address !== proof.address) {
      throw new Error("Connect the wallet that owns this claim");
    }

    // The SDK simulates and prepares the claim, returning unsigned XDR.
    const unsignedXdr = await client.buildClaimTransaction(proof);

    // Freighter prompts the user to review and approve the transaction.
    const signed = await signTransaction(unsignedXdr, {
      networkPassphrase: network.networkPassphrase,
      address: proof.address,
    });
    if (signed.error) throw new Error(signed.error.message);

    // Submit the wallet-signed XDR and wait for confirmation.
    const result = await client.submitSignedTransaction(
      signed.signedTxXdr,
      proof,
    );

    console.log("Claim confirmed:", result.txHash);
    return result;
  } catch (error) {
    if (error instanceof AirdropContractError) {
      console.error("The contract rejected the claim:", error.code, error.message);
    } else if (error instanceof RpcError) {
      console.error("RPC or transaction error:", error.message);
    } else {
      // Includes wallet not connected, wrong account, or user rejection.
      console.error("Claim was not completed:", error);
    }
    throw error;
  }
}
```

Make sure the wallet is connected to the same network as the SDK client. This example uses testnet; for mainnet, use `NETWORKS.mainnet` and the matching deployed contract ID. A user rejection should be handled as a cancelled action, not as a successful claim.

---

## Next steps

- See [Deploying](deploying.md) for deploying and initializing the Soroban contract.
- See [Architecture](architecture.md) for the Merkle proof and contract design.
- See the [full workflow example](../examples/full-workflow.ts) for a runnable Node.js demonstration.
- See the [SDK source](../sdk/src/index.ts) for the complete exported API.
