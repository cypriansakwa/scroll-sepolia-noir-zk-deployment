# SCROLL + Noir ZK Deployment Tutorial
Deploying and Interacting with Zero-Knowledge Verifier Contracts on Scroll Sepolia

## Lesson Overview

**Objective**

By the end of this lesson, learners will be able to:

- Explain what Scroll is and how it relates to Ethereum.
- Set up a Scroll Sepolia development environment.
- Deploy a zero-knowledge verifier contract on Scroll Sepolia.
- Verify and interact with the deployed contract using Foundry scripts.
- Understand the difference between read-only verification and on-chain verification.

## What is Scroll?

### Scroll in Plain Terms
Scroll is an Ethereum Layer 2 designed to make transactions faster and cheaper.

It uses zero-knowledge proofs to validate batches of transactions on Ethereum.

It is a zkEVM, meaning it fully supports Ethereum smart contracts.

Any Ethereum tool (Hardhat, Foundry, MetaMask) works on Scroll — you simply change the RPC URL.

### Reference
https://docs.scroll.io/en/developers/

## Adding Scroll Sepolia to MetaMask

Open MetaMask → Network Dropdown → Add Network → Add a network manually.

Fill in:
```yaml
Network Name: Scroll Sepolia
RPC URL: https://sepolia-rpc.scroll.io/
Chain ID: 534351
Currency Symbol: ETH
Block Explorer: https://sepolia.scrollscan.com
```


Save and switch to Scroll Sepolia.

## Funding Scroll Sepolia (Test ETH)

### Using the Telegram Faucet

1. Open Telegram and search: `@scroll_up_sepolia_bot`
2. Click Start
3. Join the faucet group: https://t.me/scroll_sepolia_faucet
4. Request funds:

```bash
 /drop 0xYourEthereumAddress

```

You should receive 0.05 ETH within seconds.

## Export Your Private Key (Handle Carefully)

MetaMask → Account → Account Details → Export Private Key

Copy the key and save it in your `.env` file:

```bash
 PRIVATE_KEY=0xYOUR_PRIVATE_KEY
```


Never share or commit this file.  
Add it to `.gitignore`.

## Getting Your Scroll RPC URL via Alchemy

Visit https://www.alchemy.com

Create App → Choose Chain: Scroll Sepolia

Recommended service configuration:

| Service | Purpose | Recommended |
|---------|---------|-------------|
| Node API | Main JSON-RPC endpoint | ON |
| Transfers API | Token and ETH movement | ON |
| Transaction Simulation | Gas and revert testing | Optional |
| Webhooks | Contract notifications | Optional |

Then copy your RPC URL:

```ini
SCROLL_SEPOLIA_RPC_URL="https://scroll-sepolia.g.alchemy.com/v2/YOUR_API_KEY"

```

## Etherscan API Key (ScrollScan Verification)

Purpose: For contract verification on ScrollScan.

Go to: https://etherscan.io/myapikey  
Add a new key.

Save it into `.env`:

```ini
ETHERSCAN_API_KEY=YOUR_KEY
```

## Environment Setup Summary

Your `.env` file should contain:

```ini
PRIVATE_KEY=0x10a5cdc835304675db...
SCROLL_SEPOLIA_RPC_URL=https://scroll-sepolia.g.alchemy.com/v2/XXXXX
CHAIN_ID=534351
ETHERSCAN_API_KEY=XXXXX

```

## Manual Deployment Using Forge

Activate the environment:

```bash
 source .env
```

Deploy:

```lua
 forge create Verifier.sol:HonkVerifier \
  --rpc-url $SCROLL_SEPOLIA_RPC_URL \
  --private-key $PRIVATE_KEY \
  --etherscan-api-key $ETHERSCAN_API_KEY \
  --verify \
  --broadcast
```
### Fixing Contract Size Errors

If you see:
```nginx
 EVM error: CreateContractSizeLimit
```
Add to `foundry.toml`:
```ini
optimizer = true
optimizer_runs = 200
```
**Foundry Configuration (`foundry.toml`)**
```toml
  [profile.default]
src = "."
out = "out"
libs = ["lib"]
ffi = true
optimizer = true
optimizer_runs = 200

[rpc_endpoints]
scroll-sepolia = "${SCROLL_SEPOLIA_RPC_URL}"

[etherscan]
scroll-sepolia = { key = "${ETHERSCAN_API_KEY}", chain = 534351 }
```
## Deployment Script (Recommended)
Create:

`script/Deploy.s.sol`
 
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

Run:

```bash
 forge script script/Deploy.s.sol \
  --rpc-url $SCROLL_SEPOLIA_RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast \
  --verify \
  --etherscan-api-key $ETHERSCAN_API_KEY
```
### Deployment Summary
```yaml
Script Execution: Success
Deployed Contract: 0xdba076765f2F6e26B672BCBb935c5C6fB5b6350D
Transaction Hash: 0xc0a4a779b9045c0de90788cfc71af5c499aed3710157424fb7da34723b7af845
Gas Used: 4,719,156
Cost: ~0.000074 ETH
Compiler: v0.8.30, optimized
```
View contract:
```bash
 https://sepolia.scrollscan.com/address/0xdba076765f2F6e26B672BCBb935c5C6fB5b6350D#code
```
## Interacting with the Deployed Contract
Create: `script/Interact.s.sol`
```solidity
  // SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.17;

import {Script, console} from "forge-std/Script.sol";
import {HonkVerifier} from "../Verifier.sol";

contract InteractHonkVerifier is Script {
    address constant VERIFIER_ADDRESS =
        0xdba076765f2F6e26B672BCBb935c5C6fB5b6350D;

    function run() external {
        vm.startBroadcast();

        HonkVerifier verifier = HonkVerifier(VERIFIER_ADDRESS);

        bytes memory proof = vm.readFileBinary("../circuits/target/proof");

        bytes32;
        publicInputs[0] = bytes32(uint256(13));

        bool result = verifier.verify(proof, publicInputs);

        console.log("Verification result:", result);
    }
}
```
## On-Chain Storage Interaction
Create: `script/StoreInteraction.s.sol`
```solidity
  // SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

import "forge-std/Script.sol";

interface IHonkVerifierWithRecord {
    function verifyAndStore(bytes calldata proof, bytes32[] calldata publicInputs)
        external
        returns (bool);
}

contract StoreInteraction is Script {
    function run() external {
        uint256 key = vm.envUint("PRIVATE_KEY");

        vm.startBroadcast(key);

        address verifierAddress =
            0xdba076765f2F6e26B672BCBb935c5C6fB5b6350D;

        bytes memory proof = vm.readFileBinary("../circuits/target/proof");

        bytes32;
        publicInputs[0] = bytes32(uint256(13));

        bool result = IHonkVerifierWithRecord(verifierAddress)
            .verifyAndStore(proof, publicInputs);

        console.log("On-chain verification result:", result);

        vm.stopBroadcast();
    }
}

```
### Comparison of Interaction Methods
| Feature              | InteractHonkVerifier   | StoreInteraction           |
| -------------------- | ---------------------- | -------------------------- |
| Function Called      | verify()               | verifyAndStore()           |
| Call Type            | Read-only (staticcall) | Transaction                |
| Gas Cost             | No                     | Yes                        |
| Stores Data On-chain | No                     | Yes                        |
| Purpose              | Proof validation       | Persistent on-chain record |

## Lesson Summary

You learned:

- What Scroll is and why zkEVM matters  
- How to configure MetaMask and RPCs  
- How to deploy and verify smart contracts using Foundry  
- How to interact with zk-verifier contracts  
- The difference between off-chain vs on-chain verification  

## Contact & Credits

**Instructor:**  
Dr. Cyprian Omukhwaya Sakwa  
Cryptography Instructor, Web3Clubs Foundation

**Email:** cypriansakwa@gmail.com  
**Twitter:** https://twitter.com/cypriansakwaOm  
**GitHub:** https://github.com/cypriansakwa

Use testnets responsibly.
