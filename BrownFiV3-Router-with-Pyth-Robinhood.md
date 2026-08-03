# BrownFiV3 Router Swap Guide for Robinhood

This guide provides step-by-step instructions for performing swaps on Robinhood using the `BrownFiV3Router` contract with Pyth price updates fetched through the BrownFi Pyth proxy.

BrownFi V3 keeps the same swap surface as the generic router guide, but the Robinhood integration should use:

- Robinhood chain ID `4663`
- BrownFi V3 router `0x2A08Da7B6590ce5D217161F234069CfC54DBe554`
- BrownFi Pyth proxy `https://api.brownfi.io/pyth/`
- Pyth update fee `0`, so no update-fee check is required

## Pyth Core Upgrade Note

Pyth Core is upgrading its Hermes infrastructure. Pyth's [EVM upgrade notes](https://docs.pyth.network/price-feeds/core/upgrade/preparing/evm) show the upgraded Hermes endpoint as `https://pyth.dourolabs.app/hermes` and use API-key authentication for direct Hermes calls.

BrownFi exposes `https://api.brownfi.io/pyth/` as a proxy in front of the upgraded Hermes endpoint. The proxy keeps the Hermes REST route shape, so callers can fetch price update payloads without passing a Pyth API key in frontend or script code.

For example, fetch latest price updates from:

```text
https://api.brownfi.io/pyth/v2/updates/price/latest?ids[]=<priceFeedId>
```

The response includes `binary.data`, which should be converted to `0x`-prefixed bytes and ABI-encoded as `bytes[]` before calling the router.

## Robinhood Deployment

| Item | Value |
| --- | --- |
| Network | Robinhood |
| Chain ID | `4663` |
| RPC URL | `https://rpc.mainnet.chain.robinhood.com/` |
| Explorer | `https://robinhoodchain.blockscout.com` |
| Factory | `0x831880Bd3b331249DF63bacC6e21495e5e8f1eAA` |
| BrownFiV3Router | `0x2A08Da7B6590ce5D217161F234069CfC54DBe554` |
| BrownFiV3Zap | `0x92927Ff9420aF3347Ae25ad618Eb844E78EFe8E1` |
| Wrapped native | `0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73` |
| USDG | `0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168` |

## Prerequisites

Before starting, ensure you have:

- Node.js `>=18`
- Hardhat development environment set up
- `PRIVATE_KEY` configured for Robinhood transactions
- `ROBINHOOD_RPC_URL` configured, or use the default public RPC
- Sufficient Robinhood native gas token
- Sufficient input token balance

## Required Dependencies

Install the necessary packages:

```bash
npm install ethers hardhat
```

The examples below use Node's built-in `fetch`, so no Pyth SDK dependency is required.

## Step-by-Step Swap Process

### Step 1: Initialize the Environment

```typescript
import hre from "hardhat";
import { AbiCoder } from "ethers";

const ROBINHOOD_CHAIN_ID = 4663n;
const BROWNFI_PYTH_PROXY = "https://api.brownfi.io/pyth";

const ROUTER_ADDRESS = "0x2A08Da7B6590ce5D217161F234069CfC54DBe554";
const WRAPPED_NATIVE = "0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73";
const USDG = "0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168";

const PRICE_FEED_IDS = {
  eth: "0xff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace",
  usdg: "0xdaa58c6a3ce7d4b9c46c32a6e646012c17c4a2b24c08dd8c5e476118b855a7da",
};

async function main() {
  const [owner] = await hre.ethers.getSigners();
  console.log(`Account: ${owner.address}`);
```

### Step 2: Validate Robinhood Network

```typescript
  const network = await hre.ethers.provider.getNetwork();
  const chainId = network.chainId;
  console.log(`Current chain ID: ${chainId}`);

  if (chainId !== ROBINHOOD_CHAIN_ID) {
    throw new Error(`Expected Robinhood chain ID ${ROBINHOOD_CHAIN_ID}, got ${chainId}`);
  }
```

### Step 3: Create Router and Token Instances

```typescript
  const router = await hre.ethers.getContractAt(
    "BrownFiV3Router",
    ROUTER_ADDRESS,
    owner
  );

  const erc20Abi = [
    "function decimals() view returns (uint8)",
    "function symbol() view returns (string)",
    "function allowance(address owner, address spender) view returns (uint256)",
    "function approve(address spender, uint256 amount) returns (bool)",
  ];

  const inputTokenAddress = USDG;
  const outputTokenAddress = WRAPPED_NATIVE;
  const pathAddresses = [inputTokenAddress, outputTokenAddress];

  const inputToken = new hre.ethers.Contract(inputTokenAddress, erc20Abi, owner);
  const [inputDecimals, inputSymbol] = await Promise.all([
    inputToken.decimals(),
    inputToken.symbol(),
  ]);
```

### Step 4: Fetch Pyth Update Data Through BrownFi Proxy

