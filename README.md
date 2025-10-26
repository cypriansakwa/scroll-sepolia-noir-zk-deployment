# Deploying & Verifying Noir ZK Verifiers on Base Sepolia (Foundry + Noir)

## Introduction

This end-to-end lesson combines onboarding to Base Sepolia with a practical pipeline for deploying and verifying a Noir-generated zero-knowledge (ZK) verifier contract using Foundry. It guides you from wallet/network setup, test-funding and bridging, through RPC configuration and tooling (Foundry, Noir), to deployment, on-chain verification, and proof submission. The guide emphasizes reproducible, secure practices and troubleshooting tips so you can go from a local circuit to an on-chain verifier that returns `true` for valid proofs.

Intended audience: Developers who know MetaMask and basic Solidity tooling and want a practical workflow to deploy and verify ZK verifiers on Base Sepolia.

---

## Objectives

By the end of this lesson you will be able to:

1. Add Base Sepolia network to MetaMask (manual or one-click).
2. Distinguish Base Sepolia test ETH from other Sepolia/Goerli balances and obtain test funds from appropriate faucets.
3. Ensure Base Mainnet uses the official RPC so bridges and faucets' checks succeed.
4. Bridge a small amount of ETH from Ethereum Mainnet → Base Mainnet, and verify balances on BaseScan.
5. Obtain and configure a reliable RPC endpoint (public or provider like Alchemy) and test JSON-RPC connectivity.
6. Create a BaseScan API key for source verification and store it securely in a local `.env` (never commit).
7. Install and use Foundry (`forge`, `cast`) to compile, deploy, and verify Solidity verifier contracts.
8. Generate proofs with Noir, format them for on-chain calls, and call the verifier via explorer or CLI.
9. Troubleshoot common issues (RPC mismatches, faucet/bridge checks, proof formatting).

---

## Prerequisites

- MetaMask installed and an active Ethereum address.
- Basic terminal and bash familiarity.
- A local repo with your contracts and Foundry project (or willing to create one).
- (Recommended) Alchemy / QuickNode account for reliable RPC.
- Noir installed for proof generation (if working with Noir circuits).

---

## Quick reference: chain & RPC values

- Base Sepolia (testnet)
  - Chain ID: 84532
  - Public RPC: https://sepolia.base.org
  - Explorer: https://sepolia.basescan.org
- Base Mainnet
  - Chain ID: 8453
  - Public RPC: https://mainnet.base.org
  - Explorer: https://basescan.org

---

## 1 — Add Base Sepolia to MetaMask

Manual:

1. MetaMask → Settings → Networks → Add Network
2. Fill:
   - Network name: Base Sepolia
   - RPC URL: https://sepolia.base.org
   - Chain ID: 84532
   - Currency: ETH
   - Block explorer: https://sepolia.basescan.org

Alternative: use a one-click/provider link (Chainlist-style) from a trusted source.

---

## 2 — Get Base Sepolia Test ETH (Faucets)

Important: you need Base Sepolia ETH specifically.

Faucets (examples):
- Alchemy Faucet: https://www.alchemy.com/faucets/base-sepolia (requires Alchemy account)
- PK910 Faucet: https://base-sepolia-faucet.pk910.de/
- QuickNode Faucet: https://faucet.quicknode.com/base/sepolia

Note: Some faucets check your Base Mainnet balance (not Ethereum mainnet). If a faucet rejects your request, read "Faucet/Bridge Checks" in Troubleshooting.

---

## 3 — Ensure Base Mainnet RPC is Correct in MetaMask

MetaMask sometimes auto-uses WalletConnect RPC for Base Mainnet which causes checks to fail.

Fix:
1. MetaMask → Settings → Networks → Base (chain ID 8453) → Edit
2. Set RPC URL to: https://mainnet.base.org  
3. Save.

This avoids bridge or faucet failures that inspect Base Mainnet state.

---

## 4 — Bridge a Small Amount to Base Mainnet (Official Base bridge)

If a faucet requires Base Mainnet balance or you want on-chain funds on Base:

