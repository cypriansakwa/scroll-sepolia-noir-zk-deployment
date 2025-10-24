# Base + Noir ZK Deployment Tutorial

## Deploying and Verifying Noir ZK Proof Verifiers on Base Sepolia Testnet

### Introduction

This tutorial walks through the complete workflow of deploying, verifying, and testing a **zero-knowledge proof verifier** contract on the **Base Sepolia Testnet**.

Context:

- The verifier smart contract is generated from a Noir circuit.
- Deployment is handled using Foundry.
- Verification is done automatically on BaseScan using an Etherscan-style API key.
- Proofs are generated using Noir’s backend (`bb prove`) and verified on-chain.

By the end of this guide you will have a full proof-to-chain pipeline operational on Base Sepolia.

---

## Objectives

1. Configure the environment and tools for Base Sepolia deployment.
2. Deploy a Noir-generated verifier contract using Foundry.
3. Verify the deployed contract on BaseScan.
4. Generate a proof and public inputs with Noir and format them for on-chain verification.
5. Interact with the verified contract on BaseScan or via Foundry CLI.

---

## 1. Environment Setup

### Required Tools

- Foundry — https://book.getfoundry.sh/
- Noir — https://noir-lang.org
- Alchemy (for RPC endpoints) — https://alchemy.com
- BaseScan / Etherscan account (for API verification)

### Environment variables (`.env`)

Create a `.env` file (or export these into your shell):

```bash
RPC="https://base-sepolia.g.alchemy.com/v2/your-key"
PK="0xyour_private_key"
ETHERSCAN_API_KEY="your_etherscan_v2_api_key"
```

Activate:

```bash
source .env
```

---

## 2. Quick Manual Deploy (Foundry CLI)

To deploy the verifier manually with Foundry:

```bash
forge create Verifier.sol:HonkVerifier \
  --rpc-url $RPC \
  --private-key $PK \
  --etherscan-api-key $ETHERSCAN_API_KEY \
  --verify \
  --broadcast
```

What this command does:

- Compiles `Verifier.sol` (contract: `HonkVerifier`)
- Deploys to Base Sepolia via your Alchemy RPC
- Attempts automatic source verification on BaseScan

Troubleshooting: if you encounter this error:

```
EVM error: CreateContractSizeLimit
```

add optimizer settings to `foundry.toml`:

```toml
optimizer = true
optimizer_runs = 200
# via_ir = true
```

---

## 3. Deployment Summary (example)

Example successful output might include:

- Deployer Address: 0x43288a1E4f64D561c93401f3B7A5D37248C4907b
- Contract Name: HonkVerifier
- Contract Address: 0x72b611E03d0f1E482Ef407a5Aff5c69320Dd0c42
- Transaction Hash: 0x415d862f...0c76f67d
- Network: Base Sepolia (chainId=84532)
- Verification: Pass – Verified on BaseScan

---

## 4. Scripted Deployment (Recommended)

If you will deploy frequently or need constructor args, use a deploy script.

Create `script/Deploy.s.sol` inside your repository `script/` folder.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;
import {Script, console} from "forge-std/Script.sol";
import {HonkVerifier} from "../Verifier.sol";
        
contract DeployScript is Script {
    function run() public {
        vm.startBroadcast();
        HonkVerifier verifier = new HonkVerifier();
        console.log("Verifier deployed at:", address(verifier));
        vm.stopBroadcast();
    }
}
```

Run the script:

```bash
forge script script/Deploy.s.sol \
  --rpc-url $RPC \
  --private-key $PK \
  --broadcast \
  --verify \
  --etherscan-api-key $ETHERSCAN_API_KEY