```typescript
type BrownFiPythProxyResponse = {
  binary?: {
    data?: string[];
    encoding?: string;
  };
};

async function getBrownFiPythUpdateData(priceFeedIds: string[]): Promise<string[]> {
  const url = new URL(`${BROWNFI_PYTH_PROXY}/v2/updates/price/latest`);

  for (const feedId of priceFeedIds) {
    url.searchParams.append("ids[]", feedId);
  }

  const response = await fetch(url);
  if (!response.ok) {
    const body = await response.text();
    throw new Error(`BrownFi Pyth proxy failed: ${response.status} ${body}`);
  }

  const payload = (await response.json()) as BrownFiPythProxyResponse;
  const data = payload.binary?.data;

  if (!data || data.length === 0) {
    throw new Error("BrownFi Pyth proxy returned no binary update data");
  }

  return data.map((item) => (item.startsWith("0x") ? item : `0x${item}`));
}
```

Use the price feeds for every token in the swap path. For the wrapped native token on Robinhood, use the ETH/USD feed.

```typescript
  const priceFeedUpdateData = await getBrownFiPythUpdateData([
    PRICE_FEED_IDS.usdg,
    PRICE_FEED_IDS.eth,
  ]);

  const encodedUpdateData = AbiCoder.defaultAbiCoder().encode(
    ["bytes[]"],
    [priceFeedUpdateData]
  );
```

Since Pyth update fee is always `0`, no `getUpdateFee` call is required.

### Step 5: Approve Token Spending

```typescript
  const amountIn = hre.ethers.parseUnits("10", inputDecimals);

  const allowance = await inputToken.allowance(owner.address, ROUTER_ADDRESS);
  if (allowance < amountIn) {
    const txApprove = await inputToken.approve(ROUTER_ADDRESS, amountIn);
    await txApprove.wait();
    console.log(`Approved ${inputSymbol}: ${txApprove.hash}`);
  }
```

### Step 6: Get an Executable Quote

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

`quoteAmountsOutWithUpdate` updates Pyth prices inside the simulated call and then returns the same cutoff-aware quote path used by swaps.

### Step 7: Execute the Swap

#### USDG to Native

Use the wrapped native token address as the final path token, then call `swapExactTokensForETH`.

```typescript
  const deadline = Math.floor(Date.now() / 1000) + 60 * 20;

  await router.swapExactTokensForETH.staticCall(
    amountIn,
    amountOutMin,
    [USDG, WRAPPED_NATIVE],
    owner.address,
    deadline,
    encodedUpdateData
  );

  const txSwap = await router.swapExactTokensForETH(
    amountIn,
    amountOutMin,
    [USDG, WRAPPED_NATIVE],
    owner.address,
    deadline,
    encodedUpdateData
  );

  await txSwap.wait();
  console.log(`Swap transaction hash: ${txSwap.hash}`);
}

main().catch((error) => {
  console.error(error?.shortMessage ?? error?.message ?? error);
  process.exitCode = 1;
});
```

#### Native to USDG

Use the wrapped native token address as the first path token, then call `swapExactETHForTokens`.

```typescript
const txSwap = await router.swapExactETHForTokens(
  amountOutMin,
  [WRAPPED_NATIVE, USDG],
  owner.address,
  deadline,
  encodedUpdateData,
  { value: amountIn }
);
```

## Example: Complete Robinhood Swap Script

