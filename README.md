# Deploying and Funding Smart Contracts on Base Sepolia Testnet

## Introduction

This lesson walks developers through the end-to-end onboarding and funding process for Base Sepolia testnet so they can deploy and verify smart contracts (e.g., Noir-generated verifiers) using Foundry. You'll learn how to:

- Add Base Sepolia to MetaMask,
- Obtain Base Sepolia test ETH from faucets,
- Bridge a small amount of ETH from Ethereum Mainnet to Base Mainnet (so faucets will allow claims),
- Configure reliable RPC providers (public vs Alchemy),
- Create a BaseScan (Etherscan-style) API key,
- Export a private key safely into a local .env for Foundry/forge,
- Test the RPC connection and prepare for contract deployment.

Intended audience: developers familiar with MetaMask and basic Solidity tooling who want a practical, reproducible workflow for testnet onboarding and contract deployment.

## Objectives

By the end of this lesson you will be able to:

1. Add Base Sepolia network to MetaMask (manual or one-click).
2. Distinguish Base Sepolia test ETH from other Sepolia/Goerli balances and obtain test funds from appropriate faucets.
3. Update MetaMask Base Mainnet configuration to use the official RPC to avoid mis-checks by faucets and bridges.
4. Bridge ETH from Ethereum Mainnet to Base Mainnet using the official Base bridge and verify the deposited balance.
5. Obtain a reliable RPC endpoint (public or Alchemy), set environment variables, and test JSON-RPC connectivity.
6. Create a BaseScan API key for contract verification and store it securely in a .env file (and understand never to commit it).
7. Install and verify Foundry tooling (`forge`, `cast`), and prepare the project for deploying contracts to Base Sepolia.
8. Troubleshoot common issues: faucet errors, RPC mismatches, insufficient balances for bridge/faucet checks.

## Prerequisites

- MetaMask installed and an Ethereum address ready.
- Basic familiarity with terminal and bash.
- (Optional but recommended) Alchemy or QuickNode account for reliable RPC access.
- A local development repository with your contracts and Foundry project structure.

---

## 1 — Step-by-step: Adding Base Sepolia to MetaMask

Steps:

1. Open MetaMask → Settings → Networks → Add Network.
2. Fill in the following network details.

Network details:

- **Network name:** Base Sepolia  
- **RPC URL:** https://sepolia.base.org  
- **Chain ID:** 84532  
- **Currency symbol:** ETH  
- **Block explorer:** https://sepolia.basescan.org