1. Visit: https://bridge.base.org
2. Connect MetaMask, set From: Ethereum Mainnet → To: Base Mainnet.
3. Enter a small amount (e.g., 0.001 ETH).
4. Click "Deposit" and confirm the MetaMask transactions (L1 gas tx + L2 message).

Wait ~5–10 minutes for completion and verify balance on BaseScan.

---

## 5 — Verify Balances on BaseScan

- Switch MetaMask to Base Mainnet and/or Base Sepolia as needed.
- Open explorer:
  - Base Mainnet: https://basescan.org/address/<your-address>
  - Base Sepolia: https://sepolia.basescan.org/address/<your-address>

Confirm expected ETH amounts. If not visible, ensure you are on the correct network and check pending transactions.

---

## 6 — Choose and Configure RPC (public vs provider)

Option A — Public:
- Sepolia public RPC: https://sepolia.base.org
- Pros: no signup. Cons: rate limits and outages under load.

Option B — Alchemy (recommended for reliability):
1. Create app on Alchemy: name it `base-sepolia-dev`, choose Base Sepolia.
2. Enable Node API; copy HTTPS RPC: e.g. `https://base-sepolia.g.alchemy.com/v2/YOUR_KEY`.

Set RPC in your environment (example `.env` shown later).

Test connectivity:

```bash
export RPC="https://sepolia.base.org"  # or your Alchemy URL
curl -X POST $RPC \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

Expected simple JSON with `"result": "0x..."`. If you get an error, re-check the RPC URL and network.

---

## 7 — Create a BaseScan API Key

1. Visit: https://basescan.org
2. Sign in / create account.
3. Profile → API Keys → Create new key (name it e.g. `base-sepolia-dev`).
4. Copy the key and keep it secret (store in local `.env`).

This key is used by Foundry to verify source code on BaseScan.

---

## 8 — Store Secrets Locally (.env) — DO NOT COMMIT

Example `.env` in your project `contract/` folder:

```bash
cat > .env <<'EOF'
RPC="https://base-sepolia.g.alchemy.com/v2/YOUR_ALCHEMY_KEY"
PK="0xyour_private_key_here"
ETHERSCAN_API_KEY="your_basescan_api_key"
EOF

# load into shell
source .env
```

Security notes:
- Never commit `.env` or private keys to git.
- Use a hardware wallet for production.
- For CI, use encrypted secrets.

---

## 9 — Export Private Key from MetaMask (careful)

1. MetaMask → Account → Account Details → Export Private Key
2. Confirm and copy `0x...` hex key.
3. Paste into `.env` as `PK="0x..."`.

Do not paste keys into public places or share them.

---

## 10 — Install and Verify Foundry

Install Foundry:

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

Verify:

```bash
forge --version
cast --version
```

If missing, follow Foundry docs: https://book.getfoundry.sh/

Recommended `foundry.toml` snippets (avoid CreateContractSizeLimit):

```toml
[profile.default]
optimizer = true
optimizer_runs = 200
# via_ir = true   # enable if needed
```

---

## 11 — Project layout (example)

```
zk_proof_mul_and_sum/
├─ circuits/        → Noir circuit + proof generation
├─ contract/        → Solidity verifier + Foundry
├─ script/          → Deploy scripts (Script.sol)
└─ js/              → Proof generation / helper scripts
```

Verifier contract typically named `Verifier.sol` (or `HonkVerifier` as an example).

---

## 12 — Deploying with Foundry (manual CLI)

Compile & deploy a contract and attempt auto-verify:

```bash
forge create src/Verifier.sol:Verifier \
  --rpc-url $RPC \
  --private-key $PK \
  --verify \
  --etherscan-api-key $ETHERSCAN_API_KEY \
  --broadcast
```

Notes:
- Replace `src/Verifier.sol:Verifier` with your contract path and name.
- `--verify` triggers source verification on BaseScan using your API key.

If you run into `CreateContractSizeLimit`, increase optimizer runs or enable `via_ir`.

---

## 13 — Scripted Deployment (recommended)

Create `script/Deploy.s.sol`:

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

Run:

```bash
forge script script/Deploy.s.sol \
  --rpc-url $RPC \
  --private-key $PK \
  --broadcast \
  --verify \
  --etherscan-api-key $ETHERSCAN_API_KEY
