# BrownFiV3 Router Swap Guide on HyperEVM Without Pyth API

This guide provides step-by-step instructions for performing swaps on HyperEVM using the `BrownFiV3Router` contract without calling Pyth Hermes, the BrownFi Pyth proxy, or any other Pyth API.

The no-API flow uses prices already available on-chain:

- Quote with the router's view helpers: `getAmountsOut` or `getAmountsIn`.
- Pass empty update data, `"0x"`, into swap calls.
- Do not install `@pythnetwork/pyth-evm-js`.
- Do not fetch Pyth update payloads.

If on-chain oracle prices are stale, quote or swap calls may revert. In that case, prices must be refreshed by an oracle updater/keeper outside this script.

## HyperEVM Deployment

| Item | Value |
| --- | --- |
| Network | HyperEVM |
| Chain ID | `999` |
| RPC URL | `https://rpc.hyperliquid.xyz/evm` |
| Explorer | `https://hyperevmscan.io/` |
| Factory | `0x6A4Bd89709b67eC846F02cF9E95A0dd2Fb515720` |
| BrownFiV3Router | `0x98F6369ecf2A2f7A519773AC40C561701a89828b` |
| BrownFiV3Zap | `0xF211a742898FC57244769163ff6a47025048c4Dc` |
| Wrapped native HYPE | `0x5555555555555555555555555555555555555555` |
| USDT | `0xB8CE59FC3717ada4C02eaDF9682A9e934F625ebb` |
| kHYPE | `0xfD739d4e423301CE9385c1fb8850539D657C296D` |
| USDC | `0xb88339CB7199b77E23DB6E890353E22632Ba630f` |
| UBTC | `0x9FDBdA0A5e284c32744D2f17Ee5c74B284993463` |
| UETH | `0xBe6727B535545C67d5cAa73dEa54865B92CF7907` |

## Prerequisites

Before starting, ensure you have:

- Node.js and npm installed
- Hardhat development environment set up
- `PRIVATE_KEY` configured for HyperEVM transactions
- `HYPEREVM_RPC_URL` configured, or use the default public RPC
- Sufficient HYPE for gas fees
- Sufficient input token balance
- A pool already created for the swap path

## Required Dependencies

Install only the standard packages:

```bash
npm install ethers hardhat
```

No Pyth SDK package is required for this guide.

## Step-by-Step Swap Process

### Step 1: Initialize the Environment

```typescript
import hre from "hardhat";

const HYPEREVM_CHAIN_ID = 999n;
const UPDATE_DATA = "0x";

const ROUTER_ADDRESS = "0x98F6369ecf2A2f7A519773AC40C561701a89828b";
const WRAPPED_HYPE = "0x5555555555555555555555555555555555555555";
const USDT = "0xB8CE59FC3717ada4C02eaDF9682A9e934F625ebb";
const KHYPE = "0xfD739d4e423301CE9385c1fb8850539D657C296D";
const USDC = "0xb88339CB7199b77E23DB6E890353E22632Ba630f";
const UBTC = "0x9FDBdA0A5e284c32744D2f17Ee5c74B284993463";
const UETH = "0xBe6727B535545C67d5cAa73dEa54865B92CF7907";

async function main() {
  const [owner] = await hre.ethers.getSigners();
  console.log(`Account: ${owner.address}`);
```

### Step 2: Validate HyperEVM Network

```typescript
  const network = await hre.ethers.provider.getNetwork();
  const chainId = network.chainId;
  console.log(`Current chain ID: ${chainId}`);

  if (chainId !== HYPEREVM_CHAIN_ID) {
    throw new Error(`Expected HyperEVM chain ID ${HYPEREVM_CHAIN_ID}, got ${chainId}`);
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

  // Example: USDT -> HYPE. Replace with another supported path if needed.
  const inputTokenAddress = USDT;
  const outputTokenAddress = WRAPPED_HYPE;
  const pathAddresses = [inputTokenAddress, outputTokenAddress];

  const inputToken = new hre.ethers.Contract(inputTokenAddress, erc20Abi, owner);
  const [inputDecimals, inputSymbol] = await Promise.all([
    inputToken.decimals(),
    inputToken.symbol(),
  ]);
```

