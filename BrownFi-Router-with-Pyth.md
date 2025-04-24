# Guide: Creating Transactions with BrownFi Router and Pyth Price Feeds

This step-by-step guide explains how to create transactions using the BrownFi Router with Pyth price feeds.

## Prerequisites

- Node.js and npm installed
- Hardhat project set up
- Contract addresses stored in a JSON file
- Private key configured in your environment

## Step 1: Set Up the Environment

```javascript
const { ethers } = require("hardhat")
const { EvmPriceServiceConnection } = require("@pythnetwork/pyth-evm-js")
const fs = require('fs')
const path = require('path')
```

## Step 2: Load Contract Addresses

```javascript
// Load addresses from JSON file
const addressPath = path.join(__dirname, `./json/${process.env.addr}.json`)
const addressList = JSON.parse(fs.readFileSync(addressPath, 'utf8'))
```

## Step 3: Connect to Network

```javascript
// Get network configuration from Hardhat
const url = hre.network.config.url
const account = hre.network.config.accounts[0]

// Set up provider and signer
const provider = new ethers.providers.JsonRpcProvider(url)
const deployer = new ethers.Wallet(account, provider)

// Verify connection
console.log('Connected address:', deployer.address)
console.log('Balance:', await provider.getBalance(deployer.address))
```

## Step 4: Fetch Pyth Price Feed Data

```javascript
// Define price feed IDs you want to fetch
const ids = [
    // ETH/USD price feed ID
    "0x7d80a0d7344c6632c5ed2b85016f32aed4f831294e274739d92bb9e32df5b22f",
    // Add other price feed IDs as needed
];

// Create connection to Pyth price service
const connection = new EvmPriceServiceConnection("https://hermes.pyth.network");

// Get price feed update data
const priceFeedUpdateData = await connection.getPriceFeedsUpdateData(ids);
console.log("Retrieved Pyth price update data");
```

## Step 5: Calculate Update Fee

```javascript
// Create contract instance for Pyth
const pythABI = [
    {
        inputs: [{ internalType: "bytes[]", name: "updateData", type: "bytes[]" }],
        name: "getUpdateFee",
        outputs: [{ internalType: "uint256", name: "feeAmount", type: "uint256" }],
        stateMutability: "view",
        type: "function"
    },
    {
        inputs: [{ internalType: "bytes[]", name: "updateData", type: "bytes[]" }],
        name: "updatePriceFeeds",
        outputs: [],
        stateMutability: "payable",
        type: "function"
    }
];

const pythContract = new ethers.Contract(addressList["PYTH"], pythABI, provider);
const updateFee = await pythContract.getUpdateFee(priceFeedUpdateData);
console.log("Update Fee:", updateFee.toString());
```

## Step 6: Create Router Contract Instance

```javascript
// Load Router ABI from artifacts
const brownFiRouterWithPriceABI = require("../artifacts/contracts/BrownFiV1Router03.sol/BrownFiV1Router03.json").abi

// Create Router contract instance
const brownFiRouterWithPrice = new ethers.Contract(
    addressList["BrownFiV1RouterWithPrice"], 
    brownFiRouterWithPriceABI, 
    provider
)
```

## Step 7: Get Price Quotes

```javascript
// Get amounts out (how much you receive for a given input)
const amountsOut = await brownFiRouterWithPrice.connect(deployer).callStatic.getAmountsOutWithPrice(
    ethers.utils.parseEther("1"),  // Amount in (1 token with 18 decimals)
    [addressList["USDC"], addressList["WETH"]],  // Path: USDC -> WETH
    priceFeedUpdateData,  // Pyth price data
    {
        value: updateFee.toString(),  // Pay for Pyth update
    }
)
console.log('Expected output amounts:', amountsOut.map(a => a.toString()))

// Get amounts in (how much you need to input for a desired output)
const amountsIn = await brownFiRouterWithPrice.connect(deployer).callStatic.getAmountsInWithPrice(
    "40000000000000000",  // Amount out (0.04 ETH)
    [addressList["USDC"], addressList["WETH"]],  // Path: USDC -> WETH
    priceFeedUpdateData,  // Pyth price data
    {
        value: updateFee.toString(),  // Pay for Pyth update
    }
)
console.log('Required input amounts:', amountsIn.map(a => a.toString()))
```