```

This will compile, deploy, verify on BaseScan, and log the contract address. Broadcast output is saved under the `broadcast/` folder, e.g.

```
broadcast/Deploy.s.sol/84532/run-latest.json
```

---

## 5. Viewing the Contract on BaseScan

- Open the contract in the explorer (example):
  - https://sepolia.basescan.org/address/0xAC3aaD8AFa5537C76ED1acAb89E255C0df5537D3

Tabs available: Code | Read Contract | Write Contract | Events

Verification badge:

- A green checkmark indicates "Contract Source Verified" — your Solidity source matches the on-chain bytecode.

Transaction details example:

- https://sepolia.basescan.org/tx/0xbef8490f998450083a5c761974835aad4b769df9c4b8f2c78fe0036f90b688ae (Status: Success)

---

## 6. Interacting with the Verified Contract

You can interact with your verified contract directly on BaseScan (Read Contract tab) or via the CLI.

From the explorer (Read Contract):

1. Choose `verify(bytes, bytes32[])`.
2. Provide:
   - proof (bytes): `0x...` (your hex-encoded proof)
   - publicInputs (bytes32[]): `[0x000...00d]`

Expected output:

```
bool : true
```

This indicates the on-chain verifier returned success.

Notes about explorer limitations:

- The "Read Contract" tab lets you query/view without sending a transaction.
- Explorer UI sometimes doesn't accept very large binary blobs — use hex-encoded strings prefixed with `0x`.
- For real proofs, prefer CLI/scripted calls.

---

## 7. Proof Preparation (Noir Output)

From your Noir circuit folder:

Generate the proof and public inputs:

```bash
bb prove
```

Typical outputs (inside `target/`):

- `target/proof` — the binary proof file
- `target/public-inputs` — public inputs (JSON/array or file)

Convert the binary proof to a hex string (from the same directory as `proof`):

```bash
xxd -p proof > proof.hex
```

`xxd -p` creates a plain hex dump (no `0x` prefix).

Inspect the start (optional):

```bash
head -c 100 proof.hex
```

View public inputs:

```bash
cat public-inputs
```

Each public input must be encoded as a 32-byte hex value for on-chain calls.

Generate a single-line proof with `0x` prefix (recommended):

```bash
echo -n "0x$(cat proof.hex | tr -d '\n' | tr -d ' ')" > proof_formatted.hex
```

Verify formatting:

```bash
cat proof_formatted.hex
```

You should see a single continuous hex string starting with `0x`, e.g.

```
0x0000000000000000000000000000008a620f9793ff4cce08088...
```

---

## 8. Using the Proof on BaseScan or via CLI

On BaseScan (Read Contract):

- `proof (bytes)` — paste the contents of `proof_formatted.hex`
- `publicInputs (bytes32[])` — paste the array of 32-byte hex values, e.g.

```text
[0x0000000000000000000000000000000...000d]
```

Click "Query" to run the read-only call.

From the CLI (Foundry / cast), run from your `contract/` folder:

```bash
cast call <VERIFIER_ADDRESS> \
  "verify(bytes,bytes32[])(bool)" \
  $PROOF \
  '[0x0000000000000000000000000000000...000d]' \
  --rpc-url $RPC
```

Replace `<VERIFIER_ADDRESS>` and `$RPC` as needed.

Expected outcomes:

- `bool : true` — proof verified successfully on-chain.
- `bool : false` — proof invalid or encoding mismatch.
- Errors like `ProofLengthWrong` or `SumcheckFailed` indicate input formatting or proof generation issues.

Practical notes:

- Do not paste raw binary into the explorer UI — use hex-encoded, `0x`-prefixed string.
- Ensure public inputs are `bytes32` values (32-byte hex strings) inside an array.
- For large proofs, prefer CLI/scripted calls.

---

## 9. On-Chain Verification Result (Sample)

Sample read output:

```
bool : true
```

Interpretation:

- `true` → Proof successfully verified.
- `false` → Proof invalid or encoding mismatch.
- Errors such as `ProofLengthWrong` or `SumcheckFailed` indicate incorrect input formatting.

Full proof lifecycle summary:

1. Proof generation in Noir.
2. Contract deployment via Foundry.
3. Verification on BaseScan.
4. Proof validation on-chain.

---

## 10. Using Foundry for Verification Calls

You can call the verifier via `cast`:

```bash
cast call 0xAC3aaD8AFa5537C76ED1acAb89E255C0df5537D3 \
  "verify(bytes,bytes32[])(bool)" \
  $(cat proof_formatted.hex) \
  '[0x000000000000000000000000000000000000000000000000000000000000000d]' \
  --rpc-url $RPC
```

Expected output:

```
bool true
```

---

## Summary and Next Steps

Achievements:

- Full ZK proof verification pipeline on Base Sepolia.
- On-chain proof validation using Noir + Foundry.
- Verified smart contract deployed and visible on BaseScan.

Next suggestions:

- Integrate the verifier into a dApp frontend.
- Add an event for easier UX and logging, e.g.

```solidity
event ProofVerified(address indexed sender);
```

- Automate verification with scripts, e.g.:

```bash
forge script script/VerifyProof.s.sol --rpc-url $RPC --broadcast
```

---

## Conclusion

You have successfully deployed and verified a Noir zero-knowledge proof verifier on Base Sepolia. This workflow is a foundation for ZK-based Layer 2 systems, privacy-preserving dApps, and rollup verifiers.

Follow-up resources:

- Base Build Portal: https://base.org/build
- Noir Language: https://noir-lang.org
- Foundry Docs: https://book.getfoundry.sh/

> “From Noir Circuits to On-Chain Verification - You Built a Full ZK Lifecycle on Base.”