### Step 4: Quote Without Pyth API

Use `getAmountsOut` directly. This is a view call that reads current on-chain oracle state through the BrownFi V3 factory.

```typescript
  const amountIn = hre.ethers.parseUnits("0.1", inputDecimals);

  const quotedAmountsOut = await router.getAmountsOut(
    amountIn,
    pathAddresses
  );

  const quotedOut = quotedAmountsOut[quotedAmountsOut.length - 1] as bigint;
  const slippageBps = 50n; // 0.50%
  const amountOutMin = (quotedOut * (10_000n - slippageBps)) / 10_000n;
```

Do not call `quoteAmountsOutWithUpdate` unless you intend to update Pyth prices. For a no-API swap, use `getAmountsOut` and send `"0x"` as `updateData`.

### Step 5: Approve Token Spending

```typescript
  const allowance = await inputToken.allowance(owner.address, ROUTER_ADDRESS);
  if (allowance < amountIn) {
    const txApprove = await inputToken.approve(ROUTER_ADDRESS, amountIn);
    await txApprove.wait();
    console.log(`Approved ${inputSymbol}: ${txApprove.hash}`);
  }
```

### Step 6: Execute the Swap With Empty Update Data

#### Token to Native HYPE

Use wrapped HYPE as the final path token, then call `swapExactTokensForETH`.

```typescript
  const deadline = Math.floor(Date.now() / 1000) + 60 * 20;

  await router.swapExactTokensForETH.staticCall(
    amountIn,
    amountOutMin,
    [USDT, WRAPPED_HYPE],
    owner.address,
    deadline,
    UPDATE_DATA
  );

  const txSwap = await router.swapExactTokensForETH(
    amountIn,
    amountOutMin,
    [USDT, WRAPPED_HYPE],
    owner.address,
    deadline,
    UPDATE_DATA
  );

  await txSwap.wait();
  console.log(`Swap transaction hash: ${txSwap.hash}`);
}

main().catch((error) => {
  console.error(error?.shortMessage ?? error?.message ?? error);
  process.exitCode = 1;
});
```

#### Native HYPE to Token

Use wrapped HYPE as the first path token, then call `swapExactETHForTokens`.

```typescript
const txSwap = await router.swapExactETHForTokens(
  amountOutMin,
  [WRAPPED_HYPE, USDT],
  owner.address,
  deadline,
  UPDATE_DATA,
  { value: amountIn }
);
```

#### Token to Token

For token-to-token swaps, keep `UPDATE_DATA = "0x"` as the final argument.

```typescript
const txSwap = await router.swapExactTokensForTokens(
  amountIn,
  amountOutMin,
  [USDT, KHYPE],
  owner.address,
  deadline,
  UPDATE_DATA
);
```

## Example: Complete HyperEVM No-API Swap Script