Alternative: use a one-click add from ChainList or other trusted providers (example guidance: https://revoke.cash/learn/wallets/add-network/base-sepolia).

---

## 2 — Get Base Sepolia Test ETH (Free)

Important: You need **Base Sepolia ETH**, not Ethereum Sepolia ETH.

Faucet options:

- Alchemy Faucet — https://www.alchemy.com/faucets/base-sepolia (free Alchemy account required)
- PK910 Faucet — https://base-sepolia-faucet.pk910.de/
- QuickNode Faucet — https://faucet.quicknode.com/base/sepolia

Note: Some faucets require your wallet to have at least 0.001 ETH on Base Mainnet before claiming.

---

## 3 — Update Base Mainnet Network Configuration (MetaMask)

MetaMask often auto-adds Base (chain ID 8453) with a generic WalletConnect RPC `rpc.walletconnect.org/v1/`. This can cause faucet or bridge checks to fail.

Fix: update the Base Mainnet entry to use the official RPC.

1. MetaMask → Settings → Networks
2. Select **Base (chain ID 8453)**
3. Click **Edit**

Replace with:

- **Network Name:** Base Mainnet  
- **New RPC URL:** https://mainnet.base.org  
- **Chain ID:** 8453  
- **Currency Symbol:** ETH  
- **Block Explorer:** https://basescan.org

Click **Save**.

---

## 4 — Faucet May Still Show Errors

Keep in mind:

- The same address (e.g., 0x4328...907b) exists across networks, but balances are network-specific.
- You may have ETH on Ethereum Mainnet but not on Base Mainnet — faucets sometimes check Base Mainnet balance.

If you see faucet errors, confirm the network MetaMask is using and that you hold the required balance on the network the faucet checks.

---

## 5 — Bridge ETH to Base Mainnet (Official Method)

Step-by-step (official bridge):

1. Visit: https://bridge.base.org  
2. Connect MetaMask.  
3. Set From: Ethereum Mainnet → To: Base Mainnet.  
4. Enter amount (e.g., 0.001 ETH — a small amount sufficient for testnet needs).  
5. Click **Deposit** and confirm prompts.

Confirm two MetaMask prompts:

- Ethereum transaction (pays gas).
- L2 message for Base.

Wait ~5–10 minutes for the bridge to complete.

---

## 6 — Verify Your Balance

After bridging:

1. Switch MetaMask to **Base Mainnet**.
2. Check your address on BaseScan (example):  
   https://basescan.org/address/0x43288a1E4f64D561c93401f3B7A5D37248C4907b

If you see ~0.001 ETH or more, you can:

- Use Base Sepolia faucets.
- Deploy and verify contracts on Base Sepolia.

Optional: use https://revoke.cash to manage token approvals and allowances.

---

## 7 — Deploying Smart Contracts on Base Sepolia (Foundry)

Project structure (example):

```
zk_proof_mul_and_sum/
├─ circuits/        → Noir circuit + proofs
├─ contract/        → Solidity verifier + Foundry setup
└─ js/              → Proof generation scripts
```

Goal: Deploy `Verifier.sol` to the Base Sepolia testnet using Foundry.

---

## 8 — Install and Update Foundry

Install Foundry and update:

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

Verify installation:

```bash
forge --version
cast --version
```

Foundry docs: https://book.getfoundry.sh/

---

## 9 — Configure Environment Variables

Export your private key and Base Sepolia RPC URL (official endpoint):

```bash
export RPC="https://sepolia.base.org"
export PK="0xYOUR_PRIVATE_KEY"
export ETHERSCAN_API_KEY="your_basescan_api_key"
```

Note: the RPC URL is a JSON-RPC endpoint (not a web page). You can test it:

```bash
curl -X POST $RPC \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

---

## Config Setup: RPC, PK, and BaseScan API Key

What these values are and where to get them:

- **RPC:** JSON-RPC endpoint for Base Sepolia — public: `https://sepolia.base.org` or provider (Recommended: Alchemy, QuickNode)
- **PK (Private Key):** Used to sign deployments — export from MetaMask (keep secret)
- **ETHERSCAN_API_KEY:** Used for contract verification on BaseScan — create in your BaseScan account

---

## Export Private Key from MetaMask (Handle Carefully)

1. Open MetaMask.
2. Click Account → Account Details → Export Private Key.
3. Confirm the prompt and copy the hex key (`0x...`).
4. Paste into your local `.env` (never commit `.env` to GitHub):

```bash
PK="0xYOUR_PRIVATE_KEY"
```

---

## Get RPC Endpoint — Option A: Public Base Sepolia

- Use the public endpoint: `https://sepolia.base.org`
- No signup required
- May have reliability limits under heavy use

---

## Get RPC Endpoint — Option B: Alchemy (Recommended)

Step 1 — Create an app on Alchemy (https://www.alchemy.com):

- Name: `base-sepolia-dev`
- Chain: Base Sepolia
- Environment: Development
- Use Case and other fields as needed

Step 2 — Choose Chains

- Select **Base Sepolia**
- Optionally also select Ethereum Sepolia

Step 3 — Activate Services

Recommended:

- Node API (JSON-RPC) — ON
- Transfers API — ON
- Transaction Simulation — Optional
- Webhooks — Optional

Step 4 — Copy the Alchemy HTTPS RPC URL:

Example:

```bash
RPC="https://base-sepolia.g.alchemy.com/v2/YOUR_API_KEY"
```

If you see `eth-sepolia`, switch to Base Sepolia and refresh.

---

## Create a BaseScan (Etherscan-style) API Key

1. Visit: https://basescan.org  
2. Sign in or sign up.  
3. Go to Profile → API Keys → Add / Create New API Key.  
4. Name it (e.g., `base-sepolia-dev`) and copy the generated key.

Use it in your `.env` as:

```bash
ETHERSCAN_API_KEY="paste_your_key_here"
```

---

## Obtaining Your Etherscan API Key (Optional Reference)

If you use Etherscan-style APIs elsewhere:

- Visit https://etherscan.io/myapikey to view or create API keys.
- Example key usage:

```
ETHERSCAN_API_KEY=VZFDUWB3YGQ1YCDKTCU1D6DDSS
```

API URL example:

```
https://api.etherscan.io/v2/api?chainid=1&module=account&action=balance&address=0xYourAddress&tag=latest&apikey=VZFDUWB3YGQ1YCDKTCU1D6DDSS
```

---

## Final .env Setup (example)

From your contract directory (e.g., `zk_proof_mul_and_sum/contract`):

```bash
cat > .env <<'EOF'
RPC="https://base-sepolia.g.alchemy.com/v2/97KJ9-rReOpP76CDrTbpC"
PK="0xYOUR_PRIVATE_KEY"
ETHERSCAN_API_KEY="your_basescan_api_key"
EOF

source .env
```

**Do not commit `.env` to version control.**

---

## Test the Connection

Run:

```bash
curl -X POST $RPC \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

Expected output (example):

```json
{"jsonrpc":"2.0","id":1,"result":"0x12a34b"}
```

If you receive a block number result, the connection to Base Sepolia via your RPC provider is successful.

---

## Deploying (next steps)

With Foundry installed and `.env` configured, you can compile and deploy your verifier contract using `forge` commands or a scripted `script/Deploy.s.sol` (see the project README or deploy scripts).

---

## Security and Best Practices

- Never commit private keys or `.env` files to a public repository.
- Use a hardware wallet for production operations.
- Limit test funds on public testnets.
- Use `revoke.cash` to manage approvals and minimize token exposure.

---
