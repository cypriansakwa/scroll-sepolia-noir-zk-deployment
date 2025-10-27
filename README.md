# End-to-End Zero-Knowledge Proof Deployment with Noir and Foundry on Base Sepolia

This repository demonstrates an **end-to-end Noir zero-knowledge proof workflow** — from writing a simple circuit, generating proofs with `bb`, and deploying a Solidity verifier using **Foundry** — all on the **Base Sepolia testnet**.

It provides a practical template for developers integrating Noir circuits into on-chain verification flows, showcasing:

- 🧮 **Noir circuit design and compilation**  
- 🔏 **Proof generation via `bb prove`**  
- ⚙️ **Solidity verifier integration and deployment**  
- 🌐 **Local and testnet interaction scripts**

Ideal for **students, researchers, and developers** learning Noir and **Base’s ZK infrastructure**.

---

It contains a minimal Noir zero-knowledge circuit that proves knowledge of two secret values `x` and `y` such that `x * y == 42` while only revealing their sum `x + y` publicly. This project demonstrates the end-to-end pipeline: circuit → tests → proof generation → proof formatting → Solidity verifier → deploy & verify on Base Sepolia using Foundry.

Contents:
- Circuit description & example code
- Unit tests
- Proof generation (CLI and JS helpers)
- Converting proofs for on-chain submission
- Solidity verifier & Foundry integration
- Onboarding, funding and deploying to Base Sepolia
- Calling the verifier (explorer & CLI)
- Troubleshooting, security, next steps, and resources

---

## Table of Contents