```typescript
import hre from "hardhat";
import { AbiCoder } from "ethers";

const ROBINHOOD_CHAIN_ID = 4663n;
const BROWNFI_PYTH_PROXY = "https://api.brownfi.io/pyth";

const ROUTER_ADDRESS = "0x2A08Da7B6590ce5D217161F234069CfC54DBe554";
const WRAPPED_NATIVE = "0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73";
const USDG = "0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168";

const PRICE_FEED_IDS = {
  eth: "0xff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace",
  usdg: "0xdaa58c6a3ce7d4b9c46c32a6e646012c17c4a2b24c08dd8c5e476118b855a7da",
};

const PATH_TOKEN_KEYS = ["usdg", "weth"] as const;
const AMOUNT_IN = "10";
const SLIPPAGE_BPS = 50n;
const DEADLINE_MINUTES = 20;

type BrownFiPythProxyResponse = {
  binary?: {
    data?: string[];
    encoding?: string;
  };
};

function resolveTokenAddress(tokenKey: string): string {
  const normalized = tokenKey.toLowerCase();
  if (normalized === "usdg") return USDG;
  if (normalized === "weth" || normalized === "eth" || normalized === "native") {
    return WRAPPED_NATIVE;
  }
  throw new Error(`Unsupported Robinhood token key: ${tokenKey}`);
}

function mapTokenKeyToPriceFeedKey(tokenKey: string): keyof typeof PRICE_FEED_IDS {
  const normalized = tokenKey.toLowerCase();
  if (normalized === "usdg") return "usdg";
  if (normalized === "weth" || normalized === "eth" || normalized === "native") {
    return "eth";
  }
  throw new Error(`Unsupported Robinhood price feed key: ${tokenKey}`);
}

async function getBrownFiPythUpdateData(priceFeedIds: string[]): Promise<string[]> {
  const url = new URL(`${BROWNFI_PYTH_PROXY}/v2/updates/price/latest`);

  for (const feedId of priceFeedIds) {
    url.searchParams.append("ids[]", feedId);
  }

  const response = await fetch(url);
  if (!response.ok) {
    const body = await response.text();
    throw new Error(`BrownFi Pyth proxy failed: ${response.status} ${body}`);
  }

  const payload = (await response.json()) as BrownFiPythProxyResponse;
  const data = payload.binary?.data;

  if (!data || data.length === 0) {
    throw new Error("BrownFi Pyth proxy returned no binary update data");
  }

  return data.map((item) => (item.startsWith("0x") ? item : `0x${item}`));
}

async function executeSwap() {
  const [owner] = await hre.ethers.getSigners();
  const network = await hre.ethers.provider.getNetwork();

  if (network.chainId !== ROBINHOOD_CHAIN_ID) {
    throw new Error(`Expected Robinhood chain ID ${ROBINHOOD_CHAIN_ID}, got ${network.chainId}`);
  }

  const router = await hre.ethers.getContractAt("BrownFiV3Router", ROUTER_ADDRESS, owner);

  const pathAddresses = PATH_TOKEN_KEYS.map(resolveTokenAddress);
  const inputTokenAddress = pathAddresses[0];
  const outputTokenAddress = pathAddresses[pathAddresses.length - 1];
  const isNativeIn = inputTokenAddress.toLowerCase() === WRAPPED_NATIVE.toLowerCase();
  const isNativeOut = outputTokenAddress.toLowerCase() === WRAPPED_NATIVE.toLowerCase();

  const erc20Abi = [
    "function decimals() view returns (uint8)",
    "function symbol() view returns (string)",
    "function allowance(address owner, address spender) view returns (uint256)",
    "function approve(address spender, uint256 amount) returns (bool)",
  ];

  const inputToken = new hre.ethers.Contract(inputTokenAddress, erc20Abi, owner);
  const inputDecimals = isNativeIn ? 18 : await inputToken.decimals();
  const amountIn = hre.ethers.parseUnits(AMOUNT_IN, inputDecimals);

  const feedIds = [...new Set(PATH_TOKEN_KEYS.map(mapTokenKeyToPriceFeedKey))]
    .map((key) => PRICE_FEED_IDS[key]);

  const priceFeedUpdateData = await getBrownFiPythUpdateData(feedIds);
  const encodedUpdateData = AbiCoder.defaultAbiCoder().encode(
    ["bytes[]"],
    [priceFeedUpdateData]
  );

  const quotedAmountsOut = await router.quoteAmountsOutWithUpdate.staticCall(
    amountIn,
    pathAddresses,
    encodedUpdateData
  );
  const quotedOut = quotedAmountsOut[quotedAmountsOut.length - 1] as bigint;
  const amountOutMin = (quotedOut * (10_000n - SLIPPAGE_BPS)) / 10_000n;

  if (!isNativeIn) {
    const allowance = await inputToken.allowance(owner.address, ROUTER_ADDRESS);
    if (allowance < amountIn) {
      const txApprove = await inputToken.approve(ROUTER_ADDRESS, amountIn);
      await txApprove.wait();
      console.log(`Approved input token: ${txApprove.hash}`);
    }
  }

  const deadline = Math.floor(Date.now() / 1000) + DEADLINE_MINUTES * 60;
  let txSwap;

  if (isNativeIn) {
    await router.swapExactETHForTokens.staticCall(
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      encodedUpdateData,
      { value: amountIn }
    );
    txSwap = await router.swapExactETHForTokens(
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      encodedUpdateData,
      { value: amountIn }
    );
  } else if (isNativeOut) {
    await router.swapExactTokensForETH.staticCall(
      amountIn,
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      encodedUpdateData
    );
    txSwap = await router.swapExactTokensForETH(
      amountIn,
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      encodedUpdateData
    );
  } else {
    await router.swapExactTokensForTokens.staticCall(
      amountIn,
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      encodedUpdateData
    );
    txSwap = await router.swapExactTokensForTokens(
      amountIn,
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      encodedUpdateData
    );
  }

  await txSwap.wait();
  console.log(`Swap completed: ${txSwap.hash}`);
}

executeSwap().catch((error) => {
  console.error("Robinhood swap failed");
  console.error(error?.shortMessage ?? error?.message ?? error);
  process.exitCode = 1;
});
```

## Example: Run Swap Script

Run the script from the V3 periphery project:

```bash
npx hardhat run scripts/SwapRobinhood.ts --network robinhood
```

The script fetches Pyth updates from the BrownFi proxy, ABI-encodes the returned `binary.data` as `bytes[]`, quotes the swap with `quoteAmountsOutWithUpdate.staticCall`, then sends the corresponding swap transaction.