```typescript
import hre from "hardhat";

const HYPEREVM_CHAIN_ID = 999n;
const UPDATE_DATA = "0x";

const ROUTER_ADDRESS = "0x98F6369ecf2A2f7A519773AC40C561701a89828b";
const WRAPPED_HYPE = "0x5555555555555555555555555555555555555555";
const USDT = "0xB8CE59FC3717ada4C02eaDF9682A9e934F625ebb";
const KHYPE = "0xfD739d4e423301CE9385c1fb8850539D657C296D";
const USDC = "0xb88339CB7199b77E23DB6E890353E22632Ba630f";
const UBTC = "0x9FDBdA0A5e284c32744D2f17Ee5c74B284993463";
const UETH = "0xBe6727B535545C67d5cAa73dEa54865B92CF7907";

const PATH_TOKEN_KEYS = ["usdt", "hype"] as const;
const AMOUNT_IN = "0.1";
const SLIPPAGE_BPS = 50n;
const DEADLINE_MINUTES = 20;

function resolveTokenAddress(tokenKey: string): string {
  const normalized = tokenKey.toLowerCase();
  if (normalized === "hype" || normalized === "native") return WRAPPED_HYPE;
  if (normalized === "usdt") return USDT;
  if (normalized === "khype") return KHYPE;
  if (normalized === "usdc") return USDC;
  if (normalized === "ubtc") return UBTC;
  if (normalized === "ueth") return UETH;
  throw new Error(`Unsupported HyperEVM token key: ${tokenKey}`);
}

async function executeSwap() {
  const [owner] = await hre.ethers.getSigners();
  const network = await hre.ethers.provider.getNetwork();

  if (network.chainId !== HYPEREVM_CHAIN_ID) {
    throw new Error(`Expected HyperEVM chain ID ${HYPEREVM_CHAIN_ID}, got ${network.chainId}`);
  }

  const router = await hre.ethers.getContractAt("BrownFiV3Router", ROUTER_ADDRESS, owner);

  const pathAddresses = PATH_TOKEN_KEYS.map(resolveTokenAddress);
  const inputTokenAddress = pathAddresses[0];
  const outputTokenAddress = pathAddresses[pathAddresses.length - 1];
  const isNativeIn = inputTokenAddress.toLowerCase() === WRAPPED_HYPE.toLowerCase();
  const isNativeOut = outputTokenAddress.toLowerCase() === WRAPPED_HYPE.toLowerCase();

  const erc20Abi = [
    "function decimals() view returns (uint8)",
    "function symbol() view returns (string)",
    "function allowance(address owner, address spender) view returns (uint256)",
    "function approve(address spender, uint256 amount) returns (bool)",
  ];

  const inputDecimals = isNativeIn
    ? 18
    : await new hre.ethers.Contract(inputTokenAddress, erc20Abi, owner).decimals();

  const amountIn = hre.ethers.parseUnits(AMOUNT_IN, inputDecimals);

  const quotedAmountsOut = await router.getAmountsOut(
    amountIn,
    pathAddresses
  );
  const quotedOut = quotedAmountsOut[quotedAmountsOut.length - 1] as bigint;
  const amountOutMin = (quotedOut * (10_000n - SLIPPAGE_BPS)) / 10_000n;

  if (!isNativeIn) {
    const inputToken = new hre.ethers.Contract(inputTokenAddress, erc20Abi, owner);
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
      UPDATE_DATA,
      { value: amountIn }
    );
    txSwap = await router.swapExactETHForTokens(
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      UPDATE_DATA,
      { value: amountIn }
    );
  } else if (isNativeOut) {
    await router.swapExactTokensForETH.staticCall(
      amountIn,
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      UPDATE_DATA
    );
    txSwap = await router.swapExactTokensForETH(
      amountIn,
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      UPDATE_DATA
    );
  } else {
    await router.swapExactTokensForTokens.staticCall(
      amountIn,
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      UPDATE_DATA
    );
    txSwap = await router.swapExactTokensForTokens(
      amountIn,
      amountOutMin,
      pathAddresses,
      owner.address,
      deadline,
      UPDATE_DATA
    );
  }

  await txSwap.wait();
  console.log(`Swap completed: ${txSwap.hash}`);
}

executeSwap().catch((error) => {
  console.error("HyperEVM no-API swap failed");
  console.error(error?.shortMessage ?? error?.message ?? error);
  process.exitCode = 1;
});
```

## Example: Run Swap Script

Run the script from the V3 periphery project:

```bash
npx hardhat run scripts/SwapHyperEVMNoPythApi.ts --network hyperevm
```

This guide intentionally avoids Pyth API calls. It assumes the BrownFi V3 on-chain oracle prices on HyperEVM are already fresh enough for the factory's configured price-age checks.
