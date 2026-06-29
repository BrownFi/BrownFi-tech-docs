# BrownFiV3 Router Swap Guide

This guide provides step-by-step instructions for performing swaps using the `BrownFiV3Router` contract with Pyth Network price feeds.

BrownFi V3 keeps a router flow similar to V2, but there are several important differences:

- Pools are permissioned and must already exist in the V3 factory.
- Router quotes are inventory-safe and cutoff-aware through `getAmountsOut`, `getAmountsIn`, and `quoteAmountsOutWithUpdate`.
- Pyth update payloads are passed to the router as ABI-encoded `bytes[]`.
- Pyth update fee is always `0`, so swaps can pass update data directly without a separate fee-check step.

## Prerequisites

Before starting, ensure you have:

- Node.js and npm installed
- Hardhat development environment set up
- Access to a deployed `BrownFiV3Router` contract
- Access to token addresses and Pyth feed IDs for the swap path
- Sufficient tokens and native gas token for gas fees

## Required Dependencies

Install the necessary packages:

```bash
npm install @pythnetwork/pyth-evm-js ethers hardhat
```

## Deployed V3 Routers

The V3 periphery stores deployment records in `scripts/data/deployed_routers_v3_chain-<chainId>.json`.

| Chain ID | Network | BrownFiV3Router | Factory | Wrapped native |
| --- | --- | --- | --- | --- |
| 1 | Ethereum | `0x92927Ff9420aF3347Ae25ad618Eb844E78EFe8E1` | `0x0A461D280891167Ee8391f4F0c03EECaa39ae632` | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` |
| 42161 | Arbitrum One | `0x96cE2973581C5bF362e0fc9f40e6B5f12AA59b61` | `0xe49805412EDFDF4C458B297e7C1534588Fa3F1F0` | `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1` |
| 8453 | Base | `0x38c91c64169c7B5eBe02DcE39060B6180065C38d` | `0x5c4B5b07AE5EaeC428C23DbC96564Bb5A3BE7aaa` | `0x4200000000000000000000000000000000000006` |
| 80094 | Berachain | `0x63D8C045ebEc54c4C4bb3e24cA3bf7FD4fFd209a` | `0x6Ccf36d3EaE84b2eB608704070B90f4419BBcD28` | `0x6969696969696969696969696969696969696969` |
| 999 | HyperEVM | `0xc0E55d0085266E9A33456610E08172f9c173F908` | `0x6A4Bd89709b67eC846F02cF9E95A0dd2Fb515720` | `0x5555555555555555555555555555555555555555` |
| 59144 | Linea | `0xB3c31fDc0a22D5725C47B1fC430F5B87353D8C3e` | `0xD9a702839510ee2859bCE697F51Aae49bF8949d7` | `0xe5D7C2a44FfDDf6b295A15c148167daaAf5Cf34f` |

## Step-by-Step Swap Process

### Step 1: Initialize the Environment

```typescript
import hre from "hardhat";
import path from "path";
import * as fs from "fs";
import { AbiCoder } from "ethers";
import { EvmPriceServiceConnection } from "@pythnetwork/pyth-evm-js";