## Step 8: Approve Tokens (if swapping tokens)

```javascript
// Load token ABI
const tokenABI = require("../artifacts/contracts/test/BrownFiERC20.sol/BrownFiERC20.json").abi

// Create token instance
const token = new ethers.Contract(addressList["TOKEN_ADDRESS"], tokenABI, provider)

// Approve router to spend tokens
const txApprove = await token.connect(deployer).approve(
    addressList["BrownFiV1RouterWithPrice"], 
    ethers.constants.MaxUint256  // Infinite approval
)
console.log('Approval transaction:', txApprove.hash)
await txApprove.wait()  // Wait for approval transaction to complete
```

## Step 9: Execute Swap Transaction

### Example 1: Swap Exact Tokens for Tokens

```javascript
const amountIn = ethers.utils.parseEther("1")  // 1 token
const amountOutMin = "0"  // Accept any output amount (set higher in production)
const path = [addressList["TOKEN_A"], addressList["TOKEN_B"]]
const to = deployer.address  // Recipient address
const deadline = Math.floor(Date.now() / 1000) + 3600  // 1 hour from now

const txSwap = await brownFiRouterWithPrice.connect(deployer).swapExactTokensForTokensWithPrice(
    amountIn,
    amountOutMin,
    path,
    to,
    deadline,
    priceFeedUpdateData,
    {
        value: updateFee.toString(),  // Pay for Pyth update
    }
)
console.log('Swap transaction hash:', txSwap.hash)
```

### Example 2: Swap Tokens for Exact ETH

```javascript
const amountOut = ethers.utils.parseUnits("0.1", 18)  // 0.1 ETH
const amountInMax = ethers.utils.parseEther("1000")  // Maximum 1000 tokens to spend
const path = [addressList["TOKEN"], addressList["WETH"]]
const to = deployer.address
const deadline = Math.floor(Date.now() / 1000) + 3600  // 1 hour from now

const txSwap = await brownFiRouterWithPrice.connect(deployer).swapTokensForExactETHWithPrice(
    amountOut,
    amountInMax,
    path,
    to,
    deadline,
    priceFeedUpdateData,
    {
        value: updateFee.toString(),
    }
)
console.log('Swap transaction hash:', txSwap.hash)
```

### Example 3: Swap ETH for Exact Tokens

```javascript
const amountOut = "1000000"  // 1 USDC (with 6 decimals)
const path = [addressList["WETH"], addressList["USDC"]]
const to = deployer.address
const deadline = Math.floor(Date.now() / 1000) + 3600

// Calculate how much ETH needed plus update fee
const amountsIn = await brownFiRouterWithPrice.getAmountsInWithPrice(
    amountOut,
    path,
    priceFeedUpdateData,
    { value: updateFee }
)
const ethNeeded = amountsIn[0]
const totalValue = ethNeeded.add(updateFee)

const txSwap = await brownFiRouterWithPrice.connect(deployer).swapETHForExactTokensWithPrice(
    amountOut,
    path,
    to,
    deadline,
    priceFeedUpdateData,
    {
        value: totalValue.toString(),
    }
)
console.log('Swap transaction hash:', txSwap.hash)
```

## Step 10: Wait for Transaction Confirmation

```javascript
// Wait for the transaction receipt
const receipt = await txSwap.wait()
console.log('Transaction confirmed in block:', receipt.blockNumber)
```

This guide outlines the key steps for creating transactions with the BrownFi Router using Pyth price feeds.