- [End-to-End Zero-Knowledge Proof Deployment with Noir and Foundry on Base Sepolia](#end-to-end-zero-knowledge-proof-deployment-with-noir-and-foundry-on-base-sepolia)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Objectives](#objectives)
  - [Prerequisites](#prerequisites)
  - [Circuit description](#circuit-description)
  - [Example Noir circuit](#example-noir-circuit)
  - [Unit tests](#unit-tests)
  - [Project structure (recommended)](#project-structure-recommended)
  - [Build \& Test (Noir)](#build--test-noir)
  - [Proof generation](#proof-generation)
    - [CLI approach (example)](#cli-approach-example)
    - [JS helper approach](#js-helper-approach)
  - [Convert proof to hex (for on-chain)](#convert-proof-to-hex-for-on-chain)
  - [Solidity verifier \& Foundry integration](#solidity-verifier--foundry-integration)
    - [Install Foundry](#install-foundry)
    - [Run Foundry tests](#run-foundry-tests)
  - [Deploying to Base Sepolia (onboarding \& funding)](#deploying-to-base-sepolia-onboarding--funding)
    - [Quick reference](#quick-reference)
    - [Add Base Sepolia to MetaMask](#add-base-sepolia-to-metamask)
    - [Faucets](#faucets)
    - [Bridge small amount to Base Mainnet (if needed)](#bridge-small-amount-to-base-mainnet-if-needed)
  - [RPC, .env and secret management](#rpc-env-and-secret-management)
  - [Deploy with Foundry](#deploy-with-foundry)
  - [Calling / verifying proofs on-chain](#calling--verifying-proofs-on-chain)
  - [Troubleshooting](#troubleshooting)
  - [Security \& best practices](#security--best-practices)
  - [Next steps \& automation ideas](#next-steps--automation-ideas)
  - [Resources](#resources)
  - [Cheat sheet — copyable commands](#cheat-sheet--copyable-commands)
  - [License \& acknowledgements](#license--acknowledgements)

---

## Introduction

This project demonstrates selective disclosure using zero-knowledge proofs: you prove the multiplicative relationship `x * y = 42` while revealing only `sum = x + y`. It's a simple, didactic ZK example that is useful for education, demos, and as a starting point for more complex protocols.

It includes:
- the Noir circuit,
- unit tests,
- guidance for generating proofs (Barretenberg CLI or JS),
- instructions to convert proofs into EVM-callable formats,
- a pattern to deploy a verifier contract with Foundry to Base Sepolia,
- and instructions to call/verify proofs on-chain.

---

## Objectives

By following this tutorial you'll be able to:
1. Understand the circuit logic and constraints.
2. Run and interpret unit tests for the circuit.
3. Generate a proof and format it for on-chain verification.
4. Integrate generated verifier artifacts into a Solidity project and test with Foundry.
5. Deploy the verifier contract to Base Sepolia and verify with BaseScan.
6. Submit proofs on-chain (via explorer or `cast`) and interpret results.
7. Apply security best practices for keys, RPCs, and API keys.

---
## Prerequisites
Make sure you have the following installed:
- [Rust](https://www.rust-lang.org/tools/install)
- [Noir](https://noir-lang.org/)
- [bb CLI](https://github.com/noir-lang/noir/releases)
- [Foundry](https://book.getfoundry.sh/)
- [Node.js + npm](https://nodejs.org/)
- [MetaMask](https://metamask.io/) configured with **Base Sepolia**

## Circuit description

Goal: prove knowledge of secret values `x` and `y` such that:
- Constraint: `x * y == 42`
- Public output: `sum = x + y`
- Secrets `x` and `y` remain private

This demonstrates selective disclosure: the verifier learns the sum and that the prover knows factors of 42 without learning the factors themselves.

---

## Example Noir circuit

File: `src/main.nr`

```rust
fn main(x: Field, y: Field) -> pub Field {
    // Constraint: product equals 42
    assert(x * y == 42);

    // Compute sum (public output)
    let sum = x + y;
    sum
}
```

Notes:
- The function returns a public `Field`: the `sum`.
- `assert(...)` creates circuit constraints verified by the prover & verifier.

---

## Unit tests

Add tests in `src/main.nr` (or the place your Noir toolchain expects tests):

```rust
#[test]
fn test_valid_factors_6_and_7() {
    let x = 6;
    let y = 7;
    let sum = main(x, y);
    assert(sum == 13);
}

#[test]
fn test_valid_factors_21_and_2() {
    let x = 21;
    let y = 2;
    let sum = main(x, y);
    assert(sum == 23);
}

#[test]
fn test_valid_factors_negative_6_and_negative_7() {
    let x = -6;
    let y = -7;
    let sum = main(x, y);
    assert(sum == -13);
}

#[test(should_fail)]
fn test_invalid_factors_should_fail() {
    let x = 5;
    let y = 8; // 5*8 != 42
    let _ = main(x, y); // should fail
}

#[test(should_fail)]
fn test_invalid_sum_should_fail() {
    let x = 7;
    let y = 6;
    let _ = main(x, y); // sum != 10, ensure a failing test case
}
```

Run tests with your Noir runner (e.g. `nargo test` depending on toolchain version). Include both positive and negative tests to ensure constraints behave as expected.

---

## Project structure (recommended)

```
zk_proof_mul_and_sum/
├─ src/main.nr         # Noir circuit + tests
├─ target/             # build & proof artifacts (generated)
├─ js/                 # optional JS helpers to generate/format proofs
├─ contract/           # Solidity verifier + Foundry project (src/, test/, script/)
└─ README.md
```

Keep generated artifacts under `target/` and do not commit private keys or large binary proof files to source control.

---

## Build & Test (Noir)

Basic commands (tooling may vary by Noir version):

```bash
# Build the circuit
nargo build

# Run circuit tests
nargo test
```

Troubleshooting:
- If tests fail, inspect the failing assertion and witness values.
- Ensure your Noir and Barretenberg (`bb`) versions match those used to build the verifier.

---

## Proof generation

Two common approaches:

1. CLI (Noir + Barretenberg `bb`) — low-level, reliable for reproducible artifacts.
2. JS helper scripts — convenient for automation, CI, and web integration.

### CLI approach (example)

1. Generate a witness for a concrete input:

```bash
# Example: generate witness for x=6,y=7 (tool names/options vary)
nargo execute --input '{ "x": 6, "y": 7 }' -o target/witness.gz
```

2. Use Barretenberg CLI (`bb`) to create proof:

```bash
bb prove -b ./target/zk_proof_mul_and_sum.json \
         -w target/witness.gz \
         -o ./target \
         --oracle_hash keccak
```

Outputs:
- `target/proof` (binary)
- `target/public-inputs` (JSON/text)
- Other metadata needed to generate the verifier contract

### JS helper approach

If you prefer automation, include a `js/` folder with scripts that call the above steps. Example `package.json` scripts:

```json
{
  "scripts": {
    "generate-proof": "node generate-proof.js"
  }
}
```

`generate-proof.js` could:
- run `nargo` to build + execute,
- call `bb prove`,
- convert outputs to on-chain formats (hex + bytes32[]).

Advantages:
- Easier integration into web apps or CI
- It can automatically format proofs for contract calls

---

## Convert proof to hex (for on-chain)

EVM verifiers typically accept:
- `proof` as `bytes` (a hex string starting with `0x`)
- `publicInputs` as `bytes32[]` (each input padded to 32 bytes)

Commands to convert:

```bash
# Convert binary proof to plain hex (no 0x)
xxd -p target/proof > proof.hex

# Create a single-line, 0x-prefixed proof string
echo -n "0x$(tr -d '\n' < proof.hex | tr -d ' ')" > proof_formatted.hex

# Check beginning:
head -c 128 proof_formatted.hex
```

Public inputs:
- Convert each decimal or JSON input to a 32-byte hex string (left-pad with zeros).
- Provide them as a `bytes32[]` in the on-chain call.

Tip: include helper scripts in `js/` to transform public inputs to EVM-ready bytes32 values.

---

## Solidity verifier & Foundry integration

Most ZK backends produce a Solidity verifier. Typical tasks:
- Add generated verifier contract to `contract/src/Verifier.sol`.
- Add helper contracts / wrappers for calling the verifier.
- Use Foundry (`forge`, `cast`) to test & deploy.

### Install Foundry

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup

forge --version
cast --version
```

Recommended `foundry.toml`:

```toml
[profile.default]
optimizer = true
optimizer_runs = 200
# via_ir = true   # enable if required for contract size or performance
```

### Run Foundry tests

Place your verifier and tests in `contract/`:

```bash
cd contract
forge test --optimize --optimizer-runs 5000 --gas-report -vvv
```

This helps detect encoding issues before deployment.

---

## Deploying to Base Sepolia (onboarding & funding)

### Quick reference

- Base Sepolia (testnet)  
  - Chain ID: `84532`  
  - Public RPC: `https://sepolia.base.org`  
  - Explorer: `https://sepolia.basescan.org`

- Base Mainnet  
  - Chain ID: `8453`  
  - Public RPC: `https://mainnet.base.org`  
  - Explorer: `https://basescan.org`

### Add Base Sepolia to MetaMask

MetaMask → Settings → Networks → Add Network

- Network name: `Base Sepolia`  
- RPC URL: `https://sepolia.base.org`  
- Chain ID: `84532`  
- Currency: `ETH`  
- Block explorer: `https://sepolia.basescan.org`

### Faucets

You need Base Sepolia ETH (not Ethereum Sepolia):

- Alchemy Faucet: https://www.alchemy.com/faucets/base-sepolia (requires Alchemy account)
- PK910 Faucet: https://base-sepolia-faucet.pk910.de/
- QuickNode Faucet: https://faucet.quicknode.com/base/sepolia

Some faucets check your Base Mainnet balance; if you get rejections, consider bridging a small amount to Base Mainnet.

### Bridge small amount to Base Mainnet (if needed)

1. Visit: https://bridge.base.org  
2. Connect MetaMask.  
3. From: Ethereum Mainnet → To: Base Mainnet.  
4. Enter a small amount (e.g., `0.001 ETH`) → Deposit → confirm L1 & L2 messages.  
5. Wait ~5–10 minutes, verify on BaseScan.

---

## RPC, .env and secret management

Prefer provider RPC (Alchemy / QuickNode) for CI and reliability. Public RPC works but may be rate-limited.

Example `.env` (do not commit):

```bash
RPC="https://base-sepolia.g.alchemy.com/v2/YOUR_ALCHEMY_KEY"
PK="0xYOUR_PRIVATE_KEY"
ETHERSCAN_API_KEY="your_basescan_api_key"
```

Load locally:

```bash
source .env
```

Security:
- Add `.env` to `.gitignore`.
- Use CI secrets for automated pipelines.
- For production, use a hardware wallet and do not export private keys.

Export private key from MetaMask (desktop):
- MetaMask → Account → Account Details → Export Private Key → confirm → copy hex → paste into `.env` as `PK="0x..."`.
- Never publish or commit this key.

---

## Deploy with Foundry

Manual CLI deploy (example):

```bash
forge create src/Verifier.sol:Verifier \
  --rpc-url $RPC \
  --private-key $PK \
  --verify \
  --etherscan-api-key $ETHERSCAN_API_KEY \
  --broadcast
```

Scripted deployment (recommended): `contract/script/Deploy.s.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;
import {Script, console} from "forge-std/Script.sol";
import {Verifier} from "../src/Verifier.sol";

contract DeployScript is Script {
    function run() public {
        vm.startBroadcast();
        Verifier v = new Verifier();
        console.log("Verifier deployed at:", address(v));
        vm.stopBroadcast();
    }
}
```

Run script:

```bash
forge script script/Deploy.s.sol \
  --rpc-url $RPC \
  --private-key $PK \
  --broadcast \
  --verify \
  --etherscan-api-key $ETHERSCAN_API_KEY
```

---

## Calling / verifying proofs on-chain

Prefer CLI for large proof payloads.

Example using `cast`:

```bash
PROOF=$(cat proof_formatted.hex)
cast call <VERIFIER_ADDRESS> \
  "verify(bytes,bytes32[])(bool)" \
  "$PROOF" \
  '[0x000000000000000000000000000000000000000000000000000000000000000d]' \
  --rpc-url $RPC
```

Expected output:

```
bool true
```

If `false` or an error, re-check:
- proof conversion / 0x prefix,
- public inputs padding (32 bytes),
- contract encoding expectations.

You can also use the explorer (BaseScan) "Read Contract" UI, but explorer UIs sometimes truncate very large blobs — prefer the CLI for reliability.

---

## Troubleshooting

Common issues:
- Faucet denies claim: make sure you're requesting Base Sepolia ETH (not Ethereum Sepolia); some faucets require Base Mainnet balance.
- RPC mismatch: ensure `RPC` points to Sepolia (chain ID `84532`).
- Rate limits on public RPC: use provider RPC (Alchemy/QuickNode).
- Verification fails: mismatch compiler version or optimizer runs — ensure `foundry.toml` matches how you compiled when publishing sources for verification.
- Proof encoding: missing `0x` prefix or incorrect public input padding will cause verification to fail.

Debugging checklist:
1. Verify RPC returns a block number (`eth_blockNumber`).
2. Confirm chain ID matches Sepolia (84532).
3. Re-compile contracts with the same solidity version and optimizer settings used for verification.
4. Confirm `target/proof` exists and is properly converted to hex.
5. Try `cast call` locally before using explorer UI.
6. Inspect transaction receipts and explorer logs for revert messages.

---

## Security & best practices

- Never commit `.env`, private keys, or API keys to source control.
- Use ephemeral test accounts for experiments.
- Prefer hardware wallets for production actions.
- Revoke approvals and monitor allowances with https://revoke.cash.
- Use provider RPC and encrypted CI secrets for automation.
- Limit test funds and rotate keys regularly.

---

## Next steps & automation ideas

- Add `script/VerifyProof.s.sol` to programmatically submit proofs.
- Create GitHub Actions to run `nargo test`, `bb prove` dry runs, and Foundry tests with encrypted secrets.
- Build a small frontend to produce witnesses client-side and submit proofs.
- Emit events from the verifier contract (e.g., `event ProofVerified(address indexed who)`).
- Provide helper scripts in `js/` to convert Noir outputs into EVM-ready encodings.

---

## Resources

- Noir: https://noir-lang.org  
- Barretenberg / bb CLI docs (see your `bb` release README)  
- Foundry: https://book.getfoundry.sh/  
- Base docs: https://base.org/build  
- Base Sepolia RPC: https://sepolia.base.org  
- Base Mainnet RPC: https://mainnet.base.org  
- BaseScan (explorer): https://sepolia.basescan.org, https://basescan.org  
- revoke.cash: https://revoke.cash

---

## Cheat sheet — copyable commands

```bash
# Build & test noir
nargo build
nargo test

# Create witness & prove (CLI)
nargo execute --input '{"x":6,"y":7}' -o target/witness.gz
bb prove -b ./target/zk_proof_mul_and_sum.json -w target/witness.gz -o ./target --oracle_hash keccak

# Convert proof to hex
xxd -p target/proof > proof.hex
echo -n "0x$(tr -d '\n' < proof.hex | tr -d ' ')" > proof_formatted.hex

# Foundry: test & deploy
cd contract
forge test --optimize --optimizer-runs 5000 -vvv
export RPC="https://base-sepolia.g.alchemy.com/v2/YOUR_KEY"
export PK="0xYOUR_PRIVATE_KEY"
export ETHERSCAN_API_KEY="your_key"
forge create src/Verifier.sol:Verifier --rpc-url $RPC --private-key $PK --verify --etherscan-api-key $ETHERSCAN_API_KEY --broadcast

# Call verifier (cast)
PROOF=$(cat proof_formatted.hex)
cast call <VERIFIER_ADDRESS> "verify(bytes,bytes32[])(bool)" "$PROOF" '[0x00...0d]' --rpc-url $RPC
```

---

## License & acknowledgements

- Put your license here (e.g., MIT).
- Acknowledge Noir & Barretenberg projects if applicable.

---

Thanks for following this tutorial — try the hands-on exercise: use `x=6`, `y=7`, generate the proof, deploy the verifier to Base Sepolia, and confirm `verify(...)` returns `true`. If you want, I can also generate a Foundry `foundry.toml`, `contract/src/Verifier.sol` stub, and a `js/` helper script next.