async function main() {
  const [owner] = await hre.ethers.getSigners();
  const wallet: any = owner;
  console.log(`Account: ${owner.address}`);
```

### Step 2: Get Network Information

```typescript
  const network = await hre.ethers.provider.getNetwork();
  const chainId = network.chainId;
  console.log(`Current chain ID: ${chainId}`);
```

### Step 3: Load Configuration Parameters

```typescript
  const parametersPath = path.join(__dirname, `../ignition/parameters-${chainId}.json`);
  const parameters = JSON.parse(fs.readFileSync(parametersPath, "utf8"));
  const routerParams = parameters.BrownFiV3RouterModule;

  const deployedRoutersPath = path.join(
    __dirname,
    `../scripts/data/deployed_routers_v3_chain-${chainId}.json`
  );

  if (!fs.existsSync(deployedRoutersPath)) {
    throw new Error(`Deployment file not found for chain ID ${chainId}`);
  }

  const deployedRouters = JSON.parse(fs.readFileSync(deployedRoutersPath, "utf8"));
```

### Step 4: Resolve Token and Router Addresses

```typescript
  function getWrappedNativeByChainId(chainId: bigint): string {
    if (chainId === 1329n) return routerParams.wsei;
    if (chainId === 143n) return routerParams.wmonad;
    if (chainId === 999n) return routerParams.hype;
    if (chainId === 80094n) return routerParams.wbera;
    return routerParams.weth;
  }

  const wrappedNative = getWrappedNativeByChainId(chainId);

  // Example: USDT -> HYPE on HyperEVM, or replace these keys with your path.
  const inputTokenAddress = routerParams.usdt;
  const outputTokenAddress = wrappedNative;
  const pathAddresses = [inputTokenAddress, outputTokenAddress];

  const routerAddress = deployedRouters.brownFiV3Router;
  const router = await hre.ethers.getContractAt("BrownFiV3Router", routerAddress, wallet);
```

### Step 5: Setup Pyth Price Feed IDs

```typescript
  const priceFeedIdsPath = path.join(__dirname, "../scripts/data/PriceFeedIds.json");
  const priceFeedIdsData = JSON.parse(fs.readFileSync(priceFeedIdsPath, "utf8"));

  const ids = [
    priceFeedIdsData.PriceFeedIds.usdt, // USDT/USD
    priceFeedIdsData.PriceFeedIds.hype, // HYPE/USD, replace for your output token
  ];
```

Common V3 feed keys include `eth`, `usdc`, `usdt`, `bera`, `hype`, `khype`, `btc`, `linea`, `bnb`, `monad`, `honey`, `ibgt`, `usde`, and `dolo`.

### Step 6: Get Price Feed Update Data from Pyth

```typescript
  const connection = new EvmPriceServiceConnection("https://hermes.pyth.network");
  const priceFeedUpdateData = await connection.getPriceFeedsUpdateData(ids);

  const encodedUpdateData = AbiCoder.defaultAbiCoder().encode(
    ["bytes[]"],
    [priceFeedUpdateData]
  );
```

If you want to use prices already stored on-chain, pass `"0x"` instead of encoded update data.

Since Pyth update fee is always `0`, no update-fee check is required.

### Step 7: Approve Token Spending

```typescript
  const erc20Abi = [
    "function decimals() view returns (uint8)",
    "function symbol() view returns (string)",
    "function allowance(address owner, address spender) view returns (uint256)",
    "function approve(address spender, uint256 amount) returns (bool)",
  ];

  const inputToken = new hre.ethers.Contract(inputTokenAddress, erc20Abi, wallet);
  const inputDecimals = await inputToken.decimals();
  const amountIn = hre.ethers.parseUnits("0.1", inputDecimals);

  const allowance = await inputToken.allowance(owner.address, routerAddress);
  if (allowance < amountIn) {
    const txApprove = await inputToken.approve(routerAddress, amountIn);
    await txApprove.wait();
    console.log(`Approved input token: ${txApprove.hash}`);
  }
```

### Step 8: Get an Executable Quote

```typescript
  const quotedAmountsOut = await router.quoteAmountsOutWithUpdate.staticCall(
    amountIn,
    pathAddresses,
    encodedUpdateData
  );

  const quotedOut = quotedAmountsOut[quotedAmountsOut.length - 1] as bigint;
  const slippageBps = 50n; // 0.50%
  const amountOutMin = (quotedOut * (10_000n - slippageBps)) / 10_000n;
```

For display-only frontend quotes, V3 also exposes raw helpers such as `getAmountsOutWithoutCutoff`, but swaps execute against the cutoff-aware quote path. Use `getAmountsOut` or `quoteAmountsOutWithUpdate` for slippage checks.

### Step 9: Execute the Swap

#### Token to Token

```typescript
  const deadline = Math.floor(Date.now() / 1000) + 60 * 20;

  await router.swapExactTokensForTokens.staticCall(
    amountIn,
    amountOutMin,
    pathAddresses,
    owner.address,
    deadline,
    encodedUpdateData
  );

  const txSwap = await router.swapExactTokensForTokens(
    amountIn,
    amountOutMin,
    pathAddresses,
    owner.address,
    deadline,
    encodedUpdateData
  );

  await txSwap.wait();
  console.log(`Swap transaction hash: ${txSwap.hash}`);
```

#### Token to Native

Use the wrapped native token address as the final path token.

```typescript
  const txSwap = await router.swapExactTokensForETH(
    amountIn,
    amountOutMin,
    [inputTokenAddress, wrappedNative],
    owner.address,
    deadline,
    encodedUpdateData
  );
```

#### Native to Token

Use the wrapped native token address as the first path token.

```typescript
  const txSwap = await router.swapExactETHForTokens(
    amountOutMin,
    [wrappedNative, outputTokenAddress],
    owner.address,
    deadline,
    encodedUpdateData,
    { value: amountIn }
  );
```

## Example: Complete Swap Script

```typescript
import hre from "hardhat";
import path from "path";
import * as fs from "fs";
import { AbiCoder } from "ethers";
import { EvmPriceServiceConnection } from "@pythnetwork/pyth-evm-js";

const PATH_TOKEN_KEYS = ["usdt", "hype"] as const;
const AMOUNT_IN = "0.1";
const SLIPPAGE_BPS = 50n;

function mapTokenKeyToPriceFeedKey(tokenKey: string): string {
  const normalized = tokenKey.toLowerCase();
  if (normalized === "native") return "eth";
  if (normalized === "weth") return "eth";
  if (normalized === "wbera") return "bera";
  if (normalized === "wbtc") return "btc";
  return normalized;
}

async function executeSwap() {
  const [owner] = await hre.ethers.getSigners();
  const network = await hre.ethers.provider.getNetwork();
  const chainId = network.chainId;

  const paramsPath = path.join(__dirname, `../ignition/parameters-${chainId}.json`);
  const params = JSON.parse(fs.readFileSync(paramsPath, "utf8"));
  const routerParams = params.BrownFiV3RouterModule;

  const deployedRoutersPath = path.join(
    __dirname,
    `../scripts/data/deployed_routers_v3_chain-${chainId}.json`
  );
  const deployedRouters = JSON.parse(fs.readFileSync(deployedRoutersPath, "utf8"));
  const routerAddress = deployedRouters.brownFiV3Router;
  const router = await hre.ethers.getContractAt("BrownFiV3Router", routerAddress, owner);

  const wrappedNative =
    chainId === 999n ? routerParams.hype :
    chainId === 80094n ? routerParams.wbera :
    chainId === 143n ? routerParams.wmonad :
    chainId === 1329n ? routerParams.wsei :
    routerParams.weth;

  const pathAddresses = PATH_TOKEN_KEYS.map((key) => {
    if (routerParams[key]) return routerParams[key];
    if (key === "hype" || key === "native") return wrappedNative;
    throw new Error(`Missing token address for ${key}`);
  });

  const inputTokenAddress = pathAddresses[0];
  const outputTokenAddress = pathAddresses[pathAddresses.length - 1];
  const isNativeIn = inputTokenAddress.toLowerCase() === wrappedNative.toLowerCase();
  const isNativeOut = outputTokenAddress.toLowerCase() === wrappedNative.toLowerCase();

  const erc20Abi = [
    "function decimals() view returns (uint8)",
    "function allowance(address owner, address spender) view returns (uint256)",
    "function approve(address spender, uint256 amount) returns (bool)",
  ];
  const inputToken = new hre.ethers.Contract(inputTokenAddress, erc20Abi, owner);
  const inputDecimals = await inputToken.decimals();
  const amountIn = hre.ethers.parseUnits(AMOUNT_IN, inputDecimals);

  const priceFeedIdsPath = path.join(__dirname, "../scripts/data/PriceFeedIds.json");
  const priceFeedIdsData = JSON.parse(fs.readFileSync(priceFeedIdsPath, "utf8"));
  const feedIds = [...new Set(PATH_TOKEN_KEYS.map(mapTokenKeyToPriceFeedKey))]
    .map((key) => priceFeedIdsData.PriceFeedIds[key]);

  const connection = new EvmPriceServiceConnection("https://hermes.pyth.network");
  const updateData = await connection.getPriceFeedsUpdateData(feedIds);
  const encodedUpdateData = AbiCoder.defaultAbiCoder().encode(["bytes[]"], [updateData]);

  const quotedAmountsOut = await router.quoteAmountsOutWithUpdate.staticCall(
    amountIn,
    pathAddresses,
    encodedUpdateData
  );
  const quotedOut = quotedAmountsOut[quotedAmountsOut.length - 1] as bigint;
  const amountOutMin = (quotedOut * (10_000n - SLIPPAGE_BPS)) / 10_000n;

  if (!isNativeIn) {
    const allowance = await inputToken.allowance(owner.address, routerAddress);
    if (allowance < amountIn) {
      await (await inputToken.approve(routerAddress, amountIn)).wait();
    }
  }

  const deadline = Math.floor(Date.now() / 1000) + 60 * 20;
  let tx;

  if (isNativeIn) {
    tx = await router.swapExactETHForTokens(
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      encodedUpdateData,
      { value: amountIn }
    );
  } else if (isNativeOut) {
    tx = await router.swapExactTokensForETH(
      amountIn,
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      encodedUpdateData
    );
  } else {
    tx = await router.swapExactTokensForTokens(
      amountIn,
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      encodedUpdateData
    );
  }

  await tx.wait();
  console.log(`Swap completed: ${tx.hash}`);
}

executeSwap().catch((error) => {
  console.error(error?.shortMessage ?? error?.message ?? error);
  process.exitCode = 1;
});
```

## Example: Run Swap Script

```bash
npx hardhat run scripts/SwapV3.ts --network hyperevm
```

This guide provides a practical overview of using `BrownFiV3Router` with Pyth price feeds. Because Pyth update fee is always `0`, the same encoded update data can be used for token-token and native-token swap paths.