```

Broadcasts and receipts are saved under `broadcast/`.

---

## 14 — Generate Noir Proofs & Prepare On-Chain Payloads

From your Noir circuit directory:

1. Generate proof:

```bash
bb prove   # or the Noir CLI command you use
```

2. Typical outputs: `target/proof` (binary), `target/public-inputs` (file)

3. Convert binary proof to hex:

```bash
xxd -p target/proof > proof.hex
echo -n "0x$(tr -d '\n' < proof.hex | tr -d ' ')" > proof_formatted.hex
```

4. Ensure public inputs are 32-byte hex values (bytes32). If `public-inputs` is JSON/array, map each to 32-byte hex strings.

Examples:

- Proof: `0x...` (single continuous string in `proof_formatted.hex`)
- Public inputs: `[0x0000...000d, 0x0000...00aa]`

---

## 15 — Verify / Call Verifier (Explorer or CLI)

Explorer (Read Contract):
- Paste `proof` (bytes) — use `proof_formatted.hex`
- Paste `publicInputs` as `bytes32[]` (array of 32-byte hex strings)
- Query — successful verification returns `true`

CLI (`cast`):

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

If `false`, re-check proof generation and public input encoding.

---

## 16 — Common Troubleshooting

- Faucet rejects request:
  - Confirm you're requesting Base Sepolia ETH (not Ethereum Sepolia).
  - Some faucets check Base Mainnet balance — bridge a small amount if required.
  - Ensure MetaMask is connected to the correct network and that your RPC is correct.

- Verification failures:
  - Wrong `ETHERSCAN_API_KEY` or using an API key from another explorer.
  - Source mismatch: ensure you are compiling with the same Solidity version and compiler settings (optimizer, runs).
  - Flattening issues: Foundry's auto-verify usually handles this; use `--verify` with correct args.

- RPC errors / wrong chain:
  - Using an Ethereum Sepolia RPC or wrong base RPC yields incorrect chain IDs. Re-check `RPC` value.
  - Public RPC rate-limited: switch to Alchemy/QuickNode.

- Proof issues:
  - Not hex-encoded or missing `0x` prefix.
  - Public inputs not 32-byte padded.
  - Explorer UI may truncate very large blobs — prefer CLI.

---

## 17 — Security & Best Practices

- Never commit `.env`, private keys, or secrets into source control.
- Use hardware wallets for production deployments.
- Limit test funds and revoke approvals with https://revoke.cash when done.
- Prefer provider RPC (Alchemy) for CI and repeated interactions.
- Rotate API keys and use least-privilege tokens where possible.

---

## 18 — Next Steps & Automation Ideas

- Add a `script/VerifyProof.s.sol` to submit proofs programmatically.
- CI: add a GitHub Actions job to run tests and optional dry-run deployments (use encrypted secrets).
- dApp integration: create a frontend button to submit proofs via the deployed verifier.
- Add events to the verifier contract to signal proof verification for easier UX:

```solidity
event ProofVerified(address indexed sender);
```

- Improve UX by building helper scripts to convert Noir outputs to the exact on-chain encoding.

---

## Resources

- Base Build Portal: https://base.org/build
- Foundry Docs: https://book.getfoundry.sh/
- Noir Language: https://noir-lang.org
- Base public RPCs:
  - Sepolia: https://sepolia.base.org
  - Mainnet: https://mainnet.base.org
- Alchemy: https://alchemy.com
- BaseScan: https://basescan.org & https://sepolia.basescan.org
- Faucet examples: Alchemy Faucet, PK910 Faucet, QuickNode Faucet

---

## Summary

This README provides a single, consolidated lesson for onboarding to Base Sepolia and building a complete pipeline from Noir circuits to on-chain verification using Foundry. Follow the sections in order: wallet/network setup → RPC configuration → Foundry & deployment → proof generation → verification. Keep your secrets local, test connections early, and prefer scripted deployments for repeatability.

Happy building — from Noir circuit to verified on-chain proof!