# BrownFiV2 Router Swap Guide

This guide provides step-by-step instructions for performing token swaps using the BrownFiV2Router contract with Pyth Network price feeds.

## Prerequisites

Before starting, ensure you have:
- Node.js and npm installed
- Hardhat development environment set up
- Access to a deployed BrownFiV2Router contract
- Sufficient tokens and ETH for gas fees

## Required Dependencies

Install the necessary packages:

```bash
npm install @pythnetwork/pyth-evm-js ethers hardhat
```

## Step-by-Step Swap Process

### Step 1: Initialize the Environment

```typescript
import hre from "hardhat";
import { AbiCoder } from "ethers";
import path from "path";
import * as fs from "fs";
import { EvmPriceServiceConnection } from "@pythnetwork/pyth-evm-js"

async function main() {
  // Get signers (wallet accounts)
  const [owner, tester] = await hre.ethers.getSigners();
  const wallet: any = owner;
  console.log(`Account: ${owner.address}`);
```

### Step 2: Get Network Information

```typescript
  // Get the current chain ID
  const network = await hre.ethers.provider.getNetwork();
  const chainId = network.chainId;
  console.log(`Current chain ID: ${chainId}`);
```

### Step 3: Load Configuration Parameters

```typescript
  // Read parameters from JSON file
  const parametersPath = path.join(__dirname, `../ignition/parameters-${chainId}.json`);
  const parameters = JSON.parse(fs.readFileSync(parametersPath, 'utf8'));
  
  // Get deploymentAddresses from ignition deployments using chain ID
  const deploymentPath = path.join(__dirname, `../ignition/deployments/chain-${chainId}/deployed_addresses.json`);
  
  if (!fs.existsSync(deploymentPath)) {
    throw new Error(`Deployment file not found for chain ID ${chainId} at ${deploymentPath}`);
  }
  
  const deploymentAddresses = JSON.parse(fs.readFileSync(deploymentPath, 'utf8'));
```

### Step 4: Get Contract Addresses and Instances

```typescript
  // Get token addresses from parameters
  const WETHAddress = parameters.BrownFiV2RouterModule.weth;
  const USDCAddress = parameters.BrownFiV2RouterModule.usdc;
  
  // Get contract instances
  const USDC = await hre.ethers.getContractAt("@openzeppelin/contracts/token/ERC20/IERC20.sol:IERC20", USDCAddress, wallet);
  const routerAddress = deploymentAddresses["BrownFiV2RouterModule#BrownFiV2Router"];
  const router = await hre.ethers.getContractAt("BrownFiV2Router", routerAddress, wallet);
```

### Step 5: Setup Pyth Price Feed IDs

```typescript
  // Define price feed IDs for the tokens you want to swap
  // You can find these IDs at https://pyth.network/developers/price-feed-ids
  const idsEthUsdc = [
    "0xff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace", // ETH/USD
    "0xeaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a", // USDC/USD
  ];
```

### Step 6: Get Price Feed Update Data from Pyth

```typescript
  // Initialize connection to Pyth Network
  const connection = new EvmPriceServiceConnection(
      "https://hermes.pyth.network"
  );

  // Get price feed update data
  const priceFeedUpdateDataEthUsdc = await connection.getPriceFeedsUpdateData(
      idsEthUsdc
  );
```

### Step 7: Approve Token Spending

```typescript
  // Approve the router to spend your tokens
  await USDC.approve(router.getAddress(), hre.ethers.parseUnits("1000", 6));
```

### Step 8: Prepare Update Data

```typescript
  // Encode price feed update data
  const abiCoder = AbiCoder.defaultAbiCoder();
  const encodedUpdateData = abiCoder.encode(
    ['bytes[]'],
    [priceFeedUpdateDataEthUsdc]
  );
```

### Step 9: Execute the Swap

```typescript
  // Execute the swap transaction
  const txSwap = await router.swapExactTokensForETH(
    hre.ethers.parseUnits("0.1", 6), // amountIn (0.1 USDC)
    hre.ethers.parseUnits("0.09", 6), // amountOutMin (minimum 0.09 ETH expected)
    [USDCAddress, WETHAddress], // path (USDC -> WETH)
    owner.address, // to (recipient address)
    Math.floor(Date.now() / 1000) + 60 * 20, // deadline (20 minutes from now)
    encodedUpdateData, // price feed update data
  );

  console.log(`Swap transaction hash: ${txSwap.hash}`);
```

## Example: Complete Swap Script

```typescript
import hre from "hardhat";
import { AbiCoder } from "ethers";
import { EvmPriceServiceConnection } from "@pythnetwork/pyth-evm-js";

async function executeSwap() {
  const [owner] = await hre.ethers.getSigners();
  
  // Setup contracts and parameters...
  
  // Get price data
  const connection = new EvmPriceServiceConnection("https://hermes.pyth.network");
  const priceData = await connection.getPriceFeedsUpdateData(priceIds);
  
  // Approve tokens
  await token.approve(routerAddress, amount);
  
  // Execute swap
  const tx = await router.swapExactTokensForETH(
    amount,
    minAmountOut,
    path,
    recipient,
    deadline,
    AbiCoder.defaultAbiCoder().encode(['bytes[]'], [priceData])
  );
  
  await tx.wait();
  console.log("Swap completed successfully!");
}
```

## Examle: Run Swap script

```
npx hardhat run scripts/Swap.ts --network arbitrumOne
```

This guide provides a comprehensive overview of how to perform swaps using the BrownFiV2Router with Pyth Network price feeds for accurate and up-to-date pricing information